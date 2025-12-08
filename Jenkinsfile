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
                withEnv(["SSH_PASS_SECRET=${params.SSH_PASS}"]) {
                    sh '''
                        sshpass -p "$SSH_PASS_SECRET" \
                        ssh -o StrictHostKeyChecking=no $SSH_USER@$TARGET_IP "echo Connected OK"
                    '''
                }
            }
        }

        /* -------------------------
   1.5) ENSURE SSH SERVICE & FIREWALL ENABLED
------------------------- */
        stage("Ensure SSH Service & Firewall") {
            steps {
                withEnv([
                    "SSH_PASS_SECRET=${params.SSH_PASS}",
                    "SSH_USER=${params.SSH_USER}",
                    "TARGET_IP=${params.TARGET_IP}"
                ]) {
        
                    sh '''
                        sshpass -p "$SSH_PASS_SECRET" ssh -o StrictHostKeyChecking=no $SSH_USER@$TARGET_IP bash -s << 'EOF'
        
                            echo "🔐 Checking SSH service..."
        
                            # --- Detect Linux ---
                            if command -v uname >/dev/null 2>&1; then
                                OS_CHECK=$(uname | tr "[:upper:]" "[:lower:]")
        
                                if [[ "$OS_CHECK" == "linux" ]]; then
                                    echo "🟢 Linux detected — configuring SSH..."
        
                                    # Install SSH server if missing
                                    if ! command -v sshd >/dev/null 2>&1; then
                                        echo "🔧 Installing OpenSSH server..."
                                        apt-get update -y || yum update -y
                                        apt-get install -y openssh-server || yum install -y openssh-server
                                    else
                                        echo "✔ SSH server already installed."
                                    fi
        
                                    # Enable/start SSH
                                    systemctl enable ssh || systemctl enable sshd || true
                                    systemctl start ssh || systemctl start sshd || true
        
                                    # Firewall
                                    if command -v ufw >/dev/null 2>&1; then
                                        echo "🔐 Opening SSH in UFW..."
                                        ufw allow ssh || true
                                    elif command -v firewall-cmd >/dev/null 2>&1; then
                                        echo "🔐 Opening SSH in firewalld..."
                                        firewall-cmd --add-service=ssh --permanent || true
                                        firewall-cmd --reload || true
                                    else
                                        echo "⚠ No firewall tool detected (UFW/firewalld). Skipping firewall."
                                    fi
                                fi
                            fi
        
                            # --- Detect Windows ---
                            if powershell -command "Get-Service sshd" >/dev/null 2>&1; then
                                echo "🟦 Windows detected — configuring SSH..."
        
                                powershell -command "
                                    Set-Service -Name sshd -StartupType Automatic;
                                    Start-Service sshd;
                                    if (!(Get-NetFirewallRule -DisplayName 'OpenSSH-Server-In-TCP')) {
                                        New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH-Server-In-TCP' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22;
                                    }
                                    Write-Host '✔ SSH and firewall configured.';
                                "
                            fi
        
                            echo "SSH configuration complete."
        
                        EOF
                    '''
                }
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
                            sshpass -p "${params.SSH_PASS}" \
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
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
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

        /* -------------------------------------------
           2.5) CHECK DOCKER, COMPOSE, POSTGRES, REDIS (LINUX)
        ------------------------------------------- */
        stage("Check Docker & Services (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '

                        echo "Checking Docker..."
                        if ! command -v docker >/dev/null; then
                            echo "Docker not installed. Installing..."
                            apt-get update -y || yum update -y
                            apt-get install -y docker.io || yum install -y docker
                            systemctl start docker || true
                            systemctl enable docker || true
                        else
                            echo "Docker already installed."
                        fi

                        echo "Checking Docker Compose..."
                        if ! command -v docker-compose >/dev/null; then
                            echo "Docker Compose not installed. Installing..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        else
                            echo "Docker Compose already installed."
                        fi

                        echo "Checking Postgres container..."
                        if ! docker ps --format "{{.Names}}" | grep -qi postgres; then
                            echo "Postgres missing. It will be launched via docker-compose."
                        else
                            echo "Postgres is already running."
                        fi

                        echo "Checking Redis container..."
                        if ! docker ps --format "{{.Names}}" | grep -qi redis; then
                            echo "Redis missing. It will be launched via docker-compose."
                        else
                            echo "Redis is already running."
                        fi
                    '
                """
            }
        }

        /* -------------------------------------------
           2.6) CHECK DOCKER, COMPOSE, POSTGRES, REDIS (WINDOWS)
        ------------------------------------------- */
        stage("Check Docker & Services (Windows)") {
            when { expression { env.OS_TYPE == "windows" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "
                        
                        Write-Host 'Checking Docker...'
                        if (!(docker --version)) {
                            Write-Host '❌ Docker not installed. Please install Docker Desktop manually.';
                        } else {
                            Write-Host '✔ Docker installed.';
                        }

                        Write-Host 'Checking Docker Compose...'
                        if (!(docker compose version)) {
                            Write-Host '❌ Docker Compose not installed. Install Docker Desktop.';
                        } else {
                            Write-Host '✔ Docker Compose installed.';
                        }

                        Write-Host 'Checking Postgres container...'
                        if (-not (docker ps --format '{{.Names}}' | findstr /I 'postgres')) {
                            Write-Host 'Postgres missing. It will be started via docker-compose if Docker exists.';
                        } else {
                            Write-Host 'Postgres is already running.';
                        }

                        Write-Host 'Checking Redis container...'
                        if (-not (docker ps --format '{{.Names}}' | findstr /I 'redis')) {
                            Write-Host 'Redis missing. It will be started via docker-compose if Docker exists.';
                        } else {
                            Write-Host 'Redis is already running.';
                        }
                    "
                """
            }
        }

        /* -------------------------
           3) INSTALL DOCKER (LINUX)
        ------------------------- */
        stage("Install Docker (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '

                        echo "Re-checking Docker..."
                        if ! command -v docker >/dev/null; then
                            echo "Installing Docker..."
                            apt-get update -y || yum update -y
                            apt-get install -y docker.io || yum install -y docker
                            systemctl start docker || true
                            systemctl enable docker || true
                        fi

                        echo "Re-checking Docker Compose..."
                        if ! command -v docker-compose >/dev/null; then
                            echo "Installing Docker Compose..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        fi
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
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                    "powershell -Command \\"Write-Host 'Windows detected. Install Docker Desktop manually or via winget.'\\""
                """
            }
        }

        /* -------------------------
           5) UPLOAD COMPOSE FILE
        ------------------------- */
        stage("Upload docker-compose.yml") {
            steps {
                script {

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${COMPOSE_DIR_LINUX}"
                            sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_LINUX}
                        """
                    }

                    if (env.OS_TYPE == "windows") {
                        sh """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "powershell -Command \\"New-Item -ItemType Directory -Force -Path '${COMPOSE_DIR_WIN}'\\""
                            sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_WIN}
                        """
                    }
                }
            }
        }

        /* -------------------------
           6) RUN COMPOSE (LINUX ONLY)
        ------------------------- */
        stage("Run docker-compose (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        cd ${COMPOSE_DIR_LINUX}
                        docker-compose down || true
                        docker-compose up -d
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
