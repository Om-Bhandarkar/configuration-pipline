pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP',  description: 'Target Server IP')
        string(name: 'SSH_USER',   defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE = ""
        LINUX_DIR = "/infra"
        LINUX_COMPOSE = "/infra/docker-compose.yml"
        WIN_DIR = "C:/infra"
        WIN_COMPOSE = "C:/infra/docker-compose.yml"
    }

    stages {

        /* ---------------------------------------------------
           0) Validate docker-compose.yml Exists
        --------------------------------------------------- */
        stage("Validate Compose File") {
            steps {
                script {
                    if (!fileExists("docker-compose.yml")) {
                        error "❌ docker-compose.yml NOT found in workspace!"
                    }
                }
            }
        }

        /* ---------------------------------------------------
           1) Validate sshpass is installed on Jenkins Agent
        --------------------------------------------------- */
        stage("Check sshpass Installed") {
            steps {
                script {
                    def installed = sh(returnStatus: true, script: "command -v sshpass >/dev/null") == 0
                    if (!installed) {
                        error """
❌ ERROR: sshpass is not installed on the Jenkins Agent.

Install using:
    sudo apt install -y sshpass
OR
    sudo yum install -y sshpass
                        """
                    }
                }
            }
        }

        /* ---------------------------------------------------
           2) Test SSH Connection
        --------------------------------------------------- */
        stage("Check SSH Connection") {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "echo 'SSH OK ✔'"
                """
            }
        }

        /* ---------------------------------------------------
           3) Detect OS (Linux vs Windows)
        --------------------------------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting OS..."

                    def checkLinux = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "uname 2>/dev/null" || true
                        """
                    ).trim().toLowerCase()

                    if (checkLinux.contains("linux")) {
                        OS_TYPE = "linux"
                        echo "🟢 Detected Linux OS"
                        return
                    }

                    def checkWin = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${SSH_PASS}" ssh \
                            -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                            "powershell -command \\"(Get-CimInstance Win32_OperatingSystem).Caption\\"" || ""
                        """
                    ).trim().toLowerCase()

                    if (checkWin.contains("windows")) {
                        OS_TYPE = "windows"
                        echo "🟦 Detected Windows OS"
                        return
                    }

                    error "❌ Unable to detect OS!"
                }
            }
        }

        /* ---------------------------------------------------
           4) Install Docker + Compose (Linux)
        --------------------------------------------------- */
        stage("Setup Docker (Linux)") {
            when { expression { OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        set -e

                        echo "Checking Docker..."
                        if ! command -v docker >/dev/null; then
                            echo "Installing Docker..."
                            if command -v apt-get >/dev/null; then
                                apt-get update -y
                                apt-get install -y docker.io
                            elif command -v yum >/dev/null; then
                                yum install -y docker
                            elif command -v dnf >/dev/null; then
                                dnf install -y docker
                            elif command -v zypper >/dev/null; then
                                zypper install -y docker
                            else
                                echo "No package manager found — using get.docker.com"
                                curl -fsSL https://get.docker.com | sh
                            fi

                            systemctl enable docker || true
                            systemctl start docker || true
                        else
                            echo "✔ Docker already installed"
                        fi

                        echo "Checking Docker Compose..."
                        if ! command -v docker-compose >/dev/null; then
                            echo "Installing Docker Compose..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        else
                            echo "✔ Docker Compose already installed"
                        fi
                    '
                """
            }
        }

        /* ---------------------------------------------------
           5) Setup Docker on Windows
        --------------------------------------------------- */
        stage("Setup Docker (Windows)") {
            when { expression { OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                    ${SSH_USER}@${TARGET_IP} \
                    "powershell -command \\
                    \\"if (!(docker --version)) { Write-Host '❌ Docker not installed. Install Docker Desktop manually or via winget.' } else { Write-Host '✔ Docker Installed' }; \\
                    if (!(docker compose version)) { Write-Host '❌ docker compose missing'; } else { Write-Host '✔ docker compose OK' }\\""
                """
            }
        }

        /* ---------------------------------------------------
           6) Upload docker-compose.yml
        --------------------------------------------------- */
        stage("Upload Compose File") {
            steps {
                script {
                    if (OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "mkdir -p ${LINUX_DIR}"
                            sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${LINUX_COMPOSE}
                        """
                    }
                    else {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                            "powershell -command \\"New-Item -ItemType Directory -Force -Path '${WIN_DIR}'\\""
                            
                            sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${WIN_COMPOSE}
                        """
                    }
                }
            }
        }

        /* ---------------------------------------------------
           7) Run Docker Compose (Linux Only)
        --------------------------------------------------- */
        stage("Run Compose (Linux)") {
            when { expression { OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                    ${SSH_USER}@${TARGET_IP} '
                        cd ${LINUX_DIR}
                        docker-compose down || true
                        docker-compose up -d
                    '
                """
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS: Infra Setup Completed for ${TARGET_IP}"
        }
        failure {
            echo "❌ FAIL: Infra Setup Failed — check logs above"
        }
    }
}
