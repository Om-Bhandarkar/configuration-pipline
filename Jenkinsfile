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

        stage("Check Connection") {
            steps {
                sh """
                    echo "Checking SSH connectivity ..."
                    sshpass -p "${params.SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "echo Connected OK"
                """
            }
        }

        stage("Detect OS") {
            steps {
                echo "Detecting OS..."
                script {

                    def linuxCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "uname" || true
                        """
                    ).trim()

                    echo "Linux check output: ${linuxCheck}"

                    if (linuxCheck.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "linux"
                        echo "OS Detected: Linux"
                        return
                    }

                    def winCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \\
                            "powershell -command \\\"[System.Environment]::OSVersion.Platform\\\""
                        """
                    ).trim()

                    echo "Windows check output: ${winCheck}"

                    if (winCheck.toLowerCase().contains("win32nt")) {
                        env.OS_TYPE = "windows"
                        echo "OS Detected: Windows"
                        return
                    }

                    error "Could not detect OS!"
                }
            }
        }

        stage("Install Docker (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    echo "Installing Docker if needed..."
                    sshpass -p "${params.SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        set -e

                        if ! command -v docker >/dev/null 2>&1; then
                            echo "Docker not found. Installing..."
                            if command -v apt-get >/dev/null 2>&1; then
                                apt-get update -y
                                apt-get install -y docker.io
                            elif command -v yum >/dev/null 2>&1; then
                                yum update -y
                                yum install -y docker
                            else
                                echo "No apt-get or yum found."
                                exit 1
                            fi
                        fi

                        if command -v systemctl >/dev/null 2>&1; then
                            systemctl start docker || true
                            systemctl enable docker || true
                        fi

                        if ! command -v docker-compose >/dev/null 2>&1 && [ ! -x /usr/local/bin/docker-compose ]; then
                            echo "Installing docker-compose..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \\
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        fi
                    '
                """
            }
        }

        stage("Install Docker (Windows)") {
            when { expression { env.OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \\
                    "powershell -Command \\"Write-Host 'Windows detected. Install Docker manually.'\\""
                """
            }
        }

        stage("Upload docker-compose.yml") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${COMPOSE_DIR_LINUX}"
                            sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml \\
                            ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_LINUX}
                        """
                    }

                    if (env.OS_TYPE == "windows") {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \\
                            "powershell -Command \\"New-Item -ItemType Directory -Force -Path '${COMPOSE_DIR_WIN}'\\""

                            sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml \\
                            ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_WIN}
                        """
                    }
                }
            }
        }

        stage("Run docker-compose (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        export PATH=$PATH:/usr/local/bin
                        cd ${COMPOSE_DIR_LINUX}

                        if command -v docker-compose >/dev/null 2>&1; then
                            docker-compose down || true
                            docker-compose up -d
                        elif [ -x /usr/local/bin/docker-compose ]; then
                            /usr/local/bin/docker-compose down || true
                            /usr/local/bin/docker-compose up -d
                        elif docker compose version >/dev/null 2>&1; then
                            docker compose down || true
                            docker compose up -d
                        else
                            echo "No docker-compose found."
                            exit 1
                        fi

                        docker ps -a
                    '
                """
            }
        }
    }

    post {
        success { echo "Deployment Successful" }
        failure { echo "Deployment Failed" }
    }
}
