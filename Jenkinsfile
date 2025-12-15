pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote machine IP')
        string(name: 'SSH_USER', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file')
    }

    environment {
        REMOTE_OS = 'UNKNOWN'
    }

    stages {

        /* ================= VALIDATION ================= */

        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.TARGET_IP) error "TARGET_IP is required"
                    if (!params.SSH_USER) error "SSH_USER is required"
                    if (!params.SSH_PASS) error "SSH_PASS is required"
                    if (!fileExists(params.COMPOSE_FILE)) error "Compose file missing"
                }
            }
        }

        /* ================= SSH CHECK ================= */

        stage('SSH Check') {
            steps {
                sh """
                which sshpass >/dev/null || exit 1
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} "echo SSH_OK"
                """
            }
        }

        /* ================= OS DETECTION (FIXED) ================= */

        stage('Detect OS') {
            steps {
                script {
                    def osCheck = sh(
                        script: """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} '
                            if command -v uname >/dev/null 2>&1; then
                                uname -s
                            else
                                echo WINDOWS
                            fi
                        '
                        """,
                        returnStdout: true
                    ).trim()

                    if (osCheck.contains("Linux")) {
                        env.REMOTE_OS = "LINUX"
                    } else if (osCheck.contains("WINDOWS")) {
                        env.REMOTE_OS = "WINDOWS"
                    } else {
                        error "Unable to detect remote OS"
                    }

                    echo "✅ Detected OS: ${env.REMOTE_OS}"
                }
            }
        }

        /* ================= LINUX ================= */

        stage('Ensure Docker (Linux)') {
            when { expression { env.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    set -e
                    sudo -n true || exit 1

                    if ! command -v docker >/dev/null; then
                        curl -fsSL https://get.docker.com | sudo sh
                    fi

                    sudo systemctl enable docker
                    sudo systemctl start docker

                    sudo docker compose version
                '
                """
            }
        }

        /* ================= WINDOWS ================= */

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
                    docker compose version
                "
                """
            }
        }

        /* ================= COPY FILE ================= */

        stage('Copy Compose File') {
            steps {
                script {
                    def path = (env.REMOTE_OS == 'WINDOWS') ?
                        "C:/Users/${params.SSH_USER}/docker-compose.yml" :
                        "~/docker-compose.yml"

                    sh """
                    sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                    ${params.COMPOSE_FILE} ${params.SSH_USER}@${params.TARGET_IP}:${path}
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
                        sshpass -p '${params.SSH_PASS}' ssh \
                        ${params.SSH_USER}@${params.TARGET_IP} '
                            cd ~
                            sudo docker compose down || true
                            sudo docker compose up -d
                        '
                        """
                    } else {
                        sh """
                        sshpass -p '${params.SSH_PASS}' ssh \
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

        stage('Verify Containers') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh \
                ${params.SSH_USER}@${params.TARGET_IP} \
                "docker ps --format 'CONTAINER: {{.Names}} STATUS: {{.Status}}'"
                """
            }
        }
    }

    post {
        success { echo "✅ Deployment successful on ${env.REMOTE_OS}" }
        failure { echo "❌ Deployment failed" }
    }
}
