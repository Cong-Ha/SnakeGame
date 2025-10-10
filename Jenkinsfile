pipeline {
    agent any
    
    environment {
        NODE_VERSION = '18'
        DOCKER_IMAGE_NAME = 'uzomaki/snake-game'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS = credentials('docker-id')
        DOCKERHUB_URL = 'https://registry.hub.docker.com'
    }
    
    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }

        stage('Static Security Scan') {
            steps {
                sh 'echo running static security scan'
            }
        }

        stage('BUILD-AND-TAG') {
            agent {label 'App-Server'}
            steps {
                script {
                    echo "Building Docker image ${DOCKER_IMAGE_NAME}..."
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.tag("latest")
                }
            }
        }
        
        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    echo "Pushing Docker image ${DOCKER_IMAGE_NAME}:latest to Docker Hub..."
                    docker.withRegistry(env.DOCKERHUB_URL, env.DOCKERHUB_CREDENTIALS) {
                        app.push("latest")
                        app.push("${env.BUILD_NUMBER}")
                    }
                }
            }
        }
        
        stage('Deployment') {
            agent {label 'App-Server'}
            steps{
                echo "Starting deployment using docker-compose..."
                script{
                    dir("$WORKSPACE") {
                        sh '''
                            docker-compose down
                            docker-compose up -d
                            docker ps
                        '''    
                    }
                }
                echo 'Deployment completed successfully'
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline succeeded! Application is running on port 172.232.3.70:5000'
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            echo 'Pipeline execution completed'
        }
    }
}

