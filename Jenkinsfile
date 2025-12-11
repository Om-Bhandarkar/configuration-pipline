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
        POSTGRES_IMAGE = "postgres:latest"
        REDIS_IMAGE = "redis:latest"
    }
    
    stages {
        stage('Input Validation') {
            steps {
                script {
                    if (!params.REMOTE_IP) {
                        error("❌ Remote IP address आवश्यक आहे!")
                    }
                    echo "✅ Remote IP: ${params.REMOTE_IP}"
                    echo "✅ OS Type: ${params.OS_TYPE}"
                }
            }
        }
        
        stage('Detect OS') {
            steps {
                script {
                    echo "🔍 Remote machine ची OS detect करत आहे..."
                    
                    if (params.OS_TYPE == 'linux') {
                        def osInfo = sh(script: """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} 'cat /etc/os-release'
                        """, returnStdout: true).trim()
                        echo "📋 OS Info:\n${osInfo}"
                    } else {
                        def osInfo = bat(script: """
                            @echo off
                            sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "systeminfo | findstr /B /C:\\"OS Name\\""
                        """, returnStdout: true).trim()
                        echo "📋 OS Info:\n${osInfo}"
                    }
                }
            }
        }
        
        stage('Linux Setup') {
            when {
                expression { params.OS_TYPE == 'linux' }
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
                                    docker tag ${POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:15
                                    docker tag ${REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:7-alpine
                                    
                                    # Push to registry
                                    docker push localhost:${params.REGISTRY_PORT}/postgres:15
                                    docker push localhost:${params.REGISTRY_PORT}/redis:7-alpine
                                '
                            """
                            echo "✅ Images successfully push झाल्या!"
                        }
                    }
                }
                
                stage('Deploy Containers on Linux') {
                    steps {
                        script {
                            echo "🚀 PostgreSQL आणि Redis containers deploy करत आहे..."
                            
                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    # Stop existing containers
                                    docker stop postgres-db redis-cache 2>/dev/null || true
                                    docker rm postgres-db redis-cache 2>/dev/null || true
                                    
                                    # Run PostgreSQL
                                    docker run -d --name postgres-db \
                                        -e POSTGRES_PASSWORD=postgres123 \
                                        -e POSTGRES_USER=postgres \
                                        -e POSTGRES_DB=mydb \
                                        -p 5432:5432 \
                                        --restart=always \
                                        localhost:${params.REGISTRY_PORT}/postgres:15
                                    
                                    # Run Redis
                                    docker run -d --name redis-cache \
                                        -p 6379:6379 \
                                        --restart=always \
                                        localhost:${params.REGISTRY_PORT}/redis:7-alpine
                                    
                                    # Wait for containers
                                    sleep 5
                                    
                                    # Verify containers
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
                expression { params.OS_TYPE == 'windows' }
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
                                    docker tag ${POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:15 && ^
                                    docker tag ${REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:7-alpine && ^
                                    docker push localhost:${params.REGISTRY_PORT}/postgres:15 && ^
                                    docker push localhost:${params.REGISTRY_PORT}/redis:7-alpine
                                "
                            """
                            echo "✅ Images successfully push झाल्या!"
                        }
                    }
                }
                
                stage('Deploy Containers on Windows') {
                    steps {
                        script {
                            echo "🚀 PostgreSQL आणि Redis containers deploy करत आहे..."
                            
                            bat """
                                sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                    docker stop postgres-db redis-cache 2>nul
                                    docker rm postgres-db redis-cache 2>nul
                                    
                                    docker run -d --name postgres-db -e POSTGRES_PASSWORD=postgres123 -e POSTGRES_USER=postgres -e POSTGRES_DB=mydb -p 5432:5432 --restart=always localhost:${params.REGISTRY_PORT}/postgres:15
                                    
                                    docker run -d --name redis-cache -p 6379:6379 --restart=always localhost:${params.REGISTRY_PORT}/redis:7-alpine
                                    
                                    timeout /t 5 /nobreak
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
                    
                    if (params.OS_TYPE == 'linux') {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                echo "=== Running Containers ==="
                                docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                                
                                echo ""
                                echo "=== Docker Registry Images ==="
                                curl -s http://localhost:${params.REGISTRY_PORT}/v2/_catalog | jq .
                            '
                        """
                    } else {
                        bat """
                            sshpass -p ${params.REMOTE_PASSWORD} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                echo === Running Containers === && ^
                                docker ps --format \"table {{.Names}}\\t{{.Status}}\\t{{.Ports}}\"
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
                ║ OS Type: ${params.OS_TYPE.toUpperCase()}
                ║ 
                ║ ✅ Docker & Docker Compose: Installed
                ║ ✅ Private Registry: Running on port ${params.REGISTRY_PORT}
                ║ ✅ PostgreSQL: Running on port 5432
                ║ ✅ Redis: Running on port 6379
                ${params.OS_TYPE == 'windows' ? '║ ✅ OpenSSH: Configured (Port 22)' : ''}
                ${params.OS_TYPE == 'windows' ? '║ ✅ Firewall: Enabled for SSH' : ''}
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
