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
                        error "❌ sshpass missing!"
                    }
                }
            }
        }

        stage("Detect OS") {
            steps {
                script {
                    def IP = TARGET_IP.trim()
                    echo "Detecting OS on ${IP}..."

                    def linuxCheck = sh(returnStatus:true, script: """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} uname >/dev/null 2>&1
                    """) == 0

                    if (linuxCheck) {
                        env.OS_TYPE = "linux"
                        echo "🟢 Linux detected"
                        return
                    }

                    def winCheck = sh(returnStatus:true, script: """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} powershell -Command "(Get-CimInstance Win32_OperatingSystem).Caption" >/dev/null 2>&1
                    """) == 0

                    if (winCheck) {
                        env.OS_TYPE = "windows"
                        echo "🟦 Windows detected"
                        return
                    }

                    error "❌ Unknown OS"
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
                    def dir  = (env.OS_TYPE == "linux") ? LINUX_DIR     : WIN_DIR
                    def path = (env.OS_TYPE == "linux") ? LINUX_COMPOSE : WIN_COMPOSE

                    sh """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} "mkdir -p ${dir}"
                        sshpass -p '${SSH_PASS}' scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${IP}:${path}
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
   LINUX SETUP
-------------------------- */
def configureLinux() {
    def IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} "
            if ! command -v docker >/dev/null; then
                apt-get update -y;
                apt-get install -y docker.io;
            fi;

            if ! command -v docker-compose >/dev/null; then
                curl -L https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m) -o /usr/local/bin/docker-compose;
                chmod +x /usr/local/bin/docker-compose;
            fi;

            echo 'Linux Ready ✔';
        "
    """
}

/* --------------------------
   WINDOWS SETUP
-------------------------- */
def configureWindows() {
    def IP = TARGET_IP.trim()

    sh """
        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${IP} powershell -Command "
            Write-Host 'Configuring Windows...';

            if (Get-Service sshd -ErrorAction SilentlyContinue) {
                Set-Service sshd -StartupType Automatic;
                Start-Service sshd;
            }

            if (!(docker --version)) {
                winget install -e --id Docker.DockerDesktop --accept-package-agreements --accept-source-agreements;
            }

            Write-Host 'Windows Ready ✔';
        "
    """
}
