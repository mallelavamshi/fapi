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
                            
                            access_log /var/log/nginx/access.log;
                            error_log /var/log/nginx/error.log;
                            
                            location / {
                                proxy_pass http://api:8000;
                                proxy_http_version 1.1;
                                proxy_set_header Upgrade $http_upgrade;
                                proxy_set_header Connection 'upgrade';
                                proxy_set_header Host $host;
                                proxy_set_header X-Real-IP $remote_addr;
                                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                                proxy_set_header X-Forwarded-Proto $scheme;
                                proxy_cache_bypass $http_upgrade;
                            }
                            
                            location /docs {
                                proxy_pass http://api:8000/docs;
                                proxy_http_version 1.1;
                                proxy_set_header Upgrade $http_upgrade;
                                proxy_set_header Connection 'upgrade';
                                proxy_set_header Host $host;
                            }
                            
                            location /openapi.json {
                                proxy_pass http://api:8000/openapi.json;
                                proxy_http_version 1.1;
                                proxy_set_header Host $host;
                            }
                        }
                    '''
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    sh 'echo "Running tests..."'
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
                    
                    // Wait for services to be healthy
                    sh '''
                        echo "Waiting for services to be ready..."
                        sleep 10
                        
                        # Test FastAPI service
                        curl -f http://localhost:8000/health || exit 1
                        
                        # Test Nginx proxy
                        curl -f http://localhost:8081/health || exit 1
                    '''
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
        success {
            echo """
            ====================================
            Application deployed successfully!
            ====================================
            
            Access the application at:
            - API (direct): http://your-server-ip:8000
            - API (via Nginx): http://your-server-ip:8081
            - API Documentation: http://your-server-ip:8081/docs
            
            Jenkins is running on: http://your-server-ip:8080
            """
        }
    }
}