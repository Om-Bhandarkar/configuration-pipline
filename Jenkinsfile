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

        stage("Validate compose") {
            steps {
                script {
                    if (!fileExists("docker-compose.yml")) {
                        error "❌ docker-compose.yml missing!"
                    }
                }
            }
        }

        stage("Check sshpass") {
            steps {
                script {
                    if (sh(returnStatus: true, script: "command -v sshpass") != 0) {
                        error "❌ sshpass missing. Install sshpass first."
                    }
                }
            }
        }

        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting OS..."

                    def IP = TARGET_IP.trim()

                    def isLinux = sh(returnStatus: true, script: """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} uname >/dev/null 2>&1
                    """) == 0

                    if (isLinux) {
                        env.OS_TYPE = "linux"
                        echo "🟢 Linux detected"
                        return
                    }

                    def isWindows = sh(returnStatus: true, script: """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} \
                        powershell -Command "(Get-CimInstance Win32_OperatingSystem).Caption" >/dev/null 2>&1
                    """) == 0

                    if (isWindows) {
                        env.OS_TYPE = "windows"
                        echo "🟦 Windows detected"
                        return
                    }

                    error "❌ Unknown OS! SSH may be failing or wrong IP."
                }
            }
        }

        stage("Configure System") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") configureLinux()
                    if (env.OS_TYPE == "windows") configureWindows()
                }
            }
        }

        stage("Upload Compose") {
            steps {
                script {
                    def IP = TARGET_IP.trim()
                    def dir  = env.OS_TYPE == "linux" ? LINUX_DIR : WIN_DIR
                    def path = env.OS_TYPE == "linux" ? LINUX_COMPOSE : WIN_COMPOSE

                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} "mkdir -p '${dir}'"
                        sshpass -p '${SSH_PASS}' scp docker-compose.yml ${SSH_USER}@${IP}:${path}
                    """
                }
            }
        }

        stage("Run Compose") {
            steps {
                script {
                    def IP = TARGET_IP.trim()

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} << 'EOF'
                                cd ${LINUX_DIR}
                                docker-compose up -d
EOF
                        """
                    } else {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} \
                                powershell -Command "docker compose -f '${WIN_COMPOSE}' up -d"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def IP = TARGET_IP.trim()

                if (env.OS_TYPE == "linux") {
                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} << 'EOF'
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

echo "\\n▶ Compose file:"
echo "${LINUX_COMPOSE}"

echo "=============================="
echo "Summary Complete ✔"
echo "=============================="
EOF
                    """
                } else {
                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} \
                        powershell -Command "
                            Write-Host '==============================';
                            Write-Host '  🔍 SYSTEM SUMMARY (WINDOWS)';
                            Write-Host '==============================';

                            Write-Host '\\n▶ Installed Tools:';
                            docker --version
                            docker-compose --version

                            Write-Host '\\n▶ Active Ports:';
                            Get-NetTCPConnection -State Listen | Format-Table -AutoSize

                            Write-Host '\\n▶ Running Containers:';
                            docker ps --format 'table {{.Names}}  {{.Image}}  {{.Ports}}'

                            Write-Host '\\n▶ Compose file:';
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

/* ------------------------------
   LINUX CONFIG
------------------------------ */
def configureLinux() {
    def IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} << 'EOF'
set -e
echo "🔧 Preparing Linux..."

if ! command -v docker >/dev/null; then
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
EOF
    """
}

/* ------------------------------
   WINDOWS CONFIG
------------------------------ */
def configureWindows() {
    def IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} \
        powershell -Command "
            Write-Host '🔧 Preparing Windows...';

            \$svc = Get-Service sshd -ErrorAction SilentlyContinue
            if (\$svc) {
                Set-Service sshd -StartupType Automatic; Start-Service sshd
                if (-not (Get-NetFirewallRule -DisplayName 'OpenSSH')) {
                    New-NetFirewallRule -DisplayName 'OpenSSH' -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow
                }
            }

            if (!(docker --version)) {
                if (Get-Command winget -ErrorAction SilentlyContinue) {
                    winget install -e --id Docker.DockerDesktop -h --accept-package-agreements --accept-source-agreements
                }
            }

            Write-Host 'Windows Ready ✔'
        "
    """
}
