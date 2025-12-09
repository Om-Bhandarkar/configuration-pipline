pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Target Server IP', defaultValue: '')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE = ""
        COMPOSE_DIR_LINUX  = "/infra"
        COMPOSE_FILE_LINUX = "/infra/docker-compose.yml"
        COMPOSE_DIR_WIN    = "C:/infra"
        COMPOSE_FILE_WIN   = "C:/infra/docker-compose.yml"
        DEBUG_LOG = "/tmp/jenkins_debug.log"
    }

    stages {

        /* -------------------------
           0) VALIDATE PARAMETERS
        ------------------------- */
        stage("Validate Parameters") {
            steps {
                script {
                    echo "🔍 DEBUG: Starting parameter validation..."
                    echo "🔍 DEBUG: TARGET_IP = ${params.TARGET_IP}"
                    echo "🔍 DEBUG: SSH_USER = ${params.SSH_USER}"
                    echo "🔍 DEBUG: SSH_PASS is set = ${params.SSH_PASS != null}"
                    
                    if (!params.TARGET_IP?.trim()) {
                        error "❌ TARGET_IP parameter is required!"
                    }
                    
                    if (!params.SSH_PASS?.trim()) {
                        error "❌ SSH_PASSWORD parameter is required!"
                    }
                    
                    echo "✅ DEBUG: All parameters validated successfully"
                }
            }
        }

        /* -------------------------
           1) CHECK SSH CONNECTION
        ------------------------- */
        stage("Check SSH Connection") {
            steps {
                script {
                    echo "🔍 DEBUG: Testing SSH connection to ${params.TARGET_IP}..."
                    
                    def maxRetries = 3
                    def retryCount = 0
                    def connected = false
                    
                    while (retryCount < maxRetries && !connected) {
                        try {
                            echo "🔍 DEBUG: SSH attempt ${retryCount + 1}/${maxRetries}"
                            
                            def result = sh(
                                returnStdout: true,
                                script: """
                                    set +x  # Disable command echoing for password security
                                    timeout 30 sshpass -p "${params.SSH_PASS}" \
                                    ssh -o ConnectTimeout=10 \
                                        -o StrictHostKeyChecking=no \
                                        -o BatchMode=no \
                                        -o PasswordAuthentication=yes \
                                        ${params.SSH_USER}@${params.TARGET_IP} \
                                        "echo 'SSH Connection Successful' && hostname && whoami"
                                """,
                                returnStatus: true
                            )
                            
                            if (result == 0) {
                                connected = true
                                echo "✅ DEBUG: SSH connection successful"
                            } else {
                                echo "⚠️ DEBUG: SSH attempt failed with exit code: ${result}"
                                retryCount++
                                if (retryCount < maxRetries) {
                                    sleep(time: 5, unit: 'SECONDS')
                                }
                            }
                        } catch (Exception e) {
                            echo "⚠️ DEBUG: Exception during SSH: ${e.message}"
                            retryCount++
                            if (retryCount < maxRetries) {
                                sleep(time: 5, unit: 'SECONDS')
                            }
                        }
                    }
                    
                    if (!connected) {
                        error "❌ Failed to establish SSH connection after ${maxRetries} attempts"
                    }
                }
            }
        }

        /* -------------------------
           2) DETECT OS TYPE
        ------------------------- */
        stage("Detect OS Type") {
            steps {
                script {
                    echo "🔍 DEBUG: Starting OS detection..."
                    
                    try {
                        // First try Linux detection
                        echo "🔍 DEBUG: Attempting Linux detection..."
                        def linuxResult = sh(
                            returnStdout: true,
                            script: """
                                set +x
                                timeout 15 sshpass -p "${params.SSH_PASS}" \
                                ssh -o ConnectTimeout=10 \
                                    -o StrictHostKeyChecking=no \
                                    ${params.SSH_USER}@${params.TARGET_IP} \
                                    "uname -s 2>/dev/null || echo 'NOT_LINUX'"
                            """,
                            returnStatus: true
                        )
                        
                        def linuxOutput = sh(
                            returnStdout: true,
                            script: """
                                set +x
                                sshpass -p "${params.SSH_PASS}" \
                                ssh -o ConnectTimeout=10 \
                                    -o StrictHostKeyChecking=no \
                                    ${params.SSH_USER}@${params.TARGET_IP} \
                                    "uname -s 2>/dev/null || echo 'NOT_LINUX'"
                            """
                        ).trim().toLowerCase()
                        
                        echo "🔍 DEBUG: Linux detection output: '${linuxOutput}'"
                        
                        if (linuxOutput.contains("linux")) {
                            env.OS_TYPE = "linux"
                            echo "✅ DEBUG: Linux OS detected: ${linuxOutput}"
                            
                            // Get additional Linux info
                            def linuxDetails = sh(
                                returnStdout: true,
                                script: """
                                    set +x
                                    sshpass -p "${params.SSH_PASS}" \
                                    ssh -o StrictHostKeyChecking=no \
                                    ${params.SSH_USER}@${params.TARGET_IP} \
                                    "cat /etc/os-release 2>/dev/null | grep PRETTY_NAME || echo 'Distro info not available'"
                                """
                            ).trim()
                            echo "🔍 DEBUG: Linux distribution: ${linuxDetails}"
                            return
                        }
                        
                        // Try Windows detection
                        echo "🔍 DEBUG: Attempting Windows detection..."
                        def winResult = sh(
                            returnStdout: true,
                            script: """
                                set +x
                                timeout 15 sshpass -p "${params.SSH_PASS}" \
                                ssh -o ConnectTimeout=10 \
                                    -o StrictHostKeyChecking=no \
                                    ${params.SSH_USER}@${params.TARGET_IP} \
                                    "powershell -Command \\\"Write-Output \\\$env:OS\\\" 2>/dev/null || echo 'NOT_WINDOWS'"
                            """,
                            returnStatus: true
                        )
                        
                        def winOutput = sh(
                            returnStdout: true,
                            script: """
                                set +x
                                sshpass -p "${params.SSH_PASS}" \
                                ssh -o StrictHostKeyChecking=no \
                                    ${params.SSH_USER}@${params.TARGET_IP} \
                                    "powershell -Command \\\"Write-Output \\\$env:OS\\\" 2>/dev/null || echo 'NOT_WINDOWS'"
                            """
                        ).trim().toLowerCase()
                        
                        echo "🔍 DEBUG: Windows detection output: '${winOutput}'"
                        
                        if (winOutput.contains("windows")) {
                            env.OS_TYPE = "windows"
                            echo "✅ DEBUG: Windows OS detected"
                            
                            // Get Windows version
                            def winVersion = sh(
                                returnStdout: true,
                                script: """
                                    set +x
                                    sshpass -p "${params.SSH_PASS}" \
                                    ssh -o StrictHostKeyChecking=no \
                                        ${params.SSH_USER}@${params.TARGET_IP} \
                                        "powershell -Command \\\"[System.Environment]::OSVersion.VersionString\\\""
                                """
                            ).trim()
                            echo "🔍 DEBUG: Windows version: ${winVersion}"
                            return
                        }
                        
                        // Fallback: Check for common files
                        echo "🔍 DEBUG: Attempting fallback detection..."
                        def fallbackCheck = sh(
                            returnStdout: true,
                            script: """
                                set +x
                                sshpass -p "${params.SSH_PASS}" \
                                ssh -o StrictHostKeyChecking=no \
                                    ${params.SSH_USER}@${params.TARGET_IP} \
                                    "ls /etc/ 2>/dev/null && echo 'linux' || (dir C:\\ 2>/dev/null && echo 'windows' || echo 'unknown')"
                            """
                        ).trim()
                        
                        echo "🔍 DEBUG: Fallback detection: ${fallbackCheck}"
                        
                        if (fallbackCheck.contains("linux")) {
                            env.OS_TYPE = "linux"
                            echo "✅ DEBUG: Linux detected via fallback"
                        } else if (fallbackCheck.contains("windows")) {
                            env.OS_TYPE = "windows"
                            echo "✅ DEBUG: Windows detected via fallback"
                        } else {
                            error "❌ DEBUG: Unable to detect OS. Linux check: '${linuxOutput}', Windows check: '${winOutput}'"
                        }
                        
                    } catch (Exception e) {
                        echo "⚠️ DEBUG: Exception during OS detection: ${e.message}"
                        error "❌ OS detection failed: ${e.message}"
                    }
                }
            }
        }

        /* -------------------------
           3) INSTALL DOCKER (LINUX ONLY)
        ------------------------- */
        stage("Install Docker (Linux)") {
            when { 
                expression { 
                    return env.OS_TYPE == "linux" 
                }
            }
            steps {
                script {
                    echo "🔍 DEBUG: Starting Docker installation for Linux..."
                    
                    sh """
                        set +x
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                            echo "=== DOCKER INSTALLATION LOG ===" > ${DEBUG_LOG}
                            
                            # Check if Docker is already installed
                            if command -v docker >/dev/null 2>&1; then
                                echo "✅ Docker is already installed: \$(docker --version)" >> ${DEBUG_LOG}
                            else
                                echo "🔍 Installing Docker..." >> ${DEBUG_LOG}
                                
                                # Detect package manager
                                if command -v apt-get >/dev/null 2>&1; then
                                    echo "🔍 Using apt package manager (Debian/Ubuntu)" >> ${DEBUG_LOG}
                                    apt-get update -y >> ${DEBUG_LOG} 2>&1
                                    apt-get install -y docker.io >> ${DEBUG_LOG} 2>&1
                                elif command -v yum >/dev/null 2>&1; then
                                    echo "🔍 Using yum package manager (RHEL/CentOS)" >> ${DEBUG_LOG}
                                    yum update -y >> ${DEBUG_LOG} 2>&1
                                    yum install -y docker >> ${DEBUG_LOG} 2>&1
                                elif command -v dnf >/dev/null 2>&1; then
                                    echo "🔍 Using dnf package manager (Fedora)" >> ${DEBUG_LOG}
                                    dnf update -y >> ${DEBUG_LOG} 2>&1
                                    dnf install -y docker >> ${DEBUG_LOG} 2>&1
                                else
                                    echo "❌ Unsupported package manager" >> ${DEBUG_LOG}
                                    exit 1
                                fi
                                
                                # Start and enable Docker service
                                systemctl enable docker >> ${DEBUG_LOG} 2>&1
                                systemctl start docker >> ${DEBUG_LOG} 2>&1
                                echo "✅ Docker installed successfully" >> ${DEBUG_LOG}
                            fi
                            
                            # Check Docker Compose
                            if command -v docker-compose >/dev/null 2>&1; then
                                echo "✅ Docker Compose is already installed: \$(docker-compose --version)" >> ${DEBUG_LOG}
                            else
                                echo "🔍 Installing Docker Compose..." >> ${DEBUG_LOG}
                                curl -L "https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-\$(uname -s)-\$(uname -m)" \
                                    -o /usr/local/bin/docker-compose >> ${DEBUG_LOG} 2>&1
                                chmod +x /usr/local/bin/docker-compose >> ${DEBUG_LOG} 2>&1
                                
                                # Verify installation
                                if docker-compose --version >> ${DEBUG_LOG} 2>&1; then
                                    echo "✅ Docker Compose installed successfully" >> ${DEBUG_LOG}
                                else
                                    echo "❌ Docker Compose installation failed" >> ${DEBUG_LOG}
                                    exit 1
                                fi
                            fi
                            
                            # Add user to docker group to avoid sudo
                            if ! groups \$(whoami) | grep -q docker; then
                                echo "🔍 Adding user to docker group..." >> ${DEBUG_LOG}
                                usermod -aG docker \$(whoami) >> ${DEBUG_LOG} 2>&1
                            fi
                            
                            echo "=== INSTALLATION COMPLETE ===" >> ${DEBUG_LOG}
                            cat ${DEBUG_LOG}
                        '
                    """
                    
                    echo "✅ DEBUG: Docker installation completed"
                }
            }
        }

        /* -------------------------
           4) CHECK DOCKER ON WINDOWS
        ------------------------- */
        stage("Check Docker (Windows)") {
            when { 
                expression { 
                    return env.OS_TYPE == "windows" 
                }
            }
            steps {
                script {
                    echo "🔍 DEBUG: Checking Docker on Windows..."
                    
                    sh """
                        set +x
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                        "powershell -Command \"
                            Write-Host '=== WINDOWS DOCKER CHECK ===' -ForegroundColor Cyan
                            
                            # Check if Docker Desktop is installed
                            if (Get-Command docker -ErrorAction SilentlyContinue) {
                                Write-Host '✅ Docker is installed:' -ForegroundColor Green
                                docker --version
                            } else {
                                Write-Host '❌ Docker is not installed or not in PATH' -ForegroundColor Red
                                Write-Host 'Please install Docker Desktop manually from:' -ForegroundColor Yellow
                                Write-Host 'https://docs.docker.com/desktop/install/windows-install/' -ForegroundColor Yellow
                                exit 1
                            }
                            
                            # Check Docker Compose
                            if (Get-Command docker-compose -ErrorAction SilentlyContinue) {
                                Write-Host '✅ Docker Compose is installed:' -ForegroundColor Green
                                docker-compose --version
                            } else {
                                Write-Host '⚠️ Docker Compose not found, using Docker built-in compose' -ForegroundColor Yellow
                                Write-Host 'For Docker Desktop, compose is included with docker command' -ForegroundColor Yellow
                            }
                            
                            Write-Host '=== CHECK COMPLETE ===' -ForegroundColor Cyan
                        \"
                    """
                }
            }
        }

        /* -------------------------
           5) CREATE DIRECTORY STRUCTURE
        ------------------------- */
        stage("Create Directory Structure") {
            steps {
                script {
                    echo "🔍 DEBUG: Creating directory structure on target..."
                    
                    if (env.OS_TYPE == "linux") {
                        sh """
                            set +x
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo "🔍 Creating directory: ${COMPOSE_DIR_LINUX}"
                                mkdir -p ${COMPOSE_DIR_LINUX}
                                echo "✅ Directory created with permissions:"
                                ls -ld ${COMPOSE_DIR_LINUX}
                            '
                        """
                    } else if (env.OS_TYPE == "windows") {
                        sh """
                            set +x
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \"
                                Write-Host '🔍 Creating directory: ${COMPOSE_DIR_WIN}' -ForegroundColor Cyan
                                New-Item -ItemType Directory -Force -Path '${COMPOSE_DIR_WIN}' | Out-Null
                                Write-Host '✅ Directory created:' -ForegroundColor Green
                                Get-Item '${COMPOSE_DIR_WIN}' | Select-Object FullName, Attributes
                            \"
                        """
                    }
                }
            }
        }

        /* -------------------------
           6) UPLOAD docker-compose.yml
        ------------------------- */
        stage("Upload docker-compose.yml") {
            steps {
                script {
                    echo "🔍 DEBUG: Uploading docker-compose.yml..."
                    
                    // Check if file exists locally
                    def fileExists = fileExists('docker-compose.yml')
                    echo "🔍 DEBUG: Local docker-compose.yml exists: ${fileExists}"
                    
                    if (!fileExists) {
                        error "❌ Local docker-compose.yml file not found!"
                    }
                    
                    // Display file info
                    sh """
                        echo "🔍 Local file info:"
                        ls -la docker-compose.yml
                        echo "--- First 10 lines of docker-compose.yml ---"
                        head -10 docker-compose.yml
                        echo "--------------------------------------------"
                    """
                    
                    if (env.OS_TYPE == "linux") {
                        sh """
                            set +x
                            echo "🔍 Uploading to Linux: ${params.TARGET_IP}:${COMPOSE_FILE_LINUX}"
                            
                            # Upload file
                            sshpass -p "${params.SSH_PASS}" \
                            scp -o StrictHostKeyChecking=no \
                                -o ConnectTimeout=10 \
                                docker-compose.yml \
                                ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_LINUX}
                            
                            # Verify upload
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo "✅ File uploaded successfully:"
                                ls -la ${COMPOSE_FILE_LINUX}
                                echo "--- File content (first 10 lines) ---"
                                head -10 ${COMPOSE_FILE_LINUX}
                            '
                        """
                    } else if (env.OS_TYPE == "windows") {
                        sh """
                            set +x
                            echo "🔍 Uploading to Windows: ${params.TARGET_IP}:${COMPOSE_FILE_WIN}"
                            
                            # Upload file
                            sshpass -p "${params.SSH_PASS}" \
                            scp -o StrictHostKeyChecking=no \
                                -o ConnectTimeout=10 \
                                docker-compose.yml \
                                ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_WIN}
                            
                            # Verify upload
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \"
                                Write-Host '✅ File uploaded successfully:' -ForegroundColor Green
                                Get-Item '${COMPOSE_FILE_WIN}' | Select-Object FullName, Length, LastWriteTime
                                
                                Write-Host '--- File content (first 10 lines) ---' -ForegroundColor Cyan
                                Get-Content '${COMPOSE_FILE_WIN}' -TotalCount 10
                            \"
                        """
                    }
                    
                    echo "✅ DEBUG: File upload completed"
                }
            }
        }

        /* -------------------------
           7) DEPLOY WITH DOCKER COMPOSE
        ------------------------- */
        stage("Deploy with Docker Compose") {
            steps {
                script {
                    echo "🔍 DEBUG: Starting Docker Compose deployment..."
                    
                    if (env.OS_TYPE == "linux") {
                        sh """
                            set +x
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo "=== DOCKER COMPOSE DEPLOYMENT ==="
                                cd ${COMPOSE_DIR_LINUX}
                                
                                echo "🔍 Current directory: \$(pwd)"
                                echo "🔍 Files in directory:"
                                ls -la
                                
                                echo "🔍 Stopping existing containers..."
                                docker-compose down 2>&1 || echo "No containers to stop"
                                
                                echo "🔍 Pulling latest images..."
                                docker-compose pull 2>&1 || echo "Pull completed or not needed"
                                
                                echo "🔍 Starting containers..."
                                docker-compose up -d
                                
                                echo "🔍 Checking container status..."
                                sleep 5
                                docker-compose ps
                                
                                echo "🔍 Container logs (last 10 lines each):"
                                docker-compose logs --tail=10
                                
                                echo "✅ Deployment completed"
                            '
                        """
                    } else if (env.OS_TYPE == "windows") {
                        sh """
                            set +x
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \"
                                Write-Host '=== WINDOWS DOCKER COMPOSE DEPLOYMENT ===' -ForegroundColor Cyan
                                
                                Set-Location '${COMPOSE_DIR_WIN}'
                                Write-Host \"🔍 Current directory: \$PWD\" -ForegroundColor Cyan
                                Write-Host \"🔍 Files in directory:\" -ForegroundColor Cyan
                                Get-ChildItem
                                
                                Write-Host '🔍 Stopping existing containers...' -ForegroundColor Cyan
                                docker-compose down 2>&1 | Write-Host
                                
                                Write-Host '🔍 Pulling latest images...' -ForegroundColor Cyan
                                docker-compose pull 2>&1 | Write-Host
                                
                                Write-Host '🔍 Starting containers...' -ForegroundColor Cyan
                                docker-compose up -d
                                
                                Write-Host '🔍 Checking container status...' -ForegroundColor Cyan
                                Start-Sleep -Seconds 5
                                docker-compose ps
                                
                                Write-Host '✅ Deployment completed' -ForegroundColor Green
                            \"
                        """
                    }
                }
            }
        }

        /* -------------------------
           8) VERIFICATION AND HEALTH CHECK
        ------------------------- */
        stage("Verification and Health Check") {
            steps {
                script {
                    echo "🔍 DEBUG: Starting verification checks..."
                    
                    if (env.OS_TYPE == "linux") {
                        sh """
                            set +x
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo "=== FINAL VERIFICATION ==="
                                echo "🔍 Docker version:"
                                docker --version
                                
                                echo "🔍 Docker Compose version:"
                                docker-compose --version
                                
                                echo "🔍 Docker service status:"
                                systemctl is-active docker || echo "Docker service check skipped"
                                
                                echo "🔍 Running containers:"
                                docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                                
                                echo "🔍 Compose file location:"
                                ls -la ${COMPOSE_FILE_LINUX}
                                
                                echo "🔍 Checking container health (last 20 logs):"
                                cd ${COMPOSE_DIR_LINUX}
                                docker-compose logs --tail=20 2>/dev/null || echo "No logs available"
                                
                                echo "✅ Verification completed"
                            '
                        """
                    } else if (env.OS_TYPE == "windows") {
                        sh """
                            set +x
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \"
                                Write-Host '=== FINAL VERIFICATION (WINDOWS) ===' -ForegroundColor Cyan
                                
                                Write-Host '🔍 Docker version:' -ForegroundColor Cyan
                                docker --version
                                
                                Write-Host '🔍 Docker Compose version:' -ForegroundColor Cyan
                                docker-compose --version 2>&1 | Write-Host
                                
                                Write-Host '🔍 Running containers:' -ForegroundColor Cyan
                                docker ps --format 'table {{.Names}}\\t{{.Status}}\\t{{.Ports}}'
                                
                                Write-Host '🔍 Compose file location:' -ForegroundColor Cyan
                                Get-Item '${COMPOSE_FILE_WIN}' -ErrorAction SilentlyContinue | Select-Object FullName, Length, LastWriteTime
                                
                                Write-Host '🔍 Checking container health:' -ForegroundColor Cyan
                                Set-Location '${COMPOSE_DIR_WIN}'
                                docker-compose logs --tail=20 2>&1 | Write-Host
                                
                                Write-Host '✅ Verification completed' -ForegroundColor Green
                            \"
                        """
                    }
                    
                    echo "✅ DEBUG: All verification checks completed"
                }
            }
        }
    }

    post {
        always {
            echo "📊 Pipeline execution completed with status: ${currentBuild.currentResult}"
            
            script {
                // Clean up sensitive data from logs
                if (params.SSH_PASS) {
                    echo "🔒 Cleaning up sensitive data from logs..."
                }
            }
        }
        
        success {
            echo "🎉 DEPLOYMENT SUCCESSFUL!"
            echo "📋 Summary:"
            echo "  Target Server: ${params.TARGET_IP}"
            echo "  OS Type: ${env.OS_TYPE}"
            echo "  SSH User: ${params.SSH_USER}"
            echo "  Compose File: ${env.OS_TYPE == 'linux' ? env.COMPOSE_FILE_LINUX : env.COMPOSE_FILE_WIN}"
            
            // Send notification (example)
            // emailext (
            //     subject: "✅ Deployment Successful: ${env.JOB_NAME}",
            //     body: "Deployment completed successfully on ${params.TARGET_IP}",
            //     to: "team@example.com"
            // )
        }
        
        failure {
            echo "❌ DEPLOYMENT FAILED!"
            echo "🔍 Please check the following:"
            echo "  1. SSH credentials and network connectivity"
            echo "  2. Target server OS compatibility"
            echo "  3. Docker installation requirements"
            echo "  4. docker-compose.yml file validity"
            echo "  5. Jenkins agent permissions"
            
            script {
                // Archive debug logs
                if (env.OS_TYPE == "linux") {
                    try {
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "cat ${DEBUG_LOG} 2>/dev/null || echo 'Debug log not found'" > debug_output.txt
                        """
                        archiveArtifacts artifacts: 'debug_output.txt', fingerprint: true
                    } catch (Exception e) {
                        echo "⚠️ Could not retrieve debug log: ${e.message}"
                    }
                }
            }
        }
        
        cleanup {
            echo "🧹 Cleaning up workspace..."
            // Clean Jenkins workspace
            cleanWs()
        }
    }
}
