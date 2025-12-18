pipeline {
    agent any

    parameters {
        choice(
            name: 'REMOTE_OS',
            choices: ['LINUX', 'WINDOWS'],
            description: 'Target machine operating system'
        )
        string(name: 'TARGET_IP', defaultValue: '', description: 'Remote machine IP')
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

        /* ===================== LINUX ===================== */

        stage('Ensure Docker & Compose (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    set -e
                    if ! command -v docker >/dev/null 2>&1; then
                        curl -fsSL https://get.docker.com | sudo sh
                    fi
                    sudo systemctl enable docker
                    sudo systemctl start docker
                    sudo usermod -aG docker ${params.SSH_USER}
                    docker compose version
                '
                """
            }
        }

        stage('Copy Compose File (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                ${params.COMPOSE_FILE} \
                ${params.SSH_USER}@${params.TARGET_IP}:~/docker-compose.yml
                """
            }
        }

        stage('Deploy Containers (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
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

        stage('Verify Containers (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} \
                "docker ps"
                """
            }
        }

        /* ===================== WINDOWS ===================== */

        stage('Verify Docker Desktop (Windows)') {
            when { expression { params.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                sshpass -p "${params.SSH_PASS}" ssh ${params.SSH_USER}@${params.TARGET_IP} \
                powershell -NoProfile -Command "docker version && docker compose version"
                """
            }
        }

        stage('Copy Compose File (Windows)') {
            when { expression { params.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no \
                ${params.COMPOSE_FILE} \
                ${params.SSH_USER}@${params.TARGET_IP}:C:/Users/${params.SSH_USER}/docker-compose.yml
                """
            }
        }

       
        stage('Deploy Containers (Windows)') {
            when { expression { params.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                PS_CMD=\$'\
                \$env:DOCKER_CONFIG=\"C:/Users/${params.SSH_USER}/.docker-ci\"; \
                New-Item -ItemType Directory -Force \$env:DOCKER_CONFIG | Out-Null; \
                Set-Location C:/Users/${params.SSH_USER}; \
                docker compose -f C:/Users/${params.SSH_USER}/docker-compose.yml down --remove-orphans; \
                docker compose -f C:/Users/${params.SSH_USER}/docker-compose.yml up -d'
                ENCODED=\$(echo \"\$PS_CMD\" | iconv -t UTF-16LE | base64 -w 0)
                sshpass -p \"${params.SSH_PASS}\" ssh ${params.SSH_USER}@${params.TARGET_IP} \"powershell -NoProfile -EncodedCommand \$ENCODED\"
                """
            }
        }

    }

    post {
        success {
            echo "✅ Deployment successful on ${params.REMOTE_OS}"
        }
        failure {
            echo "❌ Deployment failed"
        }
    }
}
