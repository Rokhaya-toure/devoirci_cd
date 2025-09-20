pipeline {
    agent any

    tools {
        maven 'Maven-3.9.0'
        jdk 'JDK-11'
    }

    environment {
        DOCKER_IMAGE = "demo-springboot"
        DOCKERHUB_REPO = "rokhayatoure/watsaap_back"
       RENDER_DEPLOY_HOOK = "https://api.render.com/deploy/srv-d37c4cnfte5s73b5gh80?key=VTeRXjQXajM"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Maven') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                bat """
                docker version
                docker build -t %DOCKER_IMAGE% .
                docker tag %DOCKER_IMAGE% %DOCKERHUB_REPO%:latest
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat """
                    echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                    docker push %DOCKERHUB_REPO%:latest
                    docker logout
                    """
                }
            }
        }

        stage('Deploy to Render') {
            steps {
                echo 'Déclenchement du déploiement Render...'
                bat "curl -X POST %RENDER_DEPLOY_HOOK%"
            }
        }
    }

    post {
        success {
            echo 'Pipeline terminé avec succès 🎉'
        }
        failure {
            echo 'Échec du pipeline ❌'
        }
    }
}