pipeline {
    agent any
    
    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP address')
        string(name: 'REMOTE_USER', defaultValue: 'administrator', description: 'Remote machine username')
        password(name: 'REMOTE_PASSWORD', defaultValue: '', description: 'Remote machine password')
    }

    stages {

        /* ------------------------------ VALIDATE INPUT ------------------------------ */
        stage('Input Validation') {
            steps {
                script {
                    if (!params.REMOTE_IP?.trim()) {
                        error("❌ Remote IP address आवश्यक आहे!")
                    }
                    echo "✅ Remote IP: ${params.REMOTE_IP}"
                }
            }
        }

        /* ------------------------------ DETECT OS ------------------------------ */
        stage('Detect OS') {
            steps {
                script {
                    echo "🔍 Remote machine OS detect करत आहे..."

                    detectedOS = "unknown"

                    // Linux check
                    try {
                        def outLinux = sh(
                            script: """sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=3 ${params.REMOTE_USER}@${params.REMOTE_IP} 'uname -s'""",
                            returnStdout: true
                        ).trim()

                        if (outLinux.contains("Linux")) {
                            detectedOS = "linux"
                        }
                    } catch(e) {}

                    // Windows check (PowerShell)
                    if (detectedOS == "unknown") {
                        try {
                            def outWin = sh(
                                script: """sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "powershell -Command \\"(Get-WmiObject Win32_OperatingSystem).Caption\\"" """,
                                returnStdout: true
                            ).trim()

                            if (outWin) detectedOS = "windows"
                        } catch(e) {
                            detectedOS = "windows"  // fallback
                        }
                    }

                    echo "🎯 OS Detected: ${detectedOS}"
                }
            }
        }

        /* ------------------------------ LINUX DEPLOY ------------------------------ */

        stage('Copy Compose to Linux') {
            when { expression { detectedOS == 'linux' } }
            steps {
                script {
                    echo "📤 Linux: docker-compose फाइल copy करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} 'mkdir -p ~/docker-deployment'
                        sshpass -p '${params.REMOTE_PASSWORD}' scp docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:~/docker-deployment/docker-compose.yml
                    """
                }
            }
        }

        stage('Run Compose on Linux') {
            when { expression { detectedOS == 'linux' } }
            steps {
                script {
                    echo "🚀 Linux वर docker-compose चालवत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "
                            cd ~/docker-deployment;
                            docker-compose down || true;
                            docker-compose up -d;
                            docker ps -a;
                        "
                    """
                }
            }
        }

        /* ------------------------------ WINDOWS DEPLOY ------------------------------ */

        stage('Copy Compose to Windows') {
            when { expression { detectedOS == 'windows' } }
            steps {
                script {
                    echo "📤 Windows: docker-compose फाइल copy करत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \\
                        "powershell -Command \\"New-Item -ItemType Directory -Force -Path 'C:\\\\docker-deployment' | Out-Null\\""

                        sshpass -p '${params.REMOTE_PASSWORD}' scp docker-compose.yml \\
                        ${params.REMOTE_USER}@${params.REMOTE_IP}:C:/docker-deployment/docker-compose.yml
                    """
                }
            }
        }

        stage('Run Compose on Windows') {
            when { expression { detectedOS == 'windows' } }
            steps {
                script {
                    echo "🚀 Windows वर docker-compose चालवत आहे..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \\
                        "powershell -Command \\
                        \\"Set-Location 'C:\\\\docker-deployment'; \\
                        docker-compose down -v; \\
                        docker-compose up -d; \\
                        docker ps -a;\\""
                    """
                }
            }
        }

        /* ------------------------------ VERIFY ------------------------------ */

        stage('Verify Containers Running') {
            steps {
                script {
                    echo "✔️ Remote machine वर containers verify करत आहे..."

                    if (detectedOS == "linux") {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \\
                            'docker ps -a'
                        """
                    } else {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \\
                            "powershell -Command \\"docker ps -a\\""
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS! Remote machine वर containers READY आहेत!"
        }
        failure {
            echo "❌ Pipeline FAILED! Logs तपासा."
        }
    }
}
