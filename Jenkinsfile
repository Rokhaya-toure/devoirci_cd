pipeline {
    agent any

    environment {
        DOCKER_HUB_REPO = 'rokhayatoure560/devoirCI'
        DOCKER_HUB_CREDENTIALS = 'docker-hub-new'
        RENDER_DEPLOY_HOOK = 'render-webhook'
        RENDER_APP_URL = 'render-app-url'
        MAVEN_OPTS = '-Dmaven.repo.local=/tmp/.m2/repository'
        DOCKER_HOST = 'tcp://localhost:2375'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Récupération du code source...'
                bat 'dir'
                bat 'echo Répertoire actuel: %CD%'
            }
        }

        stage('Environment Check') {
            steps {
                echo 'Vérification de l\'environnement...'
                bat 'java -version'
                bat 'if exist mvnw.cmd (echo Maven wrapper trouvé) else (echo Maven wrapper manquant)'
            }
        }

        stage('Build & Test') {
            steps {
                echo 'Construction et tests du projet...'
                bat 'if exist mvnw.cmd (echo Maven wrapper trouvé) else (echo Maven wrapper non trouvé && exit 1)'
                bat 'mvnw.cmd clean compile test -Dmaven.test.failure.ignore=true'
            }
        }

        stage('Package') {
            steps {
                echo 'Création du package JAR...'
                bat 'mvnw.cmd clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
            }
        }

        stage('Docker Diagnostics') {
            steps {
                echo 'Diagnostic Docker...'
                script {
                    try {
                        bat 'docker --version'
                        env.DOCKER_METHOD = 'pipe'
                        echo "Docker via pipe Windows: OK"
                    } catch (Exception e1) {
                        echo "Docker via pipe échoué: ${e1.getMessage()}"
                        try {
                            bat 'set DOCKER_HOST=tcp://localhost:2375 && docker --version'
                            env.DOCKER_METHOD = 'tcp'
                            echo "Docker via TCP: OK"
                        } catch (Exception e2) {
                            echo "Docker via TCP échoué: ${e2.getMessage()}"
                            env.DOCKER_METHOD = 'none'
                            echo "Docker non disponible"
                        }
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            when {
                expression { env.DOCKER_METHOD != 'none' }
            }
            steps {
                echo 'Construction et push de l\'image Docker...'
                script {
                    def imageName = "${DOCKER_HUB_REPO}:${BUILD_NUMBER}"
                    def latestImageName = "${DOCKER_HUB_REPO}:latest"

                    // Vérification du Dockerfile (insensible à la casse)
                    bat '''
                    if exist Dockerfile (
                        echo Dockerfile trouvé
                    ) else if exist dockerfile (
                        echo dockerfile trouvé, renommage en Dockerfile
                        ren dockerfile Dockerfile
                    ) else (
                        echo Aucun Dockerfile trouvé && exit 1
                    )
                    '''

                    def dockerCmd = ""
                    if (env.DOCKER_METHOD == 'tcp') {
                        dockerCmd = "set DOCKER_HOST=tcp://localhost:2375 && "
                    }

                    // Construction de l'image
                    bat "${dockerCmd}docker build -t ${imageName} ."
                    bat "${dockerCmd}docker tag ${imageName} ${latestImageName}"

                    // Push vers Docker Hub
                    withCredentials([usernamePassword(credentialsId: DOCKER_HUB_CREDENTIALS,
                                                    usernameVariable: 'DOCKER_USER',
                                                    passwordVariable: 'DOCKER_PASS')]) {
                        bat "${dockerCmd}docker login -u %DOCKER_USER% -p %DOCKER_PASS%"
                        bat "${dockerCmd}docker push ${imageName}"
                        bat "${dockerCmd}docker push ${latestImageName}"
                    }

                    // Nettoyage
                    bat "${dockerCmd}docker rmi ${imageName} || echo Impossible de supprimer ${imageName}"
                    bat "${dockerCmd}docker rmi ${latestImageName} || echo Impossible de supprimer ${latestImageName}"
                }
            }
        }

        stage('Deploy to Render') {
            steps {
                echo 'Déploiement sur Render...'
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
            echo 'Pipeline exécuté avec succès!'
        }
        failure {
            echo 'Pipeline échoué!'
            script {
                if (env.DOCKER_METHOD == 'none') {
                    echo '''
                    SOLUTION DOCKER:
                    1. Ouvrez Docker Desktop
                    2. Settings > General > Activez "Expose daemon on tcp://localhost:2375 without TLS"
                    3. Redémarrez Docker Desktop
                    4. OU modifiez le service Jenkins pour utiliser votre compte utilisateur
                    '''
                }
            }
        }
    }
}