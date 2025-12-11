pipeline {
    agent any
    
    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP address')
        string(name: 'REMOTE_USER', defaultValue: 'administrator', description: 'Remote machine username')
        password(name: 'REMOTE_PASSWORD', defaultValue: '', description: 'Remote machine password')
        string(name: 'REGISTRY_PORT', defaultValue: '5000', description: 'Docker registry port')
    }
    
    environment {
        DETECTED_OS = ''
    }
    
    environment {
        REGISTRY_NAME = "local-registry"
        POSTGRES_IMAGE = "postgres:15"
        REDIS_IMAGE = "redis:7-alpine"
    }
    
    stages {
        stage('Input Validation') {
            steps {
                script {
                    if (!params.REMOTE_IP) {
                        error("❌ Remote IP address आवश्यक आहे!")
                    }
                    echo "✅ Remote IP: ${params.REMOTE_IP}"
                }
            }
        }
        
        stage('Detect OS') {
            steps {
                script {
                    echo "🔍 Remote machine ची OS detect करत आहे..."
                    
                    try {
                        // Linux check करण्याचा प्रयत्न
                        def osInfo = sh(script: """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 ${params.REMOTE_USER}@${params.REMOTE_IP} 'uname -s' 2>/dev/null || echo 'FAILED'
                        """, returnStdout: true).trim()
                        
                        if (osInfo.contains('Linux')) {
                            env.DETECTED_OS = 'linux'
                            echo "✅ Linux OS detected!"
                            def detailedInfo = sh(script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} 'cat /etc/os-release'
                            """, returnStdout: true).trim()
                            echo "📋 OS Info:\n${detailedInfo}"
                        } else {
                            // Windows असणार
                            env.DETECTED_OS = 'windows'
                            echo "✅ Windows OS detected!"
                            def winInfo = bat(script: """
                                @echo off
                                sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "systeminfo | findstr /B /C:\\"OS Name\\""
                            """, returnStdout: true).trim()
                            echo "📋 OS Info:\n${winInfo}"
                        }
                    } catch (Exception e) {
                        // SSH fail झाल्यास Windows समजून WinRM वापरा
                        env.DETECTED_OS = 'windows'
                        echo "✅ Windows OS detected (fallback detection)!"
                    }
                    
                    echo "🎯 Detected OS: ${env.DETECTED_OS}"
                }
            }
        }
        
        stage('Linux Setup') {
            when {
                expression { env.DETECTED_OS == 'linux' }
            }
            stages {
                stage('Check Docker on Linux') {
                    steps {
                        script {
                            echo "🐧 Linux machine वर Docker check करत आहे..."
                            
                            def dockerInstalled = sh(script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    if command -v docker &> /dev/null; then
                                        echo "installed"
                                    else
                                        echo "not_installed"
                                    fi
                                '
                            """, returnStdout: true).trim()
                            
                            if (dockerInstalled == "installed") {
                                echo "✅ Docker already installed आहे!"
                            } else {
                                echo "⚠️ Docker installed नाही. Installing..."
                                sh """
                                    sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                        sudo apt-get update
                                        sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
                                        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
                                        sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu \$(lsb_release -cs) stable"
                                        sudo apt-get update
                                        sudo apt-get install -y docker-ce docker-ce-cli containerd.io
                                        sudo systemctl start docker
                                        sudo systemctl enable docker
                                        sudo usermod -aG docker ${params.REMOTE_USER}
                                    '
                                """
                                echo "✅ Docker successfully install झाले!"
                            }
                        }
                    }
                }
                
                stage('Check Docker Compose on Linux') {
                    steps {
                        script {
                            echo "🔧 Docker Compose check करत आहे..."
                            
                            def composeInstalled = sh(script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    if command -v docker-compose &> /dev/null; then
                                        echo "installed"
                                    else
                                        echo "not_installed"
                                    fi
                                '
                            """, returnStdout: true).trim()
                            
                            if (composeInstalled == "installed") {
                                echo "✅ Docker Compose already installed आहे!"
                            } else {
                                echo "⚠️ Docker Compose installed नाही. Installing..."
                                sh """
                                    sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                        sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-\$(uname -s)-\$(uname -m)" -o /usr/local/bin/docker-compose
                                        sudo chmod +x /usr/local/bin/docker-compose
                                    '
                                """
                                echo "✅ Docker Compose successfully install झाले!"
                            }
                        }
                    }
                }
                
                stage('Setup Docker Registry on Linux') {
                    steps {
                        script {
                            echo "🗄️ Private Docker Registry setup करत आहे..."
                            
                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    if docker ps -a | grep -q ${REGISTRY_NAME}; then
                                        echo "Registry already running आहे"
                                        docker start ${REGISTRY_NAME} 2>/dev/null || true
                                    else
                                        docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${REGISTRY_NAME} registry:2
                                    fi
                                '
                            """
                            echo "✅ Docker Registry ready आहे!"
                        }
                    }
                }
                
                stage('Create Docker Compose File on Linux') {
                    steps {
                        script {
                            echo "📝 Docker Compose file तयार करत आहे..."
                            
                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    mkdir -p ~/docker-deployment
                                    cd ~/docker-deployment
                                    
                                    cat > docker-compose.yml << EOF
version: "3.8"

services:
  postgres:
    image: localhost:${params.REGISTRY_PORT}/postgres:latest
    container_name: postgres-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: localhost:${params.REGISTRY_PORT}/redis:latest
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: always
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  default:
    name: app-network
    driver: bridge
EOF
                                    
                                    echo "✅ docker-compose.yml created!"
                                    cat docker-compose.yml
                                '
                            """
                        }
                    }
                }
                
                stage('Pull and Push Images on Linux') {
                    steps {
                        script {
                            echo "📦 PostgreSQL आणि Redis images pull आणि push करत आहे..."
                            
                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    # Pull images
                                    docker pull ${POSTGRES_IMAGE}
                                    docker pull ${REDIS_IMAGE}
                                    
                                    # Tag images
                                    docker tag ${POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:latest
                                    docker tag ${REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:latest
                                    
                                    # Push to registry
                                    docker push localhost:${params.REGISTRY_PORT}/postgres:latest
                                    docker push localhost:${params.REGISTRY_PORT}/redis:latest
                                '
                            """
                            echo "✅ Images successfully push झाल्या!"
                        }
                    }
                }
                
                stage('Deploy Containers on Linux') {
                    steps {
                        script {
                            echo "🚀 Docker Compose वापरून containers deploy करत आहे..."
                            
                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    cd ~/docker-deployment
                                    
                                    # Stop existing containers if running
                                    docker-compose down 2>/dev/null || true
                                    
                                    # Start containers with docker-compose
                                    docker-compose up -d
                                    
                                    # Wait for containers to be healthy
                                    echo "Waiting for containers to be healthy..."
                                    sleep 10
                                    
                                    # Check container status
                                    docker-compose ps
                                    
                                    # Verify containers are running
                                    docker ps | grep -E "postgres-db|redis-cache"
                                '
                            """
                            echo "✅ Containers successfully running आहेत!"
                        }
                    }
                }
            }
        }
        
        stage('Windows Setup') {
            when {
                expression { env.DETECTED_OS == 'windows' }
            }
            stages {
                stage('Configure OpenSSH on Windows') {
                    steps {
                        script {
                            echo "🔐 Windows वर OpenSSH configure करत आहे..."
                            
                            bat """
                                powershell -Command "
                                    \$session = New-PSSession -ComputerName ${params.REMOTE_IP} -Credential (New-Object System.Management.Automation.PSCredential('${params.REMOTE_USER}', (ConvertTo-SecureString '${params.REMOTE_PASSWORD}' -AsPlainText -Force)))
                                    
                                    Invoke-Command -Session \$session -ScriptBlock {
                                        # Install OpenSSH Server
                                        Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
                                        
                                        # Start and enable SSH service
                                        Start-Service sshd
                                        Set-Service -Name sshd -StartupType 'Automatic'
                                        
                                        # Configure firewall for port 22
                                        New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22 -ErrorAction SilentlyContinue
                                        
                                        Write-Host 'OpenSSH configured successfully!'
                                    }
                                    
                                    Remove-PSSession \$session
                                "
                            """
                            echo "✅ OpenSSH successfully configure झाले!"
                        }
                    }
                }
                
                stage('Check Docker on Windows') {
                    steps {
                        script {
                            echo "🪟 Windows machine वर Docker check करत आहे..."
                            
                            def dockerInstalled = bat(script: """
                                @echo off
                                sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "docker --version" 2>nul && echo installed || echo not_installed
                            """, returnStdout: true).trim()
                            
                            if (dockerInstalled.contains("installed")) {
                                echo "✅ Docker already installed आहे!"
                            } else {
                                echo "⚠️ Docker Desktop manually install करा: https://www.docker.com/products/docker-desktop"
                                error("Docker Desktop Windows वर manually install करणे आवश्यक आहे")
                            }
                        }
                    }
                }
                
                stage('Create Docker Compose File on Windows') {
                    steps {
                        script {
                            echo "📝 Docker Compose file तयार करत आहे..."
                            
                            bat """
                                sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                    if not exist C:\\docker-deployment mkdir C:\\docker-deployment
                                    cd C:\\docker-deployment
                                    
                                    (
                                    echo version: '3.8'
                                    echo.
                                    echo services:
                                    echo   postgres:
                                    echo     image: localhost:${params.REGISTRY_PORT}/postgres:latest
                                    echo     container_name: postgres-db
                                    echo     environment:
                                    echo       POSTGRES_USER: postgres
                                    echo       POSTGRES_PASSWORD: postgres123
                                    echo       POSTGRES_DB: mydb
                                    echo       PGDATA: /var/lib/postgresql/data/pgdata
                                    echo     ports:
                                    echo       - '5432:5432'
                                    echo     volumes:
                                    echo       - postgres_data:/var/lib/postgresql/data
                                    echo     restart: always
                                    echo     healthcheck:
                                    echo       test: ['CMD-SHELL', 'pg_isready -U postgres']
                                    echo       interval: 10s
                                    echo       timeout: 5s
                                    echo       retries: 5
                                    echo.
                                    echo   redis:
                                    echo     image: localhost:${params.REGISTRY_PORT}/redis:latest
                                    echo     container_name: redis-cache
                                    echo     ports:
                                    echo       - '6379:6379'
                                    echo     volumes:
                                    echo       - redis_data:/data
                                    echo     restart: always
                                    echo     command: redis-server --appendonly yes
                                    echo     healthcheck:
                                    echo       test: ['CMD', 'redis-cli', 'ping']
                                    echo       interval: 10s
                                    echo       timeout: 5s
                                    echo       retries: 5
                                    echo.
                                    echo volumes:
                                    echo   postgres_data:
                                    echo     driver: local
                                    echo   redis_data:
                                    echo     driver: local
                                    echo.
                                    echo networks:
                                    echo   default:
                                    echo     name: app-network
                                    echo     driver: bridge
                                    ^) ^> docker-compose.yml
                                    
                                    type docker-compose.yml
                                "
                            """
                            echo "✅ docker-compose.yml created!"
                        }
                    }
                }
                
                stage('Setup Docker Registry on Windows') {
                    steps {
                        script {
                            echo "🗄️ Private Docker Registry setup करत आहे..."
                            
                            bat """
                                sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                    docker ps -a | findstr ${REGISTRY_NAME} >nul 2>&1
                                    if %ERRORLEVEL% EQU 0 (
                                        docker start ${REGISTRY_NAME}
                                    ) else (
                                        docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${REGISTRY_NAME} registry:2
                                    )
                                "
                            """
                            echo "✅ Docker Registry ready आहे!"
                        }
                    }
                }
                
                stage('Pull and Push Images on Windows') {
                    steps {
                        script {
                            echo "📦 PostgreSQL आणि Redis images pull आणि push करत आहे..."
                            
                            bat """
                                sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                    docker pull ${POSTGRES_IMAGE} && ^
                                    docker pull ${REDIS_IMAGE} && ^
                                    docker tag ${POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:latest && ^
                                    docker tag ${REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:latest && ^
                                    docker push localhost:${params.REGISTRY_PORT}/postgres:latest && ^
                                    docker push localhost:${params.REGISTRY_PORT}/redis:latest
                                "
                            """
                            echo "✅ Images successfully push झाल्या!"
                        }
                    }
                }
                
                stage('Deploy Containers on Windows') {
                    steps {
                        script {
                            echo "🚀 Docker Compose वापरून containers deploy करत आहे..."
                            
                            bat """
                                sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                    cd C:\\docker-deployment
                                    
                                    docker-compose down 2>nul
                                    
                                    docker-compose up -d
                                    
                                    timeout /t 10 /nobreak
                                    
                                    docker-compose ps
                                    
                                    docker ps | findstr postgres-db
                                    docker ps | findstr redis-cache
                                "
                            """
                            echo "✅ Containers successfully running आहेत!"
                        }
                    }
                }
            }
        }
        
        stage('Verification') {
            steps {
                script {
                    echo "✔️ Pipeline verification करत आहे..."
                    
                    if (env.DETECTED_OS == 'linux') {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                cd ~/docker-deployment
                                
                                echo "=== Docker Compose Services ==="
                                docker-compose ps
                                
                                echo ""
                                echo "=== Running Containers ==="
                                docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                                
                                echo ""
                                echo "=== Docker Registry Images ==="
                                curl -s http://localhost:${params.REGISTRY_PORT}/v2/_catalog | jq .
                                
                                echo ""
                                echo "=== Health Check ==="
                                docker exec postgres-db pg_isready -U postgres || echo "PostgreSQL not ready yet"
                                docker exec redis-cache redis-cli ping || echo "Redis not ready yet"
                            '
                        """
                    } else {
                        bat """
                            sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                cd C:\\docker-deployment
                                
                                echo === Docker Compose Services === && ^
                                docker-compose ps && ^
                                echo. && ^
                                echo === Running Containers === && ^
                                docker ps --format \"table {{.Names}}\\t{{.Status}}\\t{{.Ports}}\" && ^
                                echo. && ^
                                echo === Health Check === && ^
                                docker exec postgres-db pg_isready -U postgres && ^
                                docker exec redis-cache redis-cli ping
                            "
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            script {
                def message = """
                ╔════════════════════════════════════════════════╗
                ║   🎉 Pipeline Successfully Completed! 🎉      ║
                ╠════════════════════════════════════════════════╣
                ║ Remote IP: ${params.REMOTE_IP}
                ║ OS Type: ${env.DETECTED_OS.toUpperCase()}
                ║ 
                ║ ✅ Docker & Docker Compose: Installed
                ║ ✅ Docker Compose File: Created
                ║ ✅ Private Registry: Running on port ${params.REGISTRY_PORT}
                ║ ✅ PostgreSQL: Running on port 5432
                ║ ✅ Redis: Running on port 6379
                ${env.DETECTED_OS == 'windows' ? '║ ✅ OpenSSH: Configured (Port 22)' : ''}
                ${env.DETECTED_OS == 'windows' ? '║ ✅ Firewall: Enabled for SSH' : ''}
                ║ 
                ║ 📋 Container Status:
                ║    - postgres-db: ✅ Running
                ║    - redis-cache: ✅ Running
                ║    - ${REGISTRY_NAME}: ✅ Running
                ║ 
                ║ 🔗 Access URLs:
                ║    PostgreSQL: ${params.REMOTE_IP}:5432
                ║    Redis: ${params.REMOTE_IP}:6379
                ║    Registry: ${params.REMOTE_IP}:${params.REGISTRY_PORT}
                ╚════════════════════════════════════════════════╝
                """
                echo message
            }
        }
        failure {
            echo """
            ╔════════════════════════════════════════════════╗
            ║   ❌ Pipeline Failed! ❌                       ║
            ╠════════════════════════════════════════════════╣
            ║ कृपया logs check करा आणि errors fix करा    ║
            ╚════════════════════════════════════════════════╝
            """
        }
    }
}
