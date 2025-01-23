pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'image-analysis-api'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DIFY_API_KEY = credentials('dify-api-key')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup Environment') {
            steps {
                script {
                    // Create .env file securely
                    withCredentials([string(credentialsId: 'dify-api-key', variable: 'DIFY_API_KEY')]) {
                        writeFile file: '.env', text: "DIFY_API_KEY=${DIFY_API_KEY}"
                    }
                    
                    // Create nginx configuration directory
                    sh 'mkdir -p nginx/conf.d'
                    
                    // Create default nginx configuration
                    writeFile file: 'nginx/conf.d/default.conf', text: '''
                        server {
                            listen 80;
                            server_name localhost;
                            
                            location / {
                                proxy_pass http://api:8000;
                                proxy_set_header Host $host;
                                proxy_set_header X-Real-IP $remote_addr;
                                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                                proxy_set_header X-Forwarded-Proto $scheme;
                            }
                        }
                    '''
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    sh 'echo "Running tests..."'
                    // Add your test commands here
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    // Stop existing containers
                    sh 'docker-compose down || true'
                    
                    // Start new containers
                    withCredentials([string(credentialsId: 'dify-api-key', variable: 'DIFY_API_KEY')]) {
                        sh 'docker-compose up -d'
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Clean up
            sh 'docker system prune -f'
            cleanWs()
        }
    }
}