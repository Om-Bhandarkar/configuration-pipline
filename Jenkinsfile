pipeline {
    agent any

    environment {
        REMOTE_IP = "192.168.1.8"
        REMOTE_USER = "jtsm"
        SSH_KEY = credentials('ssh-remote-key')
        REGISTRY = "192.168.1.10"
        IMAGE_REDIS = "redis:latest"
        IMAGE_POSTGRES = "postgres:latest"
    }

    stages {
        stage('Check Remote OS') {
            steps {
                script {
                    echo "Detecting remote OS..."

                    REMOTE_OS = sh(
                        script: """
                        ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} "uname 2>/dev/null || ver"
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Remote OS detected: ${REMOTE_OS}"
                }
            }
        }

        stage('Install Docker & Compose') {
            steps {
                script {
                    if (REMOTE_OS.contains("Linux")) {

                        echo "Installing Docker on Linux..."

                        sh """
                        ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} '
                            if ! command -v docker >/dev/null 2>&1; then
                                curl -fsSL https://get.docker.com | sh
                            fi

                            if ! command -v docker-compose >/dev/null 2>&1; then
                                sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-\\\$(uname -s)-\\\$(uname -m)" -o /usr/local/bin/docker-compose
                                sudo chmod +x /usr/local/bin/docker-compose
                            fi

                            sudo systemctl enable docker
                            sudo systemctl start docker
                        '
                        """
                    }

                    if (REMOTE_OS.contains("Windows")) {
                        echo "Installing Docker Desktop on Windows..."

                        sh """
                        ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} "
                            if (!(Get-Command docker -ErrorAction SilentlyContinue)) {
                                choco install docker-desktop -y
                            }
                        "
                        """
                    }
                }
            }
        }

        stage('Configure SSH & Firewall') {
            steps {
                script {
                    if (REMOTE_OS.contains("Linux")) {
                        sh """
                        ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} '
                            sudo systemctl enable ssh || sudo systemctl enable sshd
                            sudo systemctl start ssh || sudo systemctl start sshd
                            sudo ufw allow 22 || true
                        '
                        """
                    }

                    if (REMOTE_OS.contains("Windows")) {
                        sh """
                        ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} "
                            Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
                            Start-Service sshd
                            Set-Service -Name sshd -StartupType Automatic
                            netsh advfirewall firewall add rule name='OpenSSH' protocol=TCP dir=in localport=22 action=allow
                        "
                        """
                    }
                }
            }
        }

        stage('Push Images to Registry') {
            steps {
                script {
                    sh """
                    docker pull ${IMAGE_REDIS}
                    docker pull ${IMAGE_POSTGRES}

                    docker tag ${IMAGE_REDIS} ${REGISTRY}/redis
                    docker tag ${IMAGE_POSTGRES} ${REGISTRY}/postgres

                    docker push ${REGISTRY}/redis
                    docker push ${REGISTRY}/postgres
                    """
                }
            }
        }

        stage('Deploy using docker-compose') {
            steps {
                script {
                    echo "Copying docker-compose.yml to remote..."

                    sh """
                    scp -i ${SSH_KEY} docker-compose.yml ${REMOTE_USER}@${REMOTE_IP}:/tmp/docker-compose.yml
                    """

                    echo "Running docker compose..."

                    sh """
                    ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} '
                        cd /tmp
                        docker-compose pull
                        docker-compose up -d
                    '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment Completed Successfully 🚀"
        }
        failure {
            echo "Pipeline Failed ❌"
        }
    }
}
