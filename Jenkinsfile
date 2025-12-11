pipeline {
    agent any
    
    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP address')
        string(name: 'REMOTE_USER', defaultValue: 'administrator', description: 'Remote machine username')
        password(name: 'REMOTE_PASSWORD', defaultValue: '', description: 'Remote machine password')
        string(name: 'REGISTRY_PORT', defaultValue: '5000', description: 'Docker registry port')
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

                    detectedOS = "unknown"

                    try {
                        def osInfo = sh(
                            script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 ${params.REMOTE_USER}@${params.REMOTE_IP} 'uname -s'
                            """,
                            returnStdout: true
                        ).trim()

                        if (osInfo.contains("Linux")) {
                            detectedOS = "linux"
                            echo "✅ Linux OS detected!"
                        }
                    } catch(e) {
                        echo "SSH Linux check failed → Checking Windows..."
                    }

                    if (detectedOS == "unknown") {
                        try {
                            def winInfo = sh(
                                script: """
                                    sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "powershell -Command \\"(Get-WmiObject Win32_OperatingSystem).Caption\\""
                                """,
                                returnStdout: true
                            ).trim()

                            if (winInfo) {
                                detectedOS = "windows"
                                echo "✅ Windows OS detected!"
                            }
                        } catch(e) {
                            detectedOS = "windows"  // fallback
                        }
                    }

                    echo "🎯 Final Detected OS: ${detectedOS}"
                }
            }
        }

        /* =============================== LINUX SETUP =============================== */

        stage('Linux Setup') {
            when {
                expression { detectedOS == 'linux' }
            }
            stages {

                stage('Check Docker on Linux') {
                    steps {
                        script {
                            echo "🐧 Linux: Checking Docker..."

                            def dockerInstalled = sh(
                                script: """
                                    sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                        if command -v docker >/dev/null; then echo installed; else echo not; fi
                                    '
                                """,
                                returnStdout: true
                            ).trim()

                            if (dockerInstalled == "installed") {
                                echo "✅ Docker already installed!"
                            } else {
                                echo "⚠️ Installing Docker..."
                                sh """
                                    sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                        sudo apt-get update
                                        sudo apt-get install -y docker.io
                                        sudo systemctl start docker
                                        sudo systemctl enable docker
                                    '
                                """
                            }
                        }
                    }
                }

                stage('Setup Docker Registry on Linux') {
                    steps {
                        script {
                            echo "🗄️ Setting up private registry..."

                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${REGISTRY_NAME} registry:2 || true
                                '
                            """
                        }
                    }
                }

            }
        }

        /* =============================== WINDOWS SETUP =============================== */

        stage('Windows Setup') {
            when {
                expression { detectedOS == 'windows' }
            }
            stages {

                stage('Check Docker on Windows') {
                    steps {
                        script {
                            echo "🪟 Windows: Checking Docker..."

                            def dockerInstalled = sh(
                                script: """
                                    sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "docker --version"
                                """,
                                returnStdout: true
                            ).trim()

                            if (!dockerInstalled) {
                                error("❌ Windows वर Docker Desktop manually install करा!")
                            }
                        }
                    }
                }

                stage('Setup Docker Registry on Windows') {
                    steps {
                        script {

                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "
                                    docker ps -a | findstr ${REGISTRY_NAME} || docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${REGISTRY_NAME} registry:2
                                "
                            """
                        }
                    }
                }

            }
        }

    }

    post {
        success {
            echo "🎉 SUCCESS: Setup Completed for ${params.REMOTE_IP}"
        }
        failure {
            echo "❌ FAILURE: कृपया logs तपासा"
        }
    }
}
