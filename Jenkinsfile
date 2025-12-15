pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', defaultValue: '', description: 'Remote Linux machine IP')
        string(name: 'SSH_USER', defaultValue: 'om', description: 'SSH Username')
        password(name: 'SSH_PASS', defaultValue: '', description: 'SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file')
    }

    stages {

        stage('Validate Inputs') { 	
            steps {
                script {
                    if (!params.TARGET_IP) error "TARGET_IP is required"
                    if (!params.SSH_USER) error "SSH_USER is required"
                    if (!params.SSH_PASS) error "SSH_PASS is required"
                    if (!fileExists(params.COMPOSE_FILE)) error "Compose file not found"
                }
            }
        }

        stage('SSH Check') {
            steps {
                sh """
                which sshpass >/dev/null || exit 2
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} 'echo SSH_OK'
                """
            }
        }

        stage('Ensure Docker & Compose (Linux)') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '

                set -e

                # Install Docker if missing
                if ! command -v docker >/dev/null 2>&1; then
                    echo "Installing Docker..."
                    curl -fsSL https://get.docker.com | sudo sh
                fi

                # Enable & start Docker
                sudo systemctl enable docker
                sudo systemctl start docker

                # Add user to docker group
                sudo usermod -aG docker ${params.SSH_USER}

                # Verify docker compose
                if ! docker compose version >/dev/null 2>&1; then
                    echo "Docker Compose v2 not available"
                    exit 1
                fi

                echo "Docker & Compose ready"
                '
                """
            }
        }

        stage('Copy Compose File') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                ${params.COMPOSE_FILE} ${params.SSH_USER}@${params.TARGET_IP}:~/docker-compose.yml
                """
            }
        }

        stage('Deploy Containers') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    cd ~
                    docker compose up -d --remove-orphans
                '
                """
            }
        }

        stage('Verify Containers') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} \
                "docker ps --format 'CONTAINER: {{.Names}} STATUS: {{.Status}}'"
                """
            }
        }
    }

    post {
        success { echo "✅ Deployment successful on Linux" }
        failure { echo "❌ Deployment failed" }
    }
}
