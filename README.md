# Jenkins + Docker + Nginx + GitHub

## 1. Descripción del proyecto

Este proyecto implementa un flujo básico de **Integración y Despliegue Continuo (CI/CD)** utilizando:

* GitHub como repositorio del código fuente.
* Jenkins como servidor de automatización.
* Docker como plataforma de contenedores.
* Nginx como servidor web.
* HTML como tecnología utilizada para la página web.

El objetivo es que Jenkins obtenga el proyecto desde GitHub, verifique los archivos necesarios, construya automáticamente una imagen Docker con Nginx y despliegue la aplicación como un contenedor.

---

# 2. Arquitectura del proyecto

El flujo implementado es:

```text
                    GitHub
                       │
                       │
                       ▼
                  ┌─────────┐
                  │ Jenkins │
                  └────┬────┘
                       │
                  Jenkinsfile
                       │
           ┌───────────┴───────────┐
           │                       │
           ▼                       ▼
      Verificación            Construcción
                                   │
                              docker build
                                   │
                                   ▼
                            Imagen mi-web
                                   │
                              docker run
                                   │
                                   ▼
                             Contenedor
                               Nginx
                                   │
                                   ▼
                          http://localhost:8081
```

Jenkins funciona dentro de un contenedor Docker y tiene acceso al Docker Engine del sistema anfitrión mediante:

```text
/var/run/docker.sock
```

---

# 3. Estructura del proyecto

La estructura utilizada es:

```text
JENKINS PARCIAL/
├── compose.yml
├── Dockerfile
├── dockerfile.jenkins
├── index.html
├── Jenkinsfile
└── README.md
```

## Descripción de los archivos

| Archivo              | Función                                            |
| -------------------- | -------------------------------------------------- |
| `index.html`         | Página web                                         |
| `Dockerfile`         | Construcción de la imagen de Nginx                 |
| `dockerfile.jenkins` | Construcción de la imagen personalizada de Jenkins |
| `compose.yml`        | Configuración y ejecución de Jenkins               |
| `Jenkinsfile`        | Pipeline CI/CD                                     |
| `README.md`          | Documentación del proyecto                         |

---

# 4. Página web

La aplicación consiste en una página HTML básica almacenada en `index.html`.

El archivo se incorpora posteriormente a la imagen de Nginx mediante el `Dockerfile`.

---

# 5. Dockerfile de la aplicación

El archivo `Dockerfile` utiliza Nginx como imagen base:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
```

## Funcionamiento

```text
nginx:alpine
     │
     ▼
COPY index.html
     │
     ▼
/usr/share/nginx/html/
     │
     ▼
Nginx sirve la página
```

La imagen se construye mediante:

```bash
docker build -t mi-web .
```

---

# 6. Ejecución manual de la aplicación

Antes de integrar Jenkins se verificó manualmente que Docker y Nginx funcionaran correctamente.

Se ejecutó:

```bash
docker build -t mi-web .
```

Posteriormente se creó el contenedor:

```bash
docker run -d --name mi-web -p 8081:80 mi-web
```

El puerto utilizado fue `8081` porque Jenkins utiliza el puerto `8080`.

La aplicación queda disponible en:

```text
http://localhost:8081
```

Para comprobar el contenedor:

```bash
docker ps
```

---

# 7. Configuración inicial de Jenkins

Jenkins se ejecuta mediante Docker Compose.

Inicialmente se utilizó una configuración basada en:

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    ports:
      - "8080:8080"
      - "5000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
```

Jenkins queda disponible mediante:

```text
http://localhost:8080
```

---

# 8. Problema encontrado: Jenkins no podía utilizar Docker

Al intentar ejecutar:

```bash
docker exec -it jenkinsparcial-jenkins-1 docker --version
```

se obtuvo inicialmente:

```text
docker: executable file not found in $PATH
```

Esto ocurrió porque la imagen original de Jenkins no tenía instalado el cliente Docker.

Para solucionarlo se creó:

```text
dockerfile.jenkins
```

---

# 9. Dockerfile personalizado de Jenkins

El archivo `dockerfile.jenkins` quedó configurado de la siguiente manera:

```dockerfile
FROM jenkins/jenkins:lts

USER root

RUN apt-get update \
    && apt-get install -y docker.io \
    && rm -rf /var/lib/apt/lists/*

USER jenkins
```

Este Dockerfile permite instalar el cliente Docker dentro del contenedor de Jenkins.

Después de reconstruir la imagen:

```bash
docker compose build --no-cache
```

se comprobó:

```bash
docker exec -it jenkinsparcial-jenkins-1 docker --version
```

Resultado obtenido:

```text
Docker version 26.1.5+dfsg1, build a72d7cd
```

---

# 10. Acceso de Jenkins al Docker del sistema

Aunque Docker CLI ya estaba instalado, Jenkins inicialmente no tenía permisos para comunicarse con el Docker Engine.

El error obtenido fue:

```text
permission denied while trying to connect to the Docker daemon socket
```

Se verificó el socket:

```bash
ls -l /var/run/docker.sock
```

Resultado:

```text
srw-rw---- 1 root docker ... /var/run/docker.sock
```

