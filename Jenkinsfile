pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote machine IP (Linux or Windows)')
        string(name: 'SSH_USER', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file')
    }

    environment {
        REMOTE_OS = ''
    }

    stages {

        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.TARGET_IP) error "TARGET_IP required"
                    if (!params.SSH_USER) error "SSH_USER required"
                    if (!params.SSH_PASS) error "SSH_PASS required"
                    if (!fileExists(params.COMPOSE_FILE)) error "docker-compose.yml not found"
                }
            }
        }

        stage('SSH Connectivity Check') {
            steps {
                sh """
                which sshpass >/dev/null || (echo 'sshpass not installed on Jenkins' && exit 1)
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} "echo SSH_OK"
                """
            }
        }

        stage('Detect Remote OS') {
            steps {
                script {
                    def os = sh(
                        script: """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} '
                            if command -v powershell.exe >/dev/null 2>&1; then
                                echo WINDOWS
                            else
                                echo LINUX
                            fi
                        '
                        """,
                        returnStdout: true
                    ).trim()

                    env.REMOTE_OS = os
                    echo "✅ Detected OS: ${env.REMOTE_OS}"
                }
            }
        }

        /* ===================== LINUX ===================== */

        stage('Ensure Docker & Compose (Linux)') {
            when { expression { env.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} '
                    set -e

                    if ! command -v docker >/dev/null 2>&1; then
                        echo "Installing Docker..."
                        curl -fsSL https://get.docker.com | sudo sh
                    fi

                    sudo systemctl enable docker
                    sudo systemctl start docker

                    sudo usermod -aG docker ${params.SSH_USER} || true

                    docker compose version
                '
                """
            }
        }

        /* ===================== WINDOWS ===================== */

        stage('Verify Docker Desktop (Windows)') {
            when { expression { env.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                powershell -Command "
                    if (!(Get-Command docker -ErrorAction SilentlyContinue)) {
                        Write-Error 'Docker Desktop not installed or not running'
                        exit 1
                    }
                    docker compose version
                "
                """
            }
        }

        /* ===================== COMMON ===================== */

        stage('Copy Compose File') {
            steps {
                script {
                    def remotePath = (env.REMOTE_OS == 'WINDOWS') ?
                        "C:/Users/${params.SSH_USER}/docker-compose.yml" :
                        "~/docker-compose.yml"

                    sh """
                    sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                    ${params.COMPOSE_FILE} ${params.SSH_USER}@${params.TARGET_IP}:${remotePath}
                    """
                }
            }
        }

        stage('Deploy Containers') {
            steps {
                script {
                    if (env.REMOTE_OS == 'LINUX') {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} '
                            cd ~
                            docker compose down || true
                            docker compose up -d --remove-orphans
                        '
                        """
                    } else {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                        powershell -Command "
                            cd C:/Users/${params.SSH_USER}
                            docker compose down
                            docker compose up -d
                        "
                        """
                    }
                }
            }
        }

        stage('Verify Containers') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
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
