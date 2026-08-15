pipeline {
    agent any

    stages {

        stage('Construir imagen') {
            steps {
                sh 'docker build -t mi-web .'
            }
        }

        stage('Desplegar') {
            steps {
                sh 'docker stop mi-web || true'
                sh 'docker rm mi-web || true'
                sh 'docker run -d --name mi-web -p 8081:80 mi-web'
            }
        }
    }
}