También se verificó el grupo Docker:

```bash
getent group docker
```

Resultado:

```text
docker:x:998:pedfer
```

Por lo tanto, el GID del grupo Docker del sistema era:

```text
998
```

---

# 11. Configuración de Docker Compose para Jenkins

Se agregó el socket Docker al contenedor:

```yaml
- /var/run/docker.sock:/var/run/docker.sock
```

También se agregó el GID del grupo Docker:

```yaml
group_add:
  - "998"
```

La configuración final de `compose.yml` quedó:

```yaml
services:
  jenkins:
    build:
      context: .
      dockerfile: dockerfile.jenkins

    ports:
      - "8080:8080"
      - "5000:50000"

    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock

    group_add:
      - "998"

volumes:
  jenkins_home:
```

Esto permite que Jenkins pueda utilizar el Docker Engine del sistema anfitrión.

---

# 12. Reconstrucción de Jenkins

Después de modificar la configuración se detuvieron los contenedores:

```bash
docker compose down
```

Se reconstruyó la imagen:

```bash
docker compose build --no-cache
```

Y posteriormente se inició Jenkins:

```bash
docker compose up -d
```

Se verificó el acceso a Docker ejecutando:

```bash
docker exec -it jenkinsparcial-jenkins-1 docker ps
```

El resultado mostró correctamente los contenedores:

```text
jenkinsparcial-jenkins-1
mi-web
```

Esto confirmó que Jenkins podía ejecutar comandos Docker sobre el Docker Engine del sistema.

---

# 13. Repositorio GitHub

El código fuente fue almacenado en GitHub:

```text
https://github.com/ferbo11/jenkins-docker-boton.git
```

La rama utilizada es:

```text
main
```

El repositorio contiene los archivos necesarios para construir y ejecutar el proyecto.

---

# 14. Configuración del Pipeline en Jenkins

En Jenkins se creó un proyecto de tipo:

```text
Pipeline
```

El nombre utilizado fue:

```text
Parcial-Jenkins
```

En la configuración del Pipeline se seleccionó:

```text
Definition:
Pipeline script from SCM
```

Como sistema de control de versiones:

```text
Git
```

Repositorio:

```text
https://github.com/ferbo11/jenkins-docker-boton.git
```

Rama:

```text
*/main
```

Ruta del script:

```text
Jenkinsfile
```

---

# 15. Problema con la clonación

Inicialmente el `Jenkinsfile` contenía un stage llamado `Clonar`:

```groovy
stage('Clonar') {
    steps {
        git 'https://github.com/ferbo11/jenkins-docker-boton.git'
    }
}
```

Esto provocaba una segunda clonación del repositorio.

Jenkins ya realizaba automáticamente el checkout porque se había configurado:

```text
Pipeline script from SCM
```

Por lo tanto, el repositorio se estaba clonando dos veces.

Además, la segunda operación llegó a buscar la rama `master`, mientras que el repositorio utilizaba `main`.

El error obtenido fue:

```text
ERROR: Couldn't find any revision to build.
```

---

# 16. Pipeline final

Para que el proceso del parcial mostrara claramente las fases solicitadas, se establecieron las siguientes etapas:

1. Clonación
2. Verificación
3. Construcción
4. Despliegue
5. Confirmación

Para evitar el checkout automático de Jenkins se utiliza:

```groovy
options {
    skipDefaultCheckout(true)
}
```

El `Jenkinsfile` final es:

```groovy
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    stages {

        stage('Clonación') {
            steps {
                checkout scm
            }
        }

        stage('Verificación') {
            steps {
                sh '''
                    echo "=== VERIFICACIÓN DEL PROYECTO ==="
                    pwd
                    ls -la
                    test -f index.html
                    test -f Dockerfile
                    test -f Jenkinsfile
                    echo "Archivos requeridos encontrados correctamente."
                '''
            }
        }

        stage('Construcción') {
            steps {
                sh '''
                    echo "=== CONSTRUCCIÓN DE LA IMAGEN ==="
                    docker build -t mi-web .
                '''
            }
        }

        stage('Despliegue') {
            steps {
                sh '''
                    echo "=== DESPLIEGUE DEL CONTENEDOR ==="
                    docker stop mi-web || true
                    docker rm mi-web || true
                    docker run -d --name mi-web -p 8081:80 mi-web
                '''
            }
        }

        stage('Confirmación') {
            steps {
                sh '''
                    echo "=== CONFIRMACIÓN DEL DESPLIEGUE ==="
                    docker ps --filter "name=mi-web"
                    echo "Contenedor desplegado correctamente."
                    echo "Página disponible en http://localhost:8081"
                '''
            }
        }
    }
}
```

---

# 17. Descripción de las etapas del Pipeline

## 17.1 Clonación

Jenkins obtiene el código fuente desde GitHub utilizando:

```groovy
checkout scm
```

El código queda almacenado en el workspace del proyecto:

```text
/var/jenkins_home/workspace/Parcial-Jenkins
```

---

## 17.2 Verificación

Se verifica que existan los archivos principales:

```text
index.html
Dockerfile
Jenkinsfile
```

