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
                    echo "🔗 Checking SSH connectivity to ${params.SSH_USER}@${params.TARGET_IP} ..."
                    sshpass -p "${params.SSH_PASS}" \\
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

                    // --- Try Linux check first ---
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
                        echo "🖥️ OS Detected: Linux"
                        return
                    }

                    // --- Try Windows check ---
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
                        echo "🖥️ OS Detected: Windows"
                        return
                    }

                    error "❌ Could not detect OS! Linux output: ${linuxCheck}, Windows output: ${winCheck}"
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
                    echo "🐧 Installing Docker & docker-compose on Linux if needed..."
                    sshpass -p "${params.SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        set -e

                        echo "Checking Docker..."
                        if ! command -v docker >/dev/null 2>&1; then
                            echo "Docker not found. Installing..."
                            if command -v apt-get >/dev/null 2>&1; then
                                apt-get update -y
                                apt-get install -y docker.io
                            elif command -v yum >/dev/null 2>&1; then
                                yum update -y
                                yum install -y docker
                            else
                                echo "❌ Neither apt-get nor yum found. Install Docker manually."
                                exit 1
                            fi
                        else
                            echo "✅ Docker already installed: \$(docker --version)"
                        fi

                        echo "Ensuring Docker service is running..."
                        if command -v systemctl >/dev/null 2>&1; then
                            systemctl start docker || true
                            systemctl enable docker || true
                        fi

                        echo "Checking docker-compose..."
                        if ! command -v docker-compose >/dev/null 2>&1 && [ ! -x /usr/local/bin/docker-compose ]; then
                            echo "docker-compose not found. Installing binary in /usr/local/bin..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \\
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        else
                            echo "✅ docker-compose already present."
                        fi

                        echo "Docker info:"
                        docker info || echo "⚠️ docker info failed (but continuing)"
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
                    sshpass -p "${params.SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \\
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
                            echo "📤 Uploading docker-compose.yml to Linux path ${COMPOSE_FILE_LINUX} ..."
                            sshpass -p "${params.SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${COMPOSE_DIR_LINUX}"

                            sshpass -p "${params.SSH_PASS}" \\
                            scp -o StrictHostKeyChecking=no docker-compose.yml \\
                            ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_LINUX}

                            echo "✅ File uploaded. Remote listing:"
                            sshpass -p "${params.SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "ls -l ${COMPOSE_DIR_LINUX}"
                        """
                    }

                    if (env.OS_TYPE == "windows") {
                        sh """
