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

        /* -------------------------
           VALIDATE docker-compose.yml
        ------------------------- */
        stage("Validate compose") {
            steps {
                script {
                    if (!fileExists("docker-compose.yml")) {
                        error "❌ docker-compose.yml missing!"
                    }
                }
            }
        }

        /* -------------------------
           CHECK sshpass
        ------------------------- */
        stage("Check sshpass") {
            steps {
                script {
                    if (sh(returnStatus: true, script: "command -v sshpass") != 0) {
                        error "❌ sshpass missing on Jenkins agent!"
                    }
                }
            }
        }

        /* -------------------------
           DETECT OS
        ------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    def IP = TARGET_IP.trim()

                    echo "🔍 Detecting OS on ${IP}..."

                    // Linux check
                    def linuxCheck = sh(returnStatus:true, script: """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} uname >/dev/null 2>&1
                    """) == 0

                    if (linuxCheck) {
                        env.OS_TYPE = "linux"
                        echo "🟢 OS Detected: Linux"
                        return
                    }

                    // Windows check
                    def winCheck = sh(returnStatus:true, script: """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} powershell -Command "[System.Environment]::OSVersion.Platform" >/dev/null 2>&1
                    """) == 0

                    if (winCheck) {
                        env.OS_TYPE = "windows"
                        echo "🟦 OS Detected: Windows"
                        return
                    }

                    error "❌ Unknown OS! Cannot continue."
                }
            }
        }

        /* -------------------------
           CONFIGURE SYSTEM
        ------------------------- */
        stage("Configure OS") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") configureLinux()
                    if (env.OS_TYPE == "windows") configureWindows()
                }
            }
        }

        /* -------------------------
           UPLOAD COMPOSE FILE
        ------------------------- */
        stage("Upload Compose File") {
            steps {
                script {
                    def IP = TARGET_IP.trim()

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} "mkdir -p ${LINUX_DIR}"
                            sshpass -p '${SSH_PASS}' scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${IP}:${LINUX_COMPOSE}
                        """
                    } else {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} powershell -Command "New-Item -ItemType Directory -Force -Path '${WIN_DIR}'"
                            sshpass -p '${SSH_PASS}' scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${IP}:${WIN_COMPOSE}
                        """
                    }
                }
            }
        }

        /* -------------------------
           RUN DOCKER COMPOSE
        ------------------------- */
        stage("Run Compose") {
            steps {
                script {
                    def IP = TARGET_IP.trim()

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} "cd ${LINUX_DIR} && docker-compose up -d"
                        """
                    } else {
                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} powershell -Command "docker compose -f '${WIN_COMPOSE}' up -d"
                        """
                    }
                }
            }
        }
    }

    /* -------------------------
       POST DEPLOYMENT SUMMARY
    ------------------------- */
    post {
        success {
            script {
                def IP = TARGET_IP.trim()

                if (env.OS_TYPE == "linux") {
                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} "
                            echo '===== SYSTEM SUMMARY (LINUX) =====';
                            docker --version;
                            docker-compose --version;
                            ss -tulnp || netstat -tulnp;
                            docker ps;
                            echo 'Compose File: ${LINUX_COMPOSE}';
                            echo '==================================';
                        "
                    """
                } else {
                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} powershell -Command "
                            Write-Host '===== SYSTEM SUMMARY (WINDOWS) =====';
                            docker --version;
                            docker-compose --version;
                            Get-NetTCPConnection -State Listen | Format-Table -AutoSize;
                            docker ps;
                            Write-Host 'Compose File: ${WIN_COMPOSE}';
                            Write-Host '====================================';
                        "
                    """
                }
            }
        }

        failure {
            echo "❌ Deployment Failed"
        }
    }
}

/* --------------------------
   LINUX CONFIG
-------------------------- */
def configureLinux() {
    def IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} "
            if ! command -v docker >/dev/null; then
                apt-get update -y || true;
                apt-get install -y docker.io || yum install -y docker;
                systemctl start docker || true;
                systemctl enable docker || true;
            fi;

            if ! command -v docker-compose >/dev/null; then
                curl -L https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m) \
                    -o /usr/local/bin/docker-compose;
                chmod +x /usr/local/bin/docker-compose;
            fi;

            echo 'Linux Ready ✔';
        "
    """
}

/* --------------------------
   WINDOWS CONFIG
-------------------------- */
def configureWindows() {
    def IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} powershell -Command "
            Write-Host 'Configuring Windows...';

            \$svc = Get-Service sshd -ErrorAction SilentlyContinue
            if (\$svc) {
                Set-Service sshd -StartupType Automatic;
                Start-Service sshd;
            }

            if (-not (Get-NetFirewallRule -DisplayName 'OpenSSH' -ErrorAction SilentlyContinue)) {
                New-NetFirewallRule -DisplayName 'OpenSSH' -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow
            }

            if (-not (docker --version)) {
                if (Get-Command winget -ErrorAction SilentlyContinue) {
                    winget install -e --id Docker.DockerDesktop --accept-package-agreements --accept-source-agreements
                } else {
                    Write-Host 'winget not available!';
                }
            }

            Write-Host 'Windows Ready ✔';
        "
    """
}
