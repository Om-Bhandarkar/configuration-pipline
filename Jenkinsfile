pipeline {
    agent any

    environment {
        REMOTE_IP = credentials('REMOTE_IP')
        SSH_USER  = credentials('SSH_USERNAME')
        SSH_PASS  = credentials('SSH_PASSWORD')
    }

    stages {

        stage('Detect OS on Remote Machine') {
            steps {
                script {
                    sh """
                    sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no $SSH_USER@$REMOTE_IP '
                        echo "=== OS Information ==="
                        uname -a
                    '
                    """
                }
            }
        }

        stage('Install Docker & Docker Compose if Missing') {
            steps {
                script {
                    sh """
                    sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no $SSH_USER@$REMOTE_IP '
                        
                        # Docker check
                        if ! command -v docker >/dev/null 2>&1; then
                            echo "Docker not found. Installing..."
                            curl -fsSL https://get.docker.com | bash
                        else
                            echo "Docker already installed."
                        fi

                        # Docker Compose check
                        if ! command -v docker-compose >/dev/null 2>&1; then
                            echo "Docker Compose not found. Installing..."

                            sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-\$(uname -s)-\$(uname -m)" \
                                -o /usr/local/bin/docker-compose

                            sudo chmod +x /usr/local/bin/docker-compose
                        else
                            echo "Docker Compose already installed."
                        fi

                    '
                    """
                }
            }
        }

        stage('Copy docker-compose.yml to Remote') {
            steps {
                script {
                    sh """
                    echo "Copying docker-compose.yml to remote server..."
                    sshpass -p "$SSH_PASS" scp -o StrictHostKeyChecking=no docker-compose.yml \
                    $SSH_USER@$REMOTE_IP:~/docker-compose.yml
                    """
                }
            }
        }

        stage('Start Redis & Postgres if Not Running') {
            steps {
                script {
                    sh """
                    sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no $SSH_USER@$REMOTE_IP '

                        running=\$(docker ps --format "{{.Names}}")

                        # Redis check
                        if ! echo "\$running" | grep -q "redis"; then
                            echo "Redis not running. Starting..."
                            docker-compose -f ~/docker-compose.yml up -d redis
                        else
                            echo "Redis already running."
                        fi

                        # Postgres check
                        if ! echo "\$running" | grep -q "postgres"; then
                            echo "Postgres not running. Starting..."
                            docker-compose -f ~/docker-compose.yml up -d postgres
                        else
                            echo "Postgres already running."
                        fi

                    '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS: Redis & Postgres containers are running on remote machine!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
