pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP',  description: 'Target Server IP')
        string(name: 'SSH_USER',   defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE        = ""
        LINUX_DIR      = "/infra"
        LINUX_COMPOSE  = "/infra/docker-compose.yml"
        WIN_DIR        = "C:/infra"
        WIN_COMPOSE    = "C:/infra/docker-compose.yml"
    }

    stages {

        /* ---------------------------------------------------
           0) Validate docker-compose.yml exists
        --------------------------------------------------- */
        stage("Validate docker-compose.yml") {
            steps {
                script {
                    if (!fileExists("docker-compose.yml")) {
                        error "❌ docker-compose.yml workspace मध्ये सापडला नाही!"
                    }
                }
            }
        }

        /* ---------------------------------------------------
           1) Ensure sshpass installed on Jenkins agent
        --------------------------------------------------- */
        stage("Check sshpass") {
            steps {
                script {
                    if (sh(returnStatus: true, script: "command -v sshpass >/dev/null 2>&1") != 0) {
                        error """
❌ ERROR: Jenkins agent वर sshpass install नाही.

Install:
  sudo apt install -y sshpass
  OR
  sudo yum install -y sshpass
                        """
                    }
                }
            }
        }

        /* ---------------------------------------------------
           2) SSH Connection Test
        --------------------------------------------------- */
        stage("Check SSH Connection") {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \
                    ssh -o ConnectTimeout=8 -o StrictHostKeyChecking=no \
                        ${SSH_USER}@${TARGET_IP} "echo 'SSH Connected ✔'" 
                """
            }
        }

        /* ---------------------------------------------------
           3) Detect OS
        --------------------------------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting remote OS..."

                    def lc = sh(returnStdout: true, script: """
                        sshpass -p "${SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "uname 2>/dev/null" || true
                    """).trim().toLowerCase()

                    if (lc.contains("linux")) {
                        OS_TYPE = "linux"
                        echo "🟢 Linux detected"
                        return
                    }

                    def wc = sh(returnStdout: true, script: """
                        sshpass -p "${SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                        "powershell -Command \\"(Get-CimInstance Win32_OperatingSystem).Caption\\"" || ""
                    """).trim().toLowerCase()

                    if (wc.contains("windows")) {
                        OS_TYPE = "windows"
                        echo "🟦 Windows detected: ${wc}"
                        return
                    }

                    error "❌ Unable to detect OS!"
                }
            }
        }

        /* ---------------------------------------------------
           4) Linux: Ensure SSH service + firewall
        --------------------------------------------------- */
        stage("Setup SSH & Firewall (Linux)") {
            when { expression { OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        set -e
                        echo "🔐 Configuring SSH (Linux)..."

                        if ! command -v sshd >/dev/null; then
                            echo "Installing OpenSSH server..."
                            apt-get update -y || yum update -y
                            apt-get install -y openssh-server || yum install -y openssh-server
                        fi

                        systemctl enable ssh || systemctl enable sshd || true
                        systemctl start ssh || systemctl start sshd || true

                        if command -v ufw >/dev/null; then
                            ufw allow 22 || true
                            ufw reload || true
                        elif command -v firewall-cmd >/dev/null; then
                            firewall-cmd --add-port=22/tcp --permanent || true
                            firewall-cmd --reload || true
                        fi

                        echo "SSH Ready ✔"
                    '
                """
            }
        }

        /* ---------------------------------------------------
           5) Windows: Ensure SSH running (SSH must be installed)
        --------------------------------------------------- */
        stage("Setup SSH (Windows)") {
            when { expression { OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                    "powershell -Command \\
                        \\"Write-Host 'Checking SSH...'; \\
                        \$svc = Get-Service sshd -ErrorAction SilentlyContinue; \\
                        if (\$svc) { \\
                            Set-Service sshd -StartupType Automatic; Start-Service sshd; \\
                            if (-not (Get-NetFirewallRule -DisplayName 'OpenSSH')) { \\
                                New-NetFirewallRule -DisplayName 'OpenSSH' -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow; \\
                            } \\
                            Write-Host 'SSH Ready ✔'; \\
                        } else { \\
                            Write-Host '❌ OpenSSH installed नाही. First time manually install करणे आवश्यक.'; \\
                        }\\""
                """
            }
        }

        /* ---------------------------------------------------
           6) Install Docker + Compose (Linux)
        --------------------------------------------------- */
        stage("Docker Setup (Linux)") {
            when { expression { OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        set -e
                        echo "🐳 Docker Setup (Linux)..."

                        if ! command -v docker >/dev/null; then
                            echo "Installing Docker..."
                            apt-get install -y docker.io || yum install -y docker || dnf install -y docker \
                                || zypper install -y docker || curl -fsSL https://get.docker.com | sh
                            systemctl enable docker || true
                            systemctl start docker || true
                        fi

                        echo "Docker Version:"
                        docker --version

                        if ! command -v docker-compose >/dev/null; then
                            echo "Installing Docker Compose..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\$(uname -s)-\$(uname -m)" \
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        fi

                        docker-compose --version
                    '
                """
            }
        }

        /* ---------------------------------------------------
           7) Windows: Install Docker Desktop (via winget)
        --------------------------------------------------- */
        stage("Docker Setup (Windows)") {
            when { expression { OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                    "powershell -Command \\
                        \\"if (!(docker --version)) { \\
                            Write-Host 'Docker Install Attempt via winget...'; \\
                            if (Get-Command winget -ErrorAction SilentlyContinue) { \\
                                winget install -e --id Docker.DockerDesktop -h --accept-package-agreements --accept-source-agreements; \\
                            } else { Write-Host '❌ winget not available, install Docker manually.' } \\
                        } else { docker --version } \\
                        if (docker compose version) { docker compose version } \\
                        elseif (Get-Command docker-compose -ErrorAction SilentlyContinue) { docker-compose --version } \\
                        else { Write-Host '❌ docker compose missing.' }\\""
                """
            }
        }

        /* ---------------------------------------------------
           8) Upload docker-compose.yml
        --------------------------------------------------- */
        stage("Upload Compose File") {
            steps {
                script {
                    def targetPath = (OS_TYPE == "linux") ? LINUX_COMPOSE : WIN_COMPOSE
                    def dirPath    = (OS_TYPE == "linux") ? LINUX_DIR     : WIN_DIR

                    if (OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "mkdir -p ${dirPath}"
                            sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${TARGET_IP}:${targetPath}
                        """
                    } else {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                                "powershell -Command \\"New-Item -Force -ItemType Directory -Path '${dirPath}'\\""
                            sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${TARGET_IP}:${targetPath}
                        """
                    }
                }
            }
        }

        /* ---------------------------------------------------
           9) Run Docker Compose (Linux)
        --------------------------------------------------- */
        stage("Run Compose (Linux)") {
            when { expression { OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        cd ${LINUX_DIR}
                        docker-compose up -d
                    '
                """
            }
        }

        /* ---------------------------------------------------
           10) Run Docker Compose (Windows)
        --------------------------------------------------- */
        stage("Run Compose (Windows)") {
            when { expression { OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                    "powershell -Command \\
                        \\"if (docker compose version) { docker compose -f ${WIN_COMPOSE} up -d } \\
                          elseif (Get-Command docker-compose -ErrorAction SilentlyContinue) { docker-compose -f ${WIN_COMPOSE} up -d } \\
                          else { Write-Host '❌ Cannot run docker compose' }\\""
                """
            }
        }
    }

    post {
        success { echo "🎉 Infrastructure setup completed successfully!" }
        failure { echo "❌ Deployment failed. Check logs." }
    }
}
