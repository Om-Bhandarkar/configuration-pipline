pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', description: 'Remote machine IP')
        string(name: 'REMOTE_USER', defaultValue: 'administrator', description: 'Remote username')
        password(name: 'REMOTE_PASSWORD', description: 'Remote password')
    }

    stages {

        /* Detect OS ------------------------------------------------------------- */
        stage('Detect OS') {
            steps {
                script {
                    detectedOS = "unknown"

                    try {
                        def out = sh(script: """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=3 \
                            ${params.REMOTE_USER}@${params.REMOTE_IP} uname -s
                        """, returnStdout: true).trim()

                        if (out.contains("Linux")) detectedOS = "linux"
                    } catch(e) {}

                    if (detectedOS == "unknown") {
                        try {
                            def winOut = sh(script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} "powershell -Command \\"(Get-WmiObject Win32_OperatingSystem).Caption\\""
                            """, returnStdout: true).trim()

                            if (winOut) detectedOS = "windows"
                        } catch(e) {
                            detectedOS = "windows"
                        }
                    }

                    echo "🎯 Detected OS: ${detectedOS}"
                }
            }
        }

        /* Copy docker-compose.yml ------------------------------------------------------------- */
        stage('Copy Compose File') {
            steps {
                script {
                    if (detectedOS == "linux") {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} 'mkdir -p ~/deploy'
                            sshpass -p '${params.REMOTE_PASSWORD}' scp docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:~/deploy/docker-compose.yml
                        """
                    } else {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \
                            "powershell -Command \\\\"New-Item -ItemType Directory -Force -Path 'C:\\\\deploy' | Out-Null\\\\""

                            sshpass -p '${params.REMOTE_PASSWORD}' scp docker-compose.yml \
                            ${params.REMOTE_USER}@${params.REMOTE_IP}:C:/deploy/docker-compose.yml
                        """
                    }
                }
            }
        }

        /* Run docker-compose (Linux) ------------------------------------------------------------- */
        stage('Run Compose Linux') {
            when { expression { detectedOS == "linux" } }
            steps {
                script {
                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '
                            cd ~/deploy;
                            docker-compose down || true;
                            docker-compose up -d;
                            docker ps -a;
                        '
                    """
                }
            }
        }

        /* Run docker-compose (Windows) ------------------------------------------------------------- */
        stage('Run Compose Windows') {
            when { expression { detectedOS == "windows" } }
            steps {
                script {
                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \
                        "powershell -Command \\\\"
                        Set-Location 'C:\\\\deploy';
                        docker-compose down -v;
                        docker-compose up -d;
                        docker ps -a;
                        \\\\\""
                    """
                }
            }
        }

        /* Verify ------------------------------------------------------------- */
        stage('Verify Containers') {
            steps {
                script {
                    if (detectedOS == "linux") {
                        sh "sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} docker ps -a"
                    } else {
                        sh """sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \
                        "powershell -Command \\\\"docker ps -a\\\\""
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS! PostgreSQL + Redis + Registry remote वर चालू आहेत!"
        }
        failure {
            echo "❌ Failed. Logs तपासा."
        }
    }
}
