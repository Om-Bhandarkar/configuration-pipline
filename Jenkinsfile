pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Target Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE = ""
        COMPOSE_DIR_LINUX  = "/infra"
        COMPOSE_FILE_LINUX = "/infra/docker-compose.yml"

        COMPOSE_DIR_WIN    = "C:/infra"
        COMPOSE_FILE_WIN   = "C:/infra/docker-compose.yml"
    }

    stages {

        /* -------------------------
           1) CHECK CONNECTION
        ------------------------- */
        stage("Check Connection") {
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "echo Connected OK"
                """
            }
        }

        /* -------------------------
           2) DETECT OS
        ------------------------- */
        stage("Detect OS") {
            steps {
                echo "🔍 Detecting OS..."
                script {

                    def linuxCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "uname" || true
                        """
                    ).trim()

                    if (linuxCheck.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "linux"
                        echo "🟢 Linux detected"
                        return
                    }

                    def winCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -command \\\"[System.Environment]::OSVersion.Platform\\\""
                        """
                    ).trim()

                    if (winCheck.toLowerCase().contains("win32nt")) {
                        env.OS_TYPE = "windows"
                        echo "🟦 Windows detected"
                        return
                    }

                    error "❌ Unknown OS. Linux: ${linuxCheck}, Windows: ${winCheck}"
                }
            }
        }

        /* -------------------------
           3) INSTALL DOCKER (LINUX)
        ------------------------- */
        stage("Install Docker (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        if ! command -v docker >/dev/null; then
                            apt-get update -y || yum update -y
                            apt-get install -y docker.io || yum install -y docker
                            systemctl enable docker || true
                            systemctl start docker || true
                        fi

                        if ! command -v docker-compose >/dev/null; then
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        fi
                    '
                """
            }
        }

        /* -------------------------
           4) INSTALL DOCKER (WINDOWS)
        ------------------------- */
        stage("Install Docker (Windows)") {
            when { expression { env.OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                    "powershell -Command \\"Write-Host 'Windows detected. Install Docker Desktop manually or via winget.'\\""
                """
            }
        }

        /* -------------------------
           5) UPLOAD docker-compose.yml
        ------------------------- */
        stage("Upload docker-compose.yml") {
            steps {
                script {

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${COMPOSE_DIR_LINUX}"
                            sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_LINUX}
                        """
                    }

                    if (env.OS_TYPE == "windows") {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \\"New-Item -ItemType Directory -Force -Path '${COMPOSE_DIR_WIN}'\\""
                            
                            sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_WIN}
                        """
                    }
                }
            }
        }

        /* -------------------------
           6) RUN docker-compose (LINUX)
        ------------------------- */
        stage("Run docker-compose (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        cd ${COMPOSE_DIR_LINUX}
                        docker-compose down || true
                        docker-compose up -d
                    '
                """
            }
        }

        /* -------------------------
           7) VERIFY INSTALLATION
        ------------------------- */
        stage("Verify Installation") {
            steps {
                script {
                    def IP = params.TARGET_IP.trim()

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${IP} "
                                echo '===== VERIFY (LINUX) =====';
                                docker --version;
                                docker-compose --version;
                                systemctl is-active docker;
                                docker ps;
                                ls -l ${COMPOSE_FILE_LINUX} || echo '❌ Compose file missing';
                                echo '==========================';
                            "
                        """
                    } else {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${IP} powershell -Command "
                                Write-Host '===== VERIFY (WINDOWS) =====';
                                docker --version;
                                docker-compose --version;
                                docker ps;
                                Test-Path '${COMPOSE_FILE_WIN}';
                                Write-Host '==============================';
                            "
                        """
                    }
                }
            }
        }
    }

    post {
        success { echo "🎉 Deployment Successful" }
        failure { echo "❌ Deployment Failed" }
    }
}
