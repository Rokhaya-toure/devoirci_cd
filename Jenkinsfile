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
                sh 'git config --global http.version HTTP/1.1'
                sh 'git config --global http.postBuffer 524288000'
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                echo '🔨 Construction et tests du projet...'
                sh '''
                    chmod +x ./mvnw
                    ./mvnw clean compile test -Dmaven.test.failure.ignore=true
                '''
            }
        }

        stage('Package') {
            steps {
                echo '📦 Création du package JAR...'
                sh './mvnw clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo '🐳 Construction et push de l\'image Docker...'
                script {
                    def imageName = "${DOCKER_HUB_REPO}:${BUILD_NUMBER}"
                    def latestImageName = "${DOCKER_HUB_REPO}:latest"

                    dockerImage = docker.build(imageName)

                    docker.withRegistry('https://registry.hub.docker.com', DOCKER_HUB_CREDENTIALS) {
                        dockerImage.push("${BUILD_NUMBER}")
                        dockerImage.push("latest")
                    }

                    sh "docker rmi ${imageName} ${latestImageName} || true"
                }
            }
        }

        stage('Deploy to Render') {
            steps {
                echo '🚀 Déploiement sur Render...'
                script {
                    withCredentials([string(credentialsId: RENDER_DEPLOY_HOOK, variable: 'RENDER_WEBHOOK')]) {
                        sh '''
                            curl -X POST "$RENDER_WEBHOOK" \
                                -H "Content-Type: application/json" \
                                -d '{"branch": "main"}'
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            script { deleteDir() }
        }
        success {
            echo '🎉 Pipeline exécuté avec succès!'
        }
        failure {
            echo '❌ Pipeline échoué!'
        }
    }
}
