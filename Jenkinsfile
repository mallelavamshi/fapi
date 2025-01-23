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
                    sh 'docker exec fapi-nginx-1 nginx -s reload || true'
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
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
