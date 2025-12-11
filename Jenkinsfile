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

        /* ============================= INPUT VALIDATION ============================= */

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


        /* ============================= DETECT OS ============================= */

        stage('Detect OS') {
            steps {
                script {
                    echo "🔍 Remote machine OS detect करत आहे..."

                    // Jenkins needs explicit def
                    def detected = "unknown"

                    /* ---------- Linux Detection ---------- */
                    try {
                        def osInfo = sh(
                            script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' \
                                ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} "uname -s"
                            """,
                            returnStdout: true
                        ).trim()

                        if (osInfo.contains("Linux")) {
                            detected = "linux"
                            echo "🐧 Linux detected!"
                        }
                    } catch (err) {
                        // ignore
                    }

                    /* ---------- Windows Detection ---------- */
                    if (detected == "unknown") {
                        try {
                            def winInfo = sh(
                                script: """
                                    sshpass -p '${params.REMOTE_PASSWORD}' \
                                    ssh -o StrictHostKeyChecking=no \
                                    ${params.REMOTE_USER}@${params.REMOTE_IP} \
                                    "powershell -Command \\"(Get-WmiObject Win32_OperatingSystem).Caption\\""
                                """,
                                returnStdout: true
                            ).trim()

                            if (winInfo) {
                                detected = "windows"
                                echo "🪟 Windows detected!"
                            }
                        } catch (err) {
                            detected = "windows"
                        }
                    }

                    echo "🎯 Final detected OS: ${detected}"
                    // Store in global env
                    env.detectedOS = detected
                }
            }
        }


        /* ============================= LINUX SETUP ============================= */

        stage('Transfer Compose (Linux)') {
            when { expression { env.detectedOS == 'linux' } }
            steps {
                script {
                    echo "📤 Linux: docker-compose.yml transfer करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        ssh ${params.REMOTE_USER}@${params.REMOTE_IP} 'mkdir -p ~/docker-deployment'

                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        scp docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:~/docker-deployment/docker-compose.yml
                    """
                }
            }
        }

        stage('Run Compose (Linux)') {
            when { expression { env.detectedOS == 'linux' } }
            steps {
                script {
                    echo "🚀 Linux remote machine वर containers deploy करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '
                            cd ~/docker-deployment
                            docker-compose down || true
                            docker-compose up -d
                            docker ps -a
                        '
                    """
                }
            }
        }



        /* ============================= WINDOWS SETUP ============================= */

        stage('Transfer Compose (Windows)') {
            when { expression { env.detectedOS == 'windows' } }
            steps {
                script {
                    echo "📤 Windows: docker-compose.yml transfer करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "mkdir C:\\docker-deployment 2>nul"

                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        scp docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:C:/docker-deployment/docker-compose.yml
                    """
                }
            }
        }

        stage('Run Compose (Windows)') {
            when { expression { env.detectedOS == 'windows' } }
            steps {
                script {
                    echo "🚀 Windows remote machine वर containers deploy करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "
                            cd C:\\docker-deployment
                            docker-compose down || echo no-old-containers
                            docker-compose up -d
                            docker ps -a
                        "
                    """
                }
            }
        }


        /* ============================= VERIFICATION ============================= */

        stage('Verify Containers Running') {
            steps {
                script {
                    echo "✔️ Remote machine वर containers verify करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '
                            echo "==== Docker Status ===="
                            docker ps -a
                        '
                    """
                }
            }
        }

    }

    post {
        success {
            echo "🎉 SUCCESS! Remote machine वर containers चालू झाले!"
        }
        failure {
            echo "❌ FAILURE! कृपया logs तपासा."
        }
    }
}
