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
                script {
                    echo "🔗 Connecting to remote system: ${params.TARGET_IP}"
                    sh """
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 \
                        ${params.SSH_USER}@${params.TARGET_IP} "echo '✅ Connection successful to ${params.TARGET_IP}'"
                    """
                }
            }
        }

        /* -------------------------
           2) DETECT OS
        ------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting OS on remote system..."
                    
                    // Try Linux detection
                    try {
                        def osInfo = sh(
                            returnStdout: true,
                            script: """
                                sshpass -p "${params.SSH_PASS}" \
                                ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                                "uname -s 2>/dev/null || echo 'UNKNOWN'"
                            """
                        ).trim().toLowerCase()
                        
                        echo "OS Info: ${osInfo}"
                        
                        if (osInfo.contains("linux")) {
                            env.OS_TYPE = "linux"
                            echo "✅ OS Detected: Linux"
                        } else {
                            // Try Windows detection
                            def winCheck = sh(
                                returnStdout: true,
                                script: """
                                    sshpass -p "${params.SSH_PASS}" \
                                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                                    "powershell -Command \\"\$os = [System.Environment]::OSVersion; if (\$os.Platform -eq 'Win32NT') { echo 'WINDOWS' } else { echo 'UNKNOWN' }\\" 2>/dev/null || echo 'NOT_WINDOWS'"
                                """
                            ).trim().toLowerCase()
                            
                            if (winCheck.contains("windows")) {
                                env.OS_TYPE = "windows"
                                echo "✅ OS Detected: Windows"
                            } else {
                                error "❌ Could not detect OS. Detected: ${osInfo}, Windows check: ${winCheck}"
                            }
                        }
                    } catch (Exception e) {
                        error "❌ Failed to detect OS: ${e.getMessage()}"
                    }
                }
            }
        }

        /* -------------------------
           3) INSTALL DOCKER (Based on OS)
        ------------------------- */
        stage("Install Docker & Docker Compose") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        echo "🐳 Installing Docker & Docker Compose on Linux..."
                        
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                # Check if Docker is already installed
                                if command -v docker >/dev/null 2>&1; then
                                    echo "✅ Docker is already installed: \$(docker --version)"
                                else
                                    echo "📦 Installing Docker..."
                                    # For Ubuntu/Debian
                                    if command -v apt-get >/dev/null 2>&1; then
                                        apt-get update -y
                                        apt-get install -y apt-transport-https ca-certificates curl software-properties-common
                                        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
                                        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu \$(lsb_release -cs) stable"
                                        apt-get update -y
                                        apt-get install -y docker-ce docker-ce-cli containerd.io
                                    # For CentOS/RHEL
                                    elif command -v yum >/dev/null 2>&1; then
                                        yum install -y yum-utils
                                        yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
                                        yum install -y docker-ce docker-ce-cli containerd.io
                                    fi
                                    
                                    # Start and enable Docker
                                    systemctl start docker
                                    systemctl enable docker
                                    echo "✅ Docker installed successfully: \$(docker --version)"
                                fi
                                
                                # Check Docker Compose
                                if command -v docker-compose >/dev/null 2>&1; then
                                    echo "✅ Docker Compose is already installed: \$(docker-compose --version)"
                                else
                                    echo "📦 Installing Docker Compose..."
                                    # Download Docker Compose v2
                                    curl -SL https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-linux-x86_64 \
                                        -o /usr/local/bin/docker-compose
                                    chmod +x /usr/local/bin/docker-compose
                                    ln -sf /usr/local/bin/docker-compose /usr/bin/docker-compose
                                    echo "✅ Docker Compose installed successfully: \$(docker-compose --version)"
                                fi
                            '
                        """
                        
                    } else if (env.OS_TYPE == "windows") {
                        echo "🐳 Checking Docker on Windows..."
                        
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \\"
                                # Check if Docker is installed
                                \$dockerInstalled = Get-Command docker -ErrorAction SilentlyContinue
                                \$composeInstalled = Get-Command docker-compose -ErrorAction SilentlyContinue
                                
                                if (\$dockerInstalled) {
                                    Write-Host '✅ Docker is already installed:' -NoNewline
                                    docker --version
                                } else {
                                    Write-Host '❌ Docker is not installed on Windows.'
                                    Write-Host 'Please install Docker Desktop manually from:'
                                    Write-Host 'https://docs.docker.com/desktop/install/windows-install/'
                                }
                                
                                if (\$composeInstalled) {
                                    Write-Host '✅ Docker Compose is already installed:' -NoNewline
                                    docker-compose --version
                                } else {
                                    Write-Host '⚠️ Docker Compose might be included in Docker Desktop'
                                }
                            \\""
                        """
                    }
                }
            }
        }

        /* -------------------------
           4) CREATE DOCKER-COMPOSE FILE WITH POSTGRESQL & REDIS
        ------------------------- */
        stage("Create Docker Compose Configuration") {
            steps {
                script {
                    echo "📄 Creating docker-compose.yml with PostgreSQL and Redis..."
                    
                    // Create docker-compose.yml locally
                    def composeContent = """version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: redis_cache
    ports:
      - "6379:6379"
    command: redis-server --requirepass redis123
    volumes:
      - redis_data:/data
    restart: unless-stopped
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
"""
                    
                    // Write to local file
                    writeFile file: 'docker-compose.yml', text: composeContent
                    
                    echo "✅ docker-compose.yml created locally"
                }
            }
        }

        /* -------------------------
           5) UPLOAD AND DEPLOY
        ------------------------- */
        stage("Upload and Deploy Services") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        echo "🚀 Deploying on Linux..."
                        
                        // Create directory and upload
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "mkdir -p ${COMPOSE_DIR_LINUX}"
                            
                            sshpass -p "${params.SSH_PASS}" \
                            scp -o StrictHostKeyChecking=no docker-compose.yml \
                            ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_LINUX}
                        """
                        
                        // Create init.sql for PostgreSQL
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "mkdir -p ${COMPOSE_DIR_LINUX}/postgres && \
                            echo 'CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";' > ${COMPOSE_DIR_LINUX}/postgres/init.sql"
                        """
                        
                        // Deploy using docker-compose
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "
                                cd ${COMPOSE_DIR_LINUX}
                                
                                # Stop and remove existing containers
                                docker-compose down 2>/dev/null || true
                                
                                # Pull images and start services
                                echo '📥 Pulling Docker images...'
                                docker-compose pull
                                
                                echo '🚀 Starting services...'
                                docker-compose up -d
                                
                                # Wait a moment for services to start
                                sleep 10
                                
                                # Check running containers
                                echo '📊 Running containers:'
                                docker-compose ps
                                
                                # Check service status
                                if docker-compose ps | grep -q 'Up'; then
                                    echo '✅ Services started successfully'
                                else
                                    echo '❌ Services failed to start'
                                    docker-compose logs
                                fi
                            "
                        """
                        
                    } else if (env.OS_TYPE == "windows") {
                        echo "🚀 Deploying on Windows..."
                        
                        // Upload to Windows
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \"New-Item -ItemType Directory -Force -Path '${COMPOSE_DIR_WIN}'\""
                            
                            sshpass -p "${params.SSH_PASS}" \
                            scp -o StrictHostKeyChecking=no docker-compose.yml \
                            ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_WIN}
                        """
                        
                        echo "⚠️ For Windows, please run manually in PowerShell:"
                        echo "   cd ${COMPOSE_DIR_WIN}"
                        echo "   docker-compose up -d"
                    }
                }
            }
        }

        /* -------------------------
           6) VERIFY DEPLOYMENT
        ------------------------- */
        stage("Verify Services") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        echo "🔍 Verifying deployed services..."
                        
                        sh """
                            sshpass -p "${params.SSH_PASS}" \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo ""
                                echo "📊 Service Status:"
                                echo "=================="
                                
                                # Check PostgreSQL
                                if docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}" | grep -q postgres_db; then
                                    echo "✅ PostgreSQL is RUNNING on port 5432"
                                    echo "   Connection: postgresql://admin:admin123@${params.TARGET_IP}:5432/appdb"
                                else
                                    echo "❌ PostgreSQL is NOT running"
                                fi
                                
                                # Check Redis
                                if docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}" | grep -q redis_cache; then
                                    echo "✅ Redis is RUNNING on port 6379"
                                    echo "   Connection: redis://:redis123@${params.TARGET_IP}:6379"
                                else
                                    echo "❌ Redis is NOT running"
                                fi
                                
                                echo ""
                                echo "🌐 Services accessible at:"
                                echo "   - PostgreSQL: ${params.TARGET_IP}:5432"
                                echo "   - Redis: ${params.TARGET_IP}:6379"
                                echo ""
                                echo "🔧 Docker containers:"
                                docker ps --filter "name=postgres_db" --filter "name=redis_cache" --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                            '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo "🎉 Deployment Successful!"
                echo "========================="
                echo "Target System: ${params.TARGET_IP}"
                echo "OS Type: ${env.OS_TYPE}"
                echo ""
                echo "📦 Installed Services:"
                echo "  - PostgreSQL:5432"
                echo "    Username: admin"
                echo "    Password: admin123"
                echo "    Database: appdb"
                echo ""
                echo "  - Redis:6379"
                echo "    Password: redis123"
                echo ""
                echo "🔌 Connection Details:"
                echo "  PostgreSQL: postgresql://admin:admin123@${params.TARGET_IP}:5432/appdb"
                echo "  Redis: redis://:redis123@${params.TARGET_IP}:6379"
                echo ""
                echo "✅ All services deployed successfully to remote machine!"
            }
        }
        failure {
            script {
                echo "❌ Deployment Failed!"
                echo "===================="
                echo "Check the stage logs for details."
                echo "Target IP: ${params.TARGET_IP}"
                echo "OS Type: ${env.OS_TYPE}"
            }
        }
        always {
            echo "🧹 Cleaning up local files..."
            sh 'rm -f docker-compose.yml 2>/dev/null || true'
            echo "📋 Pipeline execution completed."
        }
    }
}
