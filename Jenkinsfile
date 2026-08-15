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