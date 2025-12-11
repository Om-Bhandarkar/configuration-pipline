pipeline {
    agent any
    
    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP address')
        string(name: 'REMOTE_USER', defaultValue: 'jtsm', description: 'Remote machine username')
        password(name: 'REMOTE_PASSWORD', defaultValue: '', description: 'Remote machine password')
        string(name: 'REGISTRY_PORT', defaultValue: '5000', description: 'Docker registry port')
    }

    environment {
        REGISTRY_NAME = "local-registry"
        POSTGRES_IMAGE = "postgres:15"
        REDIS_IMAGE = "redis:7-alpine"
    }

    stages {

        /* ------------------------ INPUT VALIDATION ------------------------ */

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


        /* ------------------------ OS DETECTION ------------------------ */

        stage('Detect OS') {
            steps {
                script {
                    echo "🔍 Remote machine ची OS detect करत आहे..."

                    def detected = "unknown"

                    // Linux detection
                    try {
                        def osInfo = sh(
                            script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' \
                                ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} "uname -s"
                            """,
                            returnStdout: true
                        ).trim()

                        if (osInfo.contains("Linux")) {
                            detected = "linux"
                            echo "🐧 Linux detected!"
                        }
                    } catch(e) {}

                    // Windows detection
                    if (detected == "unknown") {
                        try {
                            def winInfo = sh(
                                script: """
                                    sshpass -p '${params.REMOTE_PASSWORD}' \
                                    ssh -o StrictHostKeyChecking=no \
                                    ${params.REMOTE_USER}@${params.REMOTE_IP} "powershell -Command \\"(Get-WmiObject Win32_OperatingSystem).Caption\\"" 
                                """,
                                returnStdout: true
                            ).trim()

                            if (winInfo) {
                                detected = "windows"
                                echo "🪟 Windows detected!"
                            }
                        } catch(e) {
                            detected = "windows"
                        }
                    }

                    echo "🎯 Final detected OS: ${detected}"
                    env.detectedOS = detected
                }
            }
        }


        /* ------------------------ LINUX SETUP ------------------------ */

        stage('Linux Setup') {
            when { expression { env.detectedOS == 'linux' } }
            steps {
                script {
                    echo "📤 Linux: docker-compose.yml transfer करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} 'mkdir -p ~/docker-deployment'
                        sshpass -p '${params.REMOTE_PASSWORD}' scp docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:~/docker-deployment/docker-compose.yml
                    """
                }
            }
        }


        /* ------------------------ WINDOWS SETUP ------------------------ */

        stage('Windows Setup') {
            when { expression { env.detectedOS == 'windows' } }
            steps {
                script {
                    echo "📤 Windows: docker-compose.yml transfer करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "mkdir C:\\\\docker-deployment 2>nul"

                        sshpass -p '${params.REMOTE_PASSWORD}' \
                        scp docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:C:/docker-deployment/docker-compose.yml
                    """
                }
            }
        }

    }

    post {
        success {
            echo "🎉 SUCCESS! Setup Completed!"
        }
        failure {
            echo "❌ Pipeline Failed — कृपया logs तपासा."
        }
    }
}
