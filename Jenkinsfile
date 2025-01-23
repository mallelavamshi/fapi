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
                    
                    // Ensure nginx configuration directory exists
                    sh 'mkdir -p nginx/conf.d'
                    
                    // Write Nginx configuration
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
        
        stage('Build & Deploy') {
            steps {
                script {
                    // Stop existing containers
                    sh 'docker-compose down || true'
                    
                    // Rebuild and start new containers
                    withCredentials([string(credentialsId: 'dify-api-key', variable: 'DIFY_API_KEY')]) {
                        sh 'docker-compose up -d --build'
                    }
                    
                    // Wait a bit to ensure Nginx picks up the changes
                    sleep(5)
                    
                    // Restart Nginx to ensure it's using the new config
                    sh 'docker exec -it fapi-nginx-1 nginx -s reload || true'
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    // Check if default.conf is present in the Nginx container
                    sh 'docker exec -it fapi-nginx-1 ls -l /etc/nginx/conf.d/'
                    
                    // Test if Nginx is correctly forwarding requests
                    sh 'curl -I http://localhost:8081 || true'
                }
            }
        }
    }
    
    post {
        always {
            // Clean up unused Docker resources
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
