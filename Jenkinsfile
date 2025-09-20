pipeline {
    agent any

    environment {
        DOCKER_HUB_REPO = 'rokhayatoure560/devoirCI'
        DOCKER_HUB_CREDENTIALS = 'docker-hub-new'
        RENDER_DEPLOY_HOOK = 'render-webhook'
        RENDER_APP_URL = 'render-app-url'
        MAVEN_OPTS = '-Dmaven.repo.local=/tmp/.m2/repository'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '🔄 Récupération du code source...'
                // Vérification que nous sommes bien dans le bon répertoire
                bat 'dir'
                bat 'echo Répertoire actuel: %CD%'
            }
        }

        stage('Build & Test') {
            steps {
                echo '🔨 Construction et tests du projet...'
                // Vérification de l'existence du wrapper Maven
                bat 'if exist mvnw.cmd (echo Maven wrapper trouvé) else (echo Maven wrapper non trouvé && exit 1)'
                bat 'mvnw.cmd clean compile test -Dmaven.test.failure.ignore=true'
            }
        }

        stage('Package') {
            steps {
                echo '📦 Création du package JAR...'
                bat 'mvnw.cmd clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo '🐳 Construction et push de l\'image Docker...'
                script {
                    def imageName = "${DOCKER_HUB_REPO}:${BUILD_NUMBER}"
                    def latestImageName = "${DOCKER_HUB_REPO}:latest"

                    // Vérification de l'existence du Dockerfile
                    bat 'if exist Dockerfile (echo Dockerfile trouvé) else (echo Dockerfile non trouvé && exit 1)'

                    dockerImage = docker.build(imageName)

                    docker.withRegistry('https://registry.hub.docker.com', DOCKER_HUB_CREDENTIALS) {
                        dockerImage.push("${BUILD_NUMBER}")
                        dockerImage.push("latest")
                    }

                    // Suppression des images locales (avec gestion d'erreur améliorée)
                    bat """
                    docker images
                    docker rmi ${imageName} || echo "Impossible de supprimer ${imageName}"
                    docker rmi ${latestImageName} || echo "Impossible de supprimer ${latestImageName}"
                    """
                }
            }
        }

        stage('Deploy to Render') {
            steps {
                echo '🚀 Déploiement sur Render...'
                script {
                    withCredentials([string(credentialsId: RENDER_DEPLOY_HOOK, variable: 'RENDER_WEBHOOK')]) {
                        bat """
                        curl -X POST "%RENDER_WEBHOOK%" ^
                            -H "Content-Type: application/json" ^
                            -d "{\\"branch\\": \\"main\\"}"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo 'Nettoyage du workspace...'
                deleteDir()
            }
        }
        success {
            echo '🎉 Pipeline exécuté avec succès!'
        }
        failure {
            echo '❌ Pipeline échoué!'
        }
    }
}