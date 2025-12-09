pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP',  description: 'Target Server IP')
        string(name: 'SSH_USER',   defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        LINUX_DIR     = "/infra"
        LINUX_COMPOSE = "/infra/docker-compose.yml"
        WIN_DIR       = "C:/infra"
        WIN_COMPOSE   = "C:/infra/docker-compose.yml"
    }

    stages {

        /* ------------------------------ 
           0) Validate compose file
        ------------------------------ */
        stage("Validate compose") {
            steps {
                script {
                    if (!fileExists("docker-compose.yml")) {
                        error "❌ docker-compose.yml missing!"
                    }
                }
            }
        }

        /* ------------------------------
           1) Ensure sshpass installed
        ------------------------------ */
        stage("Check sshpass") {
            steps {
                script {
                    if (sh(returnStatus: true, script: "command -v sshpass") != 0) {
                        error "❌ sshpass missing. Install sshpass first."
                    }
                }
            }
        }

        /* ------------------------------
           2) Detect Operating System
        ------------------------------ */
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting OS..."

                    def CLEAN_IP = TARGET_IP.trim()

                    // Linux check
                    def isLinux = sh(
                        returnStatus: true,
                        script: """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} uname >/dev/null 2>&1
                        """
                    ) == 0

                    if (isLinux) {
                        env.OS_TYPE = "linux"
                        echo "🟢 Linux detected"
                        return
                    }

                    // Windows check
                    def isWindows = sh(
                        returnStatus: true,
                        script: """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} powershell -Command "(Get-CimInstance Win32_OperatingSystem).Caption" >/dev/null 2>&1
                        """
                    ) == 0

                    if (isWindows) {
                        env.OS_TYPE = "windows"
                        echo "🟦 Windows detected"
                        return
                    }

                    error "❌ Unknown OS! SSH may be failing or wrong IP."
                }
            }
        }

        /* ------------------------------
           3) Configure OS Requirements
        ------------------------------ */
        stage("Configure System") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") configureLinux()
                    if (env.OS_TYPE == "windows") configureWindows()
                }
            }
        }

        /* ------------------------------
           4) Upload Compose File
        ------------------------------ */
        stage("Upload Compose") {
            steps {
                script {
                    def CLEAN_IP = TARGET_IP.trim()
                    def dir  = (env.OS_TYPE == "linux") ? LINUX_DIR     : WIN_DIR
                    def path = (env.OS_TYPE == "linux") ? LINUX_COMPOSE : WIN_COMPOSE

                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} "mkdir -p '${dir}'"
                        sshpass -p '${SSH_PASS}' scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${CLEAN_IP}:${path}
                    """
                }
            }
        }

        /* ------------------------------
           5) Run Docker Compose
        ------------------------------ */
        stage("Run Compose") {
            steps {
                script {
                    def CLEAN_IP = TARGET_IP.trim()

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} "
                                cd ${LINUX_DIR} && docker-compose up -d
                            "
                        """
                    } else {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} \
                                powershell -Command "docker compose -f '${WIN_COMPOSE}' up -d"
                        """
                    }
                }
            }
        }
    }

    /* ------------------------------
       POST DEPLOYMENT SUMMARY
    ------------------------------ */
    post {
        success {
            echo "🎉 Deployment Complete!"
            echo "📦 Gathering System Summary..."

            script {
                def CLEAN_IP = TARGET_IP.trim()

                if (env.OS_TYPE == "linux") {
                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} '
                            echo "=============================="
                            echo "  🔍 SYSTEM SUMMARY (LINUX)"
                            echo "=============================="

                            echo "▶ Installed Tools:"
                            docker --version || true
                            docker-compose --version || true

                            echo "\\n▶ Active Ports:"
                            ss -tulnp 2>/dev/null || netstat -tulnp || true

                            echo "\\n▶ Running Containers:"
                            docker ps --format "table {{.Names}}\\t{{.Image}}\\t{{.Ports}}"

                            echo "\\n▶ docker-compose.yml Location:"
                            echo "${LINUX_COMPOSE}"

                            echo "=============================="
                            echo "Summary Complete ✔"
                            echo "=============================="
                        '
                    """
                } else {
                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} \
                            powershell -Command "
                                Write-Host '==============================';
                                Write-Host '  🔍 SYSTEM SUMMARY (WINDOWS)';
                                Write-Host '==============================';

                                Write-Host '\\n▶ Installed Tools:';
                                if (Get-Command docker -ErrorAction SilentlyContinue) { docker --version }
                                if (Get-Command docker-compose -ErrorAction SilentlyContinue) { docker-compose --version }
                                if (Get-Service sshd -ErrorAction SilentlyContinue) { Write-Host 'OpenSSH: Installed ✔' }

                                Write-Host '\\n▶ Active Listening Ports:';
                                Get-NetTCPConnection -State Listen |
                                    Select-Object LocalAddress,LocalPort,OwningProcess |
                                    Sort-Object LocalPort | Format-Table -AutoSize

                                Write-Host '\\n▶ Running Containers:';
                                docker ps --format 'table {{.Names}}    {{.Image}}    {{.Ports}}'

                                Write-Host '\\n▶ docker-compose.yml Location:';
                                Write-Host '${WIN_COMPOSE}'

                                Write-Host '==============================';
                                Write-Host 'Summary Complete ✔';
                                Write-Host '==============================';
                            "
                    """
                }
            }
        }

        failure {
            echo "❌ Deployment failed."
        }
    }
}

/* =======================================================
   🔧 LINUX CONFIG
======================================================== */
def configureLinux() {
    def CLEAN_IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} '
            set -e
            echo "🔧 Preparing Linux System..."

            if ! command -v docker >/dev/null; then
                echo "Installing Docker..."
                apt-get update -y || true
                apt-get install -y docker.io || yum install -y docker || true
                systemctl enable docker
                systemctl start docker
            fi

            if ! command -v docker-compose >/dev/null; then
                curl -L https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\$(uname -s)-\\$(uname -m) \
                    -o /usr/local/bin/docker-compose
                chmod +x /usr/local/bin/docker-compose
            fi

            echo "Linux Ready ✔"
        '
    """
}

/* =======================================================
   🔧 WINDOWS CONFIG
======================================================== */
def configureWindows() {
    def CLEAN_IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${CLEAN_IP} \
            powershell -Command "
                Write-Host '🔧 Preparing Windows System...';

                \$svc = Get-Service sshd -ErrorAction SilentlyContinue
                if (\$svc) {
                    Set-Service sshd -StartupType Automatic; Start-Service sshd
                    if (-not (Get-NetFirewallRule -DisplayName 'OpenSSH')) {
                        New-NetFirewallRule -DisplayName 'OpenSSH' -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow
                    }
                } else {
                    Write-Host '❌ OpenSSH not installed.'
                }

                if (!(docker --version)) {
                    if (Get-Command winget -ErrorAction SilentlyContinue) {
                        winget install -e --id Docker.DockerDesktop -h --accept-package-agreements --accept-source-agreements
                    }
                    else { Write-Host '❌ winget missing.' }
                }

                Write-Host 'Windows Ready ✔'
            "
    """
}
