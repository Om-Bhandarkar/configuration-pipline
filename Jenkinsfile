pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote machine IP (Linux/Windows)')
        string(name: 'SSH_USER', description: 'SSH username')
        password(name: 'SSH_PASS', description: 'SSH password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml')
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
                which sshpass >/dev/null || exit 1
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} "echo SSH_OK"
                """
            }
        }

        /* ================= OS DETECTION ================= */

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

        /* ================= LINUX FLOW ================= */

        stage('Install Docker & Compose (Linux)') {
            when { expression { env.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} '
                    set -e
                    if ! command -v docker >/dev/null; then
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

        /* ================= WINDOWS FLOW ================= */

        stage('Check Docker Desktop (Windows)') {
            when { expression { env.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                powershell -NoProfile -Command "& {
                    if (!(Get-Command docker -ErrorAction SilentlyContinue)) {
                        Write-Error 'Docker Desktop not installed or not running'
                        exit 1
                    }
                    docker compose version
                }"
                """
            }
        }

        /* ================= COMMON ================= */

        stage('Copy Compose File') {
            steps {
                script {
                    def path = (env.REMOTE_OS == 'WINDOWS')
                        ? "C:/Users/${params.SSH_USER}/docker-compose.yml"
                        : "~/docker-compose.yml"

                    sh """
                    sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                    ${params.COMPOSE_FILE} ${params.SSH_USER}@${params.TARGET_IP}:${path}
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
                            docker compose down --remove-orphans || true
                            docker compose up -d
                        '
                        """
                    } else {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                        powershell -NoProfile -Command "& {
                            cd C:/Users/${params.SSH_USER}
                            docker compose down --remove-orphans
                            docker compose up -d
                        }"
                        """
                    }
                }
            }
        }

        stage('Verify Containers') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                ${env.REMOTE_OS == 'WINDOWS'
                    ? "powershell -Command \"& { docker ps }\""
                    : "docker ps"}
                """
            }
        }
    }

    post {
        success { echo "✅ Deployment successful on ${env.REMOTE_OS}" }
        failure { echo "❌ Deployment failed" }
    }
}
