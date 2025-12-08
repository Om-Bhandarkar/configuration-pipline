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
                script {
                    def out = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "uname 2>/dev/null || ver"
                        """
                    ).trim()

                    if (out.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "linux"
                    } else if (out.toLowerCase().contains("windows") || out.toLowerCase().contains("microsoft")) {
                        env.OS_TYPE = "windows"
                    } else {
                        error "Unknown OS detected: ${out}"
                    }

                    echo "Detected OS: ${env.OS_TYPE}"
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
                            systemctl start docker || true
                            systemctl enable docker || true
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
           5) UPLOAD COMPOSE FILE (OS-wise)
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
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "powershell -Command \\"New-Item -ItemType Directory -Force -Path '${COMPOSE_DIR_WIN}'\\""
                            sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_WIN}
                        """
                    }
                }
            }
        }

        /* -------------------------
           6) RUN COMPOSE (LINUX ONLY)
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
           WINDOWS IS NOT SUPPORTED FOR docker-compose directly here
        ------------------------- */
    }

    post {
        success { echo "🎉 Deployment Successful" }
        failure { echo "❌ Deployment Failed" }
    }
}
