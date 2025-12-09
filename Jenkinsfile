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
                        sshpass -p '${params.SSH_PASS}' \
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
                                sshpass -p '${params.SSH_PASS}' \
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
                                    sshpass -p '${params.SSH_PASS}' \
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
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -t -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                # Check if Docker is already installed
                                echo "=== Checking Docker Installation ==="
                                if command -v docker >/dev/null 2>&1; then
                                    echo "✅ Docker is already installed: \$(docker --version)"
                                else
                                    echo "📦 Installing Docker..."
                                    # For Ubuntu/Debian
                                    if command -v apt-get >/dev/null 2>&1; then
                                        apt-get update -y
                                        apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release
                                        mkdir -p /etc/apt/keyrings
                                        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
                                        echo \\
                                          "deb [arch=\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \\
                                          \$(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
                                        apt-get update -y
                                        apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
                                    # For CentOS/RHEL
                                    elif command -v yum >/dev/null 2>&1; then
                                        yum install -y yum-utils
                                        yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
                                        yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
                                    else
                                        echo "❌ Unsupported package manager"
                                        exit 1
                                    fi
                                    
                                    # Start and enable Docker
                                    systemctl start docker
                                    systemctl enable docker
                                    usermod -aG docker \$USER || true
                                    echo "✅ Docker installed successfully: \$(docker --version)"
                                fi
                                
                                # Check Docker Compose
                                echo ""
                                echo "=== Checking Docker Compose ==="
                                if docker compose version >/dev/null 2>&1; then
                                    echo "✅ Docker Compose (plugin) is available: \$(docker compose version)"
                                elif command -v docker-compose >/dev/null 2>&1; then
                                    echo "✅ Docker Compose (standalone) is available: \$(docker-compose --version)"
                                else
                                    echo "📦 Installing Docker Compose..."
                                    # Install Docker Compose standalone
                                    curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-\$(uname -s)-\$(uname -m)" \\
                                        -o /usr/local/bin/docker-compose
                                    chmod +x /usr/local/bin/docker-compose
                                    ln -sf /usr/local/bin/docker-compose /usr/bin/docker-compose
                                    echo "✅ Docker Compose installed: \$(docker-compose --version)"
                                fi
                                
                                echo ""
                                echo "=== Verification ==="
                                docker --version
                                docker info --format "{{.ServerVersion}}"
                            '
                        """
                        
                    } else if (env.OS_TYPE == "windows") {
                        echo "🐳 Checking Docker on Windows..."
                        
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
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
                                    exit 1
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
           4) CREATE DOCKER-COMPOSE FILE
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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 30s
      timeout: 10s
      retries: 3

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
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

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
           5) UPLOAD FILES TO REMOTE
        ------------------------- */
        stage("Upload Files to Remote Server") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        echo "📤 Uploading files to remote server..."
                        
                        // Create directory structure
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "mkdir -p ${COMPOSE_DIR_LINUX}/postgres"
                        """
                        
                        // Upload docker-compose.yml
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            scp -o StrictHostKeyChecking=no docker-compose.yml \
                            ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_LINUX}
                        """
                        
                        // Create init.sql file
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "echo 'CREATE EXTENSION IF NOT EXISTS \\\"uuid-ossp\\\";' > ${COMPOSE_DIR_LINUX}/postgres/init.sql && \\
                             echo 'SELECT \\'PostgreSQL initialized successfully\\';' >> ${COMPOSE_DIR_LINUX}/postgres/init.sql"
                        """
                        
                        // Verify uploaded files
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "echo '📁 Files in ${COMPOSE_DIR_LINUX}:' && ls -la ${COMPOSE_DIR_LINUX}/ && \\
                             echo '' && \\
                             echo '📄 docker-compose.yml content (first 10 lines):' && head -10 ${COMPOSE_FILE_LINUX}"
                        """
                        
                    } else if (env.OS_TYPE == "windows") {
                        echo "📤 Uploading to Windows..."
                        
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                            "powershell -Command \"New-Item -ItemType Directory -Force -Path '${COMPOSE_DIR_WIN}'\""
                            
                            sshpass -p '${params.SSH_PASS}' \
                            scp -o StrictHostKeyChecking=no docker-compose.yml \
                            ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE_WIN}
                        """
                    }
                }
            }
        }

        /* -------------------------
           6) DEPLOY SERVICES
        ------------------------- */
        stage("Deploy Services") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        echo "🚀 Deploying services on Linux..."
                        
                        // Stop any existing containers
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -t -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                cd ${COMPOSE_DIR_LINUX}
                                echo "Stopping existing containers..."
                                docker-compose down 2>/dev/null || true
                                docker rm -f postgres_db redis_cache 2>/dev/null || true
                                docker volume prune -f 2>/dev/null || true
                            '
                        """
                        
                        sleep 5
                        
                        // Pull images
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -t -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                cd ${COMPOSE_DIR_LINUX}
                                echo "Pulling Docker images..."
                                docker-compose pull || docker compose pull
                            '
                        """
                        
                        sleep 5
                        
                        // Start services
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -tt -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                cd ${COMPOSE_DIR_LINUX}
                                echo "Starting services..."
                                # Try docker-compose first, then docker compose
                                docker-compose up -d || docker compose up -d
                                echo "Services started. Waiting for initialization..."
                            '
                        """
                        
                        sleep 15
                        
                        // Check status
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo ""
                                echo "=== Service Status ==="
                                cd ${COMPOSE_DIR_LINUX}
                                docker-compose ps || docker compose ps
                                
                                echo ""
                                echo "=== All Docker Containers ==="
                                docker ps -a --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                                
                                echo ""
                                echo "=== Network Information ==="
                                docker network inspect app-network 2>/dev/null || echo "Network not found yet"
                            '
                        """
                        
                    } else if (env.OS_TYPE == "windows") {
                        echo "⚠️ Windows deployment requires manual steps:"
                        echo "1. SSH to ${params.TARGET_IP}"
                        echo "2. Run: cd ${COMPOSE_DIR_WIN}"
                        echo "3. Run: docker-compose up -d"
                    }
                }
            }
        }

        /* -------------------------
           7) DEBUG AND VERIFY
        ------------------------- */
        stage("Debug and Verify") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        echo "🔍 Debugging and verifying deployment..."
                        
                        // Check containers in detail
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo ""
                                echo "=== DETAILED CONTAINER CHECK ==="
                                
                                # Check PostgreSQL
                                echo "1. PostgreSQL Container:"
                                if docker ps --filter "name=postgres_db" --format "{{.Names}}" | grep -q postgres_db; then
                                    echo "   ✅ Container is RUNNING"
                                    echo "   📊 Status: \$(docker ps --filter "name=postgres_db" --format "{{.Status}}")"
                                    echo "   🔌 Ports: \$(docker ps --filter "name=postgres_db" --format "{{.Ports}}")"
                                    
                                    # Check logs
                                    echo "   📝 Last 5 log lines:"
                                    docker logs --tail 5 postgres_db 2>/dev/null || echo "   No logs available"
                                else
                                    echo "   ❌ Container is NOT running"
                                    echo "   Checking if container exists:"
                                    docker ps -a --filter "name=postgres_db" --format "table {{.Names}}\\t{{.Status}}"
                                fi
                                
                                echo ""
                                echo "2. Redis Container:"
                                if docker ps --filter "name=redis_cache" --format "{{.Names}}" | grep -q redis_cache; then
                                    echo "   ✅ Container is RUNNING"
                                    echo "   📊 Status: \$(docker ps --filter "name=redis_cache" --format "{{.Status}}")"
                                    echo "   🔌 Ports: \$(docker ps --filter "name=redis_cache" --format "{{.Ports}}")"
                                    
                                    # Check logs
                                    echo "   📝 Last 5 log lines:"
                                    docker logs --tail 5 redis_cache 2>/dev/null || echo "   No logs available"
                                else
                                    echo "   ❌ Container is NOT running"
                                    echo "   Checking if container exists:"
                                    docker ps -a --filter "name=redis_cache" --format "table {{.Names}}\\t{{.Status}}"
                                fi
                                
                                echo ""
                                echo "3. Checking Network Ports:"
                                netstat -tulpn 2>/dev/null | grep -E "(5432|6379)" || \\
                                ss -tulpn 2>/dev/null | grep -E "(5432|6379)" || \\
                                echo "   Port check not available or ports not listening"
                                
                                echo ""
                                echo "4. Docker System Information:"
                                echo "   Images:"
                                docker images --filter "reference=postgres*" --filter "reference=redis*" --format "table {{.Repository}}\\t{{.Tag}}\\t{{.Size}}"
                                
                                echo ""
                                echo "   Volumes:"
                                docker volume ls --filter "name=postgres_data" --filter "name=redis_data"
                                
                                echo ""
                                echo "5. Direct Container Check:"
                                echo "   Trying to list all containers with details:"
                                docker ps -a
                            '
                        """
                        
                        sleep 10
                        
                        // Test connections
                        echo "🔌 Testing service connections..."
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo ""
                                echo "=== CONNECTION TESTS ==="
                                
                                # Test PostgreSQL connection
                                echo "1. Testing PostgreSQL (port 5432):"
                                timeout 10 bash -c "echo > /dev/tcp/localhost/5432" 2>/dev/null
                                if [ \$? -eq 0 ]; then
                                    echo "   ✅ PostgreSQL port is open"
                                    echo "   Trying to connect with psql..."
                                    docker exec postgres_db pg_isready -U admin 2>/dev/null && \\
                                        echo "   ✅ PostgreSQL is accepting connections" || \\
                                        echo "   ⚠️ PostgreSQL port open but not responding"
                                else
                                    echo "   ❌ PostgreSQL port is closed"
                                fi
                                
                                # Test Redis connection
                                echo ""
                                echo "2. Testing Redis (port 6379):"
                                timeout 10 bash -c "echo > /dev/tcp/localhost/6379" 2>/dev/null
                                if [ \$? -eq 0 ]; then
                                    echo "   ✅ Redis port is open"
                                    echo "   Testing Redis authentication..."
                                    docker exec redis_cache redis-cli -a redis123 ping 2>/dev/null | grep -q PONG && \\
                                        echo "   ✅ Redis is responding" || \\
                                        echo "   ⚠️ Redis port open but not responding correctly"
                                else
                                    echo "   ❌ Redis port is closed"
                                fi
                            '
                        """
                    }
                }
            }
        }

        /* -------------------------
           8) FINAL VERIFICATION
        ------------------------- */
        stage("Final Verification") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        echo "✅ Final verification..."
                        
                        sh """
                            sshpass -p '${params.SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                                echo ""
                                echo "========================================"
                                echo "          DEPLOYMENT SUMMARY           "
                                echo "========================================"
                                echo "Target Server: ${params.TARGET_IP}"
                                echo "OS Type: ${env.OS_TYPE}"
                                echo ""
                                
                                echo "📦 DEPLOYED SERVICES:"
                                echo "---------------------"
                                
                                # PostgreSQL Status
                                PG_STATUS=\$(docker inspect -f '{{.State.Status}}' postgres_db 2>/dev/null || echo "NOT_FOUND")
                                if [ "\$PG_STATUS" = "running" ]; then
                                    echo "✅ PostgreSQL: RUNNING"
                                    echo "   • Port: 5432"
                                    echo "   • User: admin"
                                    echo "   • Password: admin123"
                                    echo "   • Database: appdb"
                                    echo "   • Connection: postgresql://admin:admin123@${params.TARGET_IP}:5432/appdb"
                                else
                                    echo "❌ PostgreSQL: \$PG_STATUS"
                                fi
                                
                                echo ""
                                
                                # Redis Status
                                REDIS_STATUS=\$(docker inspect -f '{{.State.Status}}' redis_cache 2>/dev/null || echo "NOT_FOUND")
                                if [ "\$REDIS_STATUS" = "running" ]; then
                                    echo "✅ Redis: RUNNING"
                                    echo "   • Port: 6379"
                                    echo "   • Password: redis123"
                                    echo "   • Connection: redis://:redis123@${params.TARGET_IP}:6379"
                                else
                                    echo "❌ Redis: \$REDIS_STATUS"
                                fi
                                
                                echo ""
                                echo "🔧 CONTAINER DETAILS:"
                                echo "---------------------"
                                docker ps --filter "name=postgres_db" --filter "name=redis_cache" \\
                                    --format "table {{.Names}}\\t{{.Image}}\\t{{.Status}}\\t{{.Ports}}"
                                
                                echo ""
                                echo "📊 RESOURCE USAGE:"
                                echo "------------------"
                                docker stats --no-stream --format "table {{.Name}}\\t{{.CPUPerc}}\\t{{.MemUsage}}\\t{{.NetIO}}" \\
                                    postgres_db redis_cache 2>/dev/null || echo "Stats not available yet"
                                
                                echo ""
                                echo "========================================"
                                echo "Deployment completed at: \$(date)"
                                echo "========================================"
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
                echo "📦 Deployed Services:"
                echo "  ✅ PostgreSQL on port 5432"
                echo "  ✅ Redis on port 6379"
                echo ""
                echo "🔌 Connection Details:"
                echo "  PostgreSQL: postgresql://admin:admin123@${params.TARGET_IP}:5432/appdb"
                echo "  Redis: redis://:redis123@${params.TARGET_IP}:6379"
                echo ""
                echo "✅ All services deployed successfully!"
            }
        }
        failure {
            script {
                echo "❌ Deployment Failed!"
                echo "===================="
                echo "Target IP: ${params.TARGET_IP}"
                echo "OS Type: ${env.OS_TYPE}"
                echo ""
                echo "📋 Troubleshooting Steps:"
                echo "1. Check SSH connectivity: ssh ${params.SSH_USER}@${params.TARGET_IP}"
                echo "2. Verify Docker is installed on remote"
                echo "3. Check Docker daemon status: systemctl status docker"
                echo "4. Manual deployment:"
                echo "   ssh ${params.SSH_USER}@${params.TARGET_IP}"
                echo "   cd /infra"
                echo "   docker-compose up -d"
            }
        }
        always {
            echo "🧹 Cleaning up..."
            sh 'rm -f docker-compose.yml 2>/dev/null || true'
            echo "📋 Pipeline execution completed at: ${new Date().format('yyyy-MM-dd HH:mm:ss')}"
            
            // Send notification
            script {
                if (currentBuild.result == 'SUCCESS') {
                    echo "✅ Pipeline completed successfully!"
                } else if (currentBuild.result == 'FAILURE') {
                    echo "❌ Pipeline failed!"
                } else {
                    echo "⚠️ Pipeline completed with status: ${currentBuild.result}"
                }
            }
        }
    }
}
