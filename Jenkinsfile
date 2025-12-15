pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote machine IP')
        string(name: 'SSH_USER', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file')
    }

    environment {
        REMOTE_OS = ''
    }

    stages {

        /* ================= VALIDATION ================= */

        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.TARGET_IP) error "TARGET_IP is required"
                    if (!params.SSH_USER) error "SSH_USER is required"
                    if (!params.SSH_PASS) error "SSH_PASS is required"
                    if (!fileExists(params.COMPOSE_FILE)) error "docker-compose.yml not found in workspace"
                }
            }
        }

        /* ================= SSH CHECK ================= */

        stage('SSH Check') {
            steps {
                sh """
                which sshpass >/dev/null || (echo 'sshpass not installed on Jenkins agent' && exit 1)
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} "echo SSH_OK"
                """
            }
        }

        /* ================= OS DETECTION ================= */

        stage('Detect OS') {
            steps {
                script {
                    def os = sh(
                        script: """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "uname -s" 2>/dev/null || echo WINDOWS
                        """,
                        returnStdout: true
                    ).trim()

                    env.REMOTE_OS = os.contains("Linux") ? "LINUX" : "WINDOWS"
                    echo "Detected OS: ${env.REMOTE_OS}"
                }
            }
        }

        /* ================= LINUX SETUP ================= */

        stage('Ensure Docker & Compose (Linux)') {
            when { expression { env.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    set -e

                    # Check passwordless sudo
                    sudo -n true || { echo "Passwordless sudo required"; exit 1; }

                    # Install Docker if missing
                    if ! command -v docker >/dev/null 2>&1; then
                        echo "Installing Docker..."
                        curl -fsSL https://get.docker.com | sudo sh
                    fi

                    # systemctl check
                    command -v systemctl >/dev/null || { echo "systemctl not available"; exit 1; }

                    sudo systemctl enable docker
                    sudo systemctl start docker

                    # Add user to docker group (for future sessions)
                    sudo usermod -aG docker ${params.SSH_USER}

                    # Verify docker compose (run with sudo for current session)
                    sudo docker compose version
                '
                """
            }
        }

        /* ================= WINDOWS CHECK ================= */

        stage('Verify Docker Desktop (Windows)') {
            when { expression { env.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} powershell -Command "
                    if (!(Get-Command docker -ErrorAction SilentlyContinue)) {
                        Write-Error 'Docker Desktop not installed'
                        exit 1
                    }

                    # Wait for Docker Desktop to be ready
                    \$retry = 0
                    while (\$retry -lt 20) {
                        if (docker info 2>\$null) { break }
                        Start-Sleep 5
                        \$retry++
                    }

                    if (\$retry -eq 20) {
                        Write-Error 'Docker Desktop not ready'
                        exit 1
                    }

                    docker compose version
                "
                """
            }
        }

        /* ================= COPY COMPOSE FILE ================= */

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

        /* ================= DEPLOY ================= */

        stage('Deploy Containers') {
            steps {
                script {
                    if (env.REMOTE_OS == 'LINUX') {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} '
                            cd ~
                            sudo docker compose down || true
                            sudo docker compose up -d --remove-orphans
                        '
                        """
                    } else {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} powershell -Command "
                            cd \$HOME
                            docker compose down 2>\$null
                            docker compose up -d
                        "
                        """
                    }
                }
            }
        }

        /* ================= VERIFY ================= */

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
