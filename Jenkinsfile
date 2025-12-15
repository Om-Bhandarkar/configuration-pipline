pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', defaultValue: '', description: 'Remote Linux/Windows machine IP')
        string(name: 'SSH_USER', defaultValue: 'om', description: 'SSH Username')
        password(name: 'SSH_PASS', defaultValue: '', description: 'SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file')
    }

    environment {
        REMOTE_OS = ''
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

        /* ========= OS DETECTION ========= */
        stage('Detect Remote OS') {
            steps {
                script {
                    def os = sh(
                        script: """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} \
                        "test -f /etc/os-release && echo LINUX || echo WINDOWS"
                        """,
                        returnStdout: true
                    ).trim()

                    env.REMOTE_OS = os
                    echo "✅ Detected OS: ${env.REMOTE_OS}"
                }
            }
        }
        /* =============================== */

        /* ========== LINUX FLOW (UNCHANGED) ========== */
        stage('Ensure Docker & Compose (Linux)') {
            when { expression { env.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    set -e

                    if ! command -v docker >/dev/null 2>&1; then
                        echo "Installing Docker..."
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
        /* =========================================== */

        /* ========== WINDOWS FLOW (ADDED) ========== */
        stage('Verify Docker Desktop (Windows)') {
            when { expression { env.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                "powershell -Command \\
                    if (!(Get-Command docker -ErrorAction SilentlyContinue)) { \\
                        Write-Error 'Docker Desktop not installed or not running'; exit 1 \\
                    } \\
                    docker compose version"
                """
            }
        }
        /* ========================================== */

        stage('Copy Compose File') {
            steps {
                script {
                    if (env.REMOTE_OS == 'LINUX') {
                        sh """
                        sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                        ${params.COMPOSE_FILE} ${params.SSH_USER}@${params.TARGET_IP}:~/docker-compose.yml
                        """
                    } else {
                        sh """
                        sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                        ${params.COMPOSE_FILE} ${params.SSH_USER}@${params.TARGET_IP}:C:/Users/${params.SSH_USER}/docker-compose.yml
                        """
                    }
                }
            }
        }

        stage('Deploy Containers') {
            steps {
                script {
                    if (env.REMOTE_OS == 'LINUX') {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} '
                            cd ~
                            docker compose up -d --remove-orphans
                        '
                        """
                    } else {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                        "powershell -Command \\
                            cd C:/Users/${params.SSH_USER}; \\
                            docker compose up -d --remove-orphans"
                        """
                    }
                }
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
        success {
            echo "✅ Deployment successful on ${env.REMOTE_OS}"
        }
        failure {
            echo "❌ Deployment failed"
        }
    }
}
