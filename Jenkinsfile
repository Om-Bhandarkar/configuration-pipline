pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Target Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE = ""
        COMPOSE_DIR_LINUX = "/infra"
        COMPOSE_FILE_LINUX = "/infra/docker-compose.yml"
        COMPOSE_DIR_WIN = "C:/infra"
        COMPOSE_FILE_WIN = "C:/infra/docker-compose.yml"
    }

    stages {

        /* -------------------------
           1) CHECK CONNECTION
        ------------------------- */
        stage("Check Connection") {
            steps {
                withEnv(["SSH_PASS_SECRET=${params.SSH_PASS}"]) {
                    sh '''
                        sshpass -p "$SSH_PASS_SECRET" \
                        ssh -o StrictHostKeyChecking=no -q $SSH_USER@$TARGET_IP "echo Connected OK"
                    '''
                }
            }
        }

        /* -------------------------
           1.5) ENSURE SSH SERVICE
        ------------------------- */
        stage("Ensure SSH Service & Firewall") {
            steps {
                withEnv([
                    "SSH_PASS_SECRET=${params.SSH_PASS}",
                    "SSH_USER=${params.SSH_USER}",
                    "TARGET_IP=${params.TARGET_IP}"
                ]) {

                    script {
                        def os = sh(
                            returnStdout: true,
                            script: '''
                                sshpass -p "$SSH_PASS_SECRET" ssh -q -o StrictHostKeyChecking=no $SSH_USER@$TARGET_IP "uname 2>/dev/null" || true
                            '''
                        ).trim().toLowerCase()

                        if (os.contains("linux")) {

                            sh '''
                                sshpass -p "$SSH_PASS_SECRET" ssh -q -o StrictHostKeyChecking=no $SSH_USER@$TARGET_IP 'bash -s' << "EOF"
                                    echo "Ensuring SSH is running..."

                                    (apt-get update -y >/dev/null 2>&1 || yum update -y >/dev/null 2>&1) || true
                                    (apt-get install -y openssh-server >/dev/null 2>&1 || yum install -y openssh-server >/dev/null 2>&1) || true

                                    (systemctl enable ssh >/dev/null 2>&1 || systemctl enable sshd >/dev/null 2>&1) || true
                                    (systemctl start ssh >/dev/null 2>&1 || systemctl start sshd >/dev/null 2>&1) || true

                                    echo "SSH Ready."
                                EOF
                            '''

                        } else {

                            sh '''
                                sshpass -p "$SSH_PASS_SECRET" ssh -q -o StrictHostKeyChecking=no $SSH_USER@$TARGET_IP powershell -Command "
                                    Write-Host 'Ensuring SSH service...';
                                    Set-Service -Name sshd -StartupType Automatic -ErrorAction SilentlyContinue;
                                    Start-Service sshd -ErrorAction SilentlyContinue;
                                    Write-Host 'SSH Ready.';
                                "
                            '''
                        }
                    }
                }
            }
        }

        /* -------------------------
           2) DETECT OS
        ------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    def linuxCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -q -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "uname" || true
                        """
                    ).trim()

                    if (linuxCheck.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "linux"
                        echo "OS: Linux"
                        return
                    }

                    def winCheck = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -q -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "powershell -command \\\"[System.Environment]::OSVersion.Platform\\\"" || true
                        """
                    ).trim()

                    if (winCheck.toLowerCase().contains("win32nt")) {
                        env.OS_TYPE = "windows"
                        echo "OS: Windows"
                        return
                    }

                    error "Could not detect OS!"
                }
            }
        }

        /* -------------------------------------------
           2.5) CHECK DOCKER (LINUX) — QUIET MODE
        ------------------------------------------- */
        stage("Check Docker (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" ssh -q -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        echo "Checking Docker..."
                        if ! command -v docker >/dev/null; then
                            echo "Installing Docker..."
                            (apt-get install -y docker.io >/dev/null 2>&1 || yum install -y docker >/dev/null 2>&1) || true
                            systemctl start docker >/dev/null 2>&1
                        else
                            echo "Docker OK"
                        fi

                        echo "Checking Compose..."
                        if ! command -v docker-compose >/dev/null; then
                            echo "Installing docker-compose..."
                            curl -sL "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        else
                            echo "Compose OK"
                        fi
                    '
                """
            }
        }

        /* -------------------------
           5) UPLOAD COMPOSE FILE
        ------------------------- */
        stage("Upload docker-compose.yml") {
            steps {
                script {
                    def destDir = env.OS_TYPE == "linux" ? COMPOSE_DIR_LINUX : COMPOSE_DIR_WIN
                    def destFile = env.OS_TYPE == "linux" ? COMPOSE_FILE_LINUX : COMPOSE_FILE_WIN

                    sh """
                        sshpass -p "${params.SSH_PASS}" ssh -q -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${destDir}"
                        sshpass -p "${params.SSH_PASS}" scp -q -o StrictHostKeyChecking=no docker-compose.yml ${params.SSH_USER}@${params.TARGET_IP}:${destFile}
                    """
                }
            }
        }

        /* -------------------------
           6) RUN COMPOSE
        ------------------------- */
        stage("Run docker-compose (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" ssh -q -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        cd ${COMPOSE_DIR_LINUX}
                        docker-compose down >/dev/null 2>&1 || true
                        docker-compose up -d --quiet-pull
                        echo "Compose Up Completed."
                    '
                """
            }
        }
    }

    post {
        success { echo "🎉 Deployment Successful" }
        failure { echo "❌ Deployment Failed" }
    }
}