Se utilizan comandos como:

```bash
pwd
ls -la
test -f index.html
test -f Dockerfile
test -f Jenkinsfile
```

Si alguno de los archivos requeridos no existe, la etapa falla y Jenkins detiene el Pipeline.

---

## 17.3 Construcción

Jenkins ejecuta:

```bash
docker build -t mi-web .
```

Este comando utiliza el `Dockerfile` del proyecto para construir una imagen basada en:

```text
nginx:alpine
```

La imagen resultante se llama:

```text
mi-web:latest
```

---

## 17.4 Despliegue

Primero se detiene el contenedor anterior:

```bash
docker stop mi-web || true
```

Luego se elimina:

```bash
docker rm mi-web || true
```

Finalmente se crea un nuevo contenedor:

```bash
docker run -d --name mi-web -p 8081:80 mi-web
```

El contenedor ejecuta Nginx y expone la aplicación en:

```text
http://localhost:8081
```

---

## 17.5 Confirmación

Finalmente Jenkins ejecuta:

```bash
docker ps --filter "name=mi-web"
```

Esto permite comprobar que el contenedor está ejecutándose correctamente.

La aplicación queda disponible mediante:

```text
http://localhost:8081
```

---

# 18. Puertos utilizados

| Servicio               |       Puerto |
| ---------------------- | -----------: |
| Jenkins                |         8080 |
| Jenkins Agent          | 5000 → 50000 |
| Nginx / aplicación web |    8081 → 80 |

La relación de puertos es:

```text
Jenkins:
localhost:8080 → contenedor Jenkins:8080

Nginx:
localhost:8081 → contenedor mi-web:80
```

Se utilizó el puerto `8081` para Nginx porque el puerto `8080` ya estaba ocupado por Jenkins.

---

# 19. Comandos principales utilizados

### Ver contenedores activos

```bash
docker ps
```

### Ver todos los contenedores

```bash
docker ps -a
```

### Ver imágenes

```bash
docker images
```

### Construir imagen de la página

```bash
docker build -t mi-web .
```

### Ejecutar página manualmente

```bash
docker run -d --name mi-web -p 8081:80 mi-web
```

### Detener Jenkins mediante Compose

```bash
docker compose down
```

### Construir la imagen personalizada de Jenkins

```bash
docker compose build --no-cache
```

### Iniciar Jenkins

```bash
docker compose up -d
```

### Comprobar Docker dentro de Jenkins

```bash
docker exec -it jenkinsparcial-jenkins-1 docker --version
```

### Comprobar acceso de Jenkins al Docker Engine

```bash
docker exec -it jenkinsparcial-jenkins-1 docker ps
```

---

# 20. Flujo final del proyecto

El funcionamiento completo queda de la siguiente manera:

```text
             ┌──────────────────────┐
             │       GitHub         │
             │ jenkins-docker-      │
             │ boton.git            │
             └──────────┬───────────┘
                        │
                        ▼
             ┌──────────────────────┐
             │       Jenkins        │
             │   Parcial-Jenkins    │
             └──────────┬───────────┘
                        │
                        ▼
                  ┌───────────┐
                  │ Clonación │
                  └─────┬─────┘
                        │
                        ▼
                 ┌─────────────┐
                 │ Verificación│
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │ Construcción│
                 │ Docker      │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │   Imagen    │
                 │   mi-web    │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │ Despliegue  │
                 │ Docker      │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │  Contenedor │
                 │  Nginx      │
                 └──────┬──────┘
                        │
                        ▼
             http://localhost:8081
```

---

# 21. Resultado esperado

Al ejecutar correctamente el Pipeline, Jenkins debe finalizar mostrando:

```text
Finished: SUCCESS
```

Y el contenedor debe aparecer mediante:

```bash
docker ps
```

como:

```text
mi-web
```

La página web debe poder visualizarse desde:

```text
http://localhost:8081
```

Jenkins debe estar disponible desde:

```text
http://localhost:8080
```

---

# 22. Tecnologías utilizadas

* **GitHub** — Control y almacenamiento del código fuente.
* **Jenkins** — Automatización del Pipeline CI/CD.
* **Docker** — Construcción y ejecución de contenedores.
* **Nginx** — Servidor web.
* **Docker Compose** — Administración del contenedor de Jenkins.
* **HTML** — Página web utilizada como aplicación de prueba.

---

# 23. Conclusión

El proyecto implementa un flujo básico de CI/CD en el que el código fuente de una página web se almacena en GitHub y Jenkins automatiza su proceso de construcción y despliegue.

Jenkins se ejecuta dentro de un contenedor Docker y posee acceso al Docker Engine del sistema anfitrión mediante el socket `/var/run/docker.sock`. Esto permite que el Pipeline construya imágenes Docker y cree contenedores automáticamente.

El proceso completo está dividido en las etapas de **Clonación, Verificación, Construcción, Despliegue y Confirmación**, permitiendo visualizar y comprobar cada fase del proceso de integración y despliegue continuo.

El resultado final es una página web servida mediante Nginx dentro de un contenedor Docker y desplegada automáticamente por Jenkins.
