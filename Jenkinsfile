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
           0) docker-compose.yml workspace मध्ये आहे का?
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
           1) Jenkins agent वर sshpass आहे का?
        --------------------------------------------------- */
        stage("Check sshpass on Jenkins agent") {
            steps {
                script {
                    int status = sh(returnStatus: true, script: "command -v sshpass >/dev/null 2>&1")
                    if (status != 0) {
                        error """
❌ ERROR: Jenkins agent वर 'sshpass' install नाही.

Ubuntu/Debian:
  sudo apt update && sudo apt install -y sshpass

RHEL/CentOS:
  sudo yum install -y epel-release
  sudo yum install -y sshpass
                        """
                    }
                }
            }
        }

        /* ---------------------------------------------------
           2) SSH Connection test
        --------------------------------------------------- */
        stage("Check SSH Connection") {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "echo 'SSH connected ✔'"
                """
            }
        }

        /* ---------------------------------------------------
           3) OS Detect (Linux / Windows)
        --------------------------------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 OS detect करत आहे..."

                    def linuxCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "uname 2>/dev/null" || true
                        """
                    ).trim().toLowerCase()

                    if (linuxCheck.contains("linux")) {
                        env.OS_TYPE = "linux"
                        echo "🟢 Linux OS detected"
                        return
                    }

                    def winCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \\
                            "powershell -Command \\"(Get-CimInstance Win32_OperatingSystem).Caption\\"" || ""
                        """
                    ).trim().toLowerCase()

                    if (winCheck.contains("windows")) {
                        env.OS_TYPE = "windows"
                        echo "🟦 Windows OS detected: ${winCheck}"
                        return
                    }

                    error "❌ OS detect करता आला नाही! Linux='${linuxCheck}', Windows='${winCheck}'"
                }
            }
        }

        /* ---------------------------------------------------
           4) Ensure SSH service + firewall (Linux)
        --------------------------------------------------- */
        stage("Ensure SSH & Firewall (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        set -e
                        echo "🔐 SSH service check (Linux)..."

                        if ! command -v sshd >/dev/null 2>&1; then
                            echo "OpenSSH server install करत आहे..."
                            if command -v apt-get >/dev/null 2>&1; then
                                apt-get update -y
                                apt-get install -y openssh-server
                            elif command -v yum >/dev/null 2>&1; then
                                yum install -y openssh-server
                            fi
                        fi

                        systemctl enable ssh || systemctl enable sshd || true
                        systemctl start ssh || systemctl start sshd || true

                        if command -v ufw >/dev/null 2>&1; then
                            ufw allow ssh || true
                        elif command -v firewall-cmd >/dev/null 2>&1; then
                            firewall-cmd --add-service=ssh --permanent || true
                            firewall-cmd --reload || true
                        fi

                        echo "SSH service & firewall ready (Linux)."
                    '
                """
            }
        }

        /* ---------------------------------------------------
           5) Ensure SSH service + firewall (Windows)
        --------------------------------------------------- */
        stage("Ensure SSH & Firewall (Windows)") {
            when { expression { env.OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \\
                    "powershell -Command \\
                    \\"Write-Host '🔐 Checking SSH service (Windows)...'; \\
                    \$svc = Get-Service -Name 'sshd' -ErrorAction SilentlyContinue; \\
                    if (\$svc) { \\
                        Set-Service -Name 'sshd' -StartupType Automatic; \\
                        if (\$svc.Status -ne 'Running') { Start-Service 'sshd'; } \\
                        if (-not (Get-NetFirewallRule -DisplayName 'OpenSSH-Server-In-TCP' -ErrorAction SilentlyContinue)) { \\
                            New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' -DisplayName 'OpenSSH-Server-In-TCP' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22; \\
                        } \\
                        Write-Host 'SSH service & firewall ready (Windows).'; \\
                    } else { \\
                        Write-Host '⚠ OpenSSH Server install नाही. Windows Features मधून install करा.'; \\
                    }\\""
                """
            }
        }

        /* ---------------------------------------------------
           6) Docker & Docker Compose setup (Linux)
        --------------------------------------------------- */
        stage("Docker & Compose Setup (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        set -e
                        echo "🐳 Docker check (Linux)..."

                        if ! command -v docker >/dev/null 2>&1; then
                            echo "Docker install करत आहे..."
                            if command -v apt-get >/dev/null 2>&1; then
                                apt-get update -y
                                apt-get install -y docker.io
                            elif command -v yum >/dev/null 2>&1; then
                                yum install -y docker
                            elif command -v dnf >/dev/null 2>&1; then
                                dnf install -y docker
                            elif command -v zypper >/dev/null 2>&1; then
                                zypper install -y docker
                            else
                                echo "package manager सापडला नाही, get.docker.com वापरतो..."
                                curl -fsSL https://get.docker.com | sh
                            fi
                            systemctl enable docker || true
                            systemctl start docker || true
                        else
                            echo "✔ Docker आधीच install आहे."
                        fi

                        echo "Docker version:"
                        docker --version || true

                        echo "Docker Compose check..."
                        if ! command -v docker-compose >/dev/null 2>&1; then
                            echo "Docker Compose install करत आहे..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        else
                            echo "✔ Docker Compose आधीच install आहे."
                        fi

                        echo "Docker Compose version:"
                        docker-compose --version || true
                    '
                """
            }
        }

        /* ---------------------------------------------------
           7) Docker & Docker Compose setup (Windows)
        --------------------------------------------------- */
        stage("Docker & Compose Setup (Windows)") {
            when { expression { env.OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \\
                    "powershell -Command \\
                    \\"Write-Host '🐳 Docker check (Windows)...'; \\
                    if (docker --version) { \\
                        Write-Host '✔ Docker installed'; docker --version; \\
                    } else { \\
                        Write-Host '❌ Docker install नाही. Docker Desktop manually/winget ने install करा.'; \\
                    } \\
                    Write-Host 'Docker Compose check...'; \\
                    if (docker compose version) { \\
                        Write-Host '✔ docker compose available'; docker compose version; \\
                    } elseif (Get-Command docker-compose -ErrorAction SilentlyContinue) { \\
                        Write-Host '✔ docker-compose available'; docker-compose --version; \\
                    } else { \\
                        Write-Host '❌ docker compose/docker-compose नाही. Docker Desktop settings मधून enable करा.'; \\
                    }\\""
                """
            }
        }

        /* ---------------------------------------------------
           8) Upload docker-compose.yml
        --------------------------------------------------- */
        stage("Upload docker-compose.yml") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "mkdir -p ${LINUX_DIR}"

                            sshpass -p "${SSH_PASS}" \\
                            scp -o StrictHostKeyChecking=no docker-compose.yml \\
                                ${SSH_USER}@${TARGET_IP}:${LINUX_COMPOSE}
                        """
                    } else if (env.OS_TYPE == "windows") {
                        sh """
                            sshpass -p "${SSH_PASS}" \\
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \\
                            "powershell -Command \\"New-Item -ItemType Directory -Force -Path '${WIN_DIR}' | Out-Null\\""

                            sshpass -p "${SSH_PASS}" \\
                            scp -o StrictHostKeyChecking=no docker-compose.yml \\
                                ${SSH_USER}@${TARGET_IP}:${WIN_COMPOSE}
                        """
                    }
                }
            }
        }

        /* ---------------------------------------------------
           9) Postgres setup (Linux via docker-compose)
        --------------------------------------------------- */
        stage("Postgres Setup (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        echo "🐘 Postgres check (Linux via Docker)..."

                        if docker ps | grep -iq "postgres"; then
                            echo "✔ Postgres container already running."
                        else
                            echo "Postgres container नाही, docker-compose ने start करतो..."
                            cd ${LINUX_DIR}
                            docker-compose up -d
                        fi
                    '
                """
            }
        }

        /* ---------------------------------------------------
           10) Postgres setup (Windows via docker-compose)
        --------------------------------------------------- */
        stage("Postgres Setup (Windows)") {
            when { expression { env.OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" \\
                    ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \\
                    "powershell -Command \\
                    \\"Write-Host '🐘 Postgres check (Windows via Docker)...'; \\
                    if (!(docker ps | Select-String -Pattern 'postgres' -SimpleMatch -Quiet)) { \\
                        Write-Host 'Postgres container नाही, docker compose ने start करतो...'; \\
                        if (docker compose version) { \\
                            docker compose -f ${WIN_COMPOSE} up -d; \\
                        } elseif (Get-Command docker-compose -ErrorAction SilentlyContinue) { \\
                            docker-compose -f ${WIN_COMPOSE} up -d; \\
                        } else { \\
                            Write-Host '❌ docker compose/docker-compose सापडला नाही. Postgres start करू शकत नाही.'; \\
                        } \\
                    } else { \\
                        Write-Host '✔ Postgres container already running.'; \\
                    }\\""
                """
            }
        }

    }

    post {
        success {
            echo "🎉 Infrastructure setup यशस्वी ${TARGET_IP} साठी!"
        }
        failure {
            echo "❌ Infrastructure setup fail झाला. वरचे logs तपासा."
        }
    }
}
