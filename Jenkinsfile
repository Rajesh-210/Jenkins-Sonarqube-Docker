
pipeline {
    agent any

    environment {
        DOCKER_HOST_IP = '172.31.6.186'
        DOCKER_USER = 'ubuntu'
        APP_NAME = 'my-web-app'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Rajesh-210/Jenkins-Sonarqube-Docker.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonarqube'

                    withSonarQubeEnv('sonar-server') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=jenkins-sonarqube-docker \
                              -Dsonar.projectName=Jenkins-Sonarqube-Docker \
                              -Dsonar.sources=. \
                              -Dsonar.exclusions=vendor/**,assets/**,.git/**
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
             }
           }
        }

        stage('Build and Deploy') {
            steps {
                sshagent(credentials: ['docker-ssh']) {
                    sh '''
                        set -eu

                        mkdir -p ~/.ssh
                        chmod 700 ~/.ssh

                        ssh-keyscan -H "$DOCKER_HOST_IP" >> ~/.ssh/known_hosts

                        tar --exclude=.git -czf - . | \
                          ssh "$DOCKER_USER@$DOCKER_HOST_IP" \
                          "mkdir -p /opt/app-ci &&
                           tar -xzf - -C /opt/app-ci &&
                           cd /opt/app-ci &&
                           docker build -t $APP_NAME:$BUILD_NUMBER . &&
                           (docker rm -f $APP_NAME || true) &&
                           docker run -d \
                             --name $APP_NAME \
                             --restart unless-stopped \
                             -p 80:80 \
                             $APP_NAME:$BUILD_NUMBER"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}

