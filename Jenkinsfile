pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', description: 'Remote machine IP')
        string(name: 'REMOTE_USER', defaultValue: 'administrator', description: 'Remote username')
        password(name: 'REMOTE_PASSWORD', description: 'Remote password')
    }

    environment {
        REGISTRY_PORT = "5000"
        REGISTRY_NAME = "private-registry"
        POSTGRES_SRC = "postgres:latest"
        REDIS_SRC = "redis:latest"
        POSTGRES_LOCAL = "localhost:5000/postgres:latest"
        REDIS_LOCAL = "localhost:5000/redis:latest"
    }

    stages {

        /* ---------------------------------------------------
                        Detect Remote OS
        --------------------------------------------------- */
        stage('Detect OS') {
            steps {
                script {
                    detectedOS = "unknown"

                    // Linux check
                    try {
                        def out = sh(script: """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=3 \
                            ${params.REMOTE_USER}@${params.REMOTE_IP} uname -s
                        """, returnStdout: true).trim()

                        if (out.contains("Linux")) detectedOS = "linux"
                    } catch (e) {}

                    // Windows check
                    if (detectedOS == "unknown") {
                        try {
                            def winOut = sh(script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} "powershell -Command \\"(Get-WmiObject Win32_OperatingSystem).Caption\\""
                            """, returnStdout: true).trim()

                            if (winOut) detectedOS = "windows"
                        } catch (e) {
                            detectedOS = "windows"
                        }
                    }

                    echo "🎯 Detected Remote OS: ${detectedOS}"
                }
            }
        }

        /* ---------------------------------------------------
                Step 1: Build & Push Images to Registry
        --------------------------------------------------- */
        stage('Prepare Local Registry Images') {
            steps {
                script {
                    echo "📦 Pulling & Tagging Images..."

                    sh """
                        docker pull ${POSTGRES_SRC}
                        docker pull ${REDIS_SRC}

                        docker tag ${POSTGRES_SRC} ${POSTGRES_LOCAL}
                        docker tag ${REDIS_SRC} ${REDIS_LOCAL}
                    """

                    echo "🚀 Pushing images to local registry..."
                    sh """
                        docker push ${POSTGRES_LOCAL}
                        docker push ${REDIS_LOCAL}
                    """
                }
            }
        }

        /* ---------------------------------------------------
                Step 2: Copy Compose File
        --------------------------------------------------- */
        stage('Copy docker-compose.yml') {
            steps {
                script {
                    if (detectedOS == "linux") {

                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} 'mkdir -p ~/deploy'
                                
                            sshpass -p '${params.REMOTE_PASSWORD}' scp -o StrictHostKeyChecking=no \
                                docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:~/deploy/docker-compose.yml
                        """

                    } else {

                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \
                            "powershell -Command \\"New-Item -ItemType Directory -Force -Path 'C:\\\\deploy' | Out-Null\\""

                            sshpass -p '${params.REMOTE_PASSWORD}' scp docker-compose.yml \
                            ${params.REMOTE_USER}@${params.REMOTE_IP}:C:/deploy/docker-compose.yml
                        """

                    }
                }
            }
        }

        /* ---------------------------------------------------
                Step 3: Run docker-compose on Linux
        --------------------------------------------------- */
        stage('Run Compose - Linux') {
            when { expression { detectedOS == "linux" } }
            steps {
                script {
                    echo "🐧 Running compose on Linux..."

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

        /* ---------------------------------------------------
                Step 3: Run docker-compose on Windows
        --------------------------------------------------- */
        stage('Run Compose - Windows') {
            when { expression { detectedOS == "windows" } }
            steps {
                script {
                    echo "🪟 Running compose on Windows..."

                    sh """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \
                        "powershell -Command \\
                        \\"Set-Location 'C:\\\\deploy'; \\
                        docker-compose down -v; \\
                        docker-compose up -d; \\
                        docker ps -a;\\""
                    """
                }
            }
        }

        /* ---------------------------------------------------
                Verify
        --------------------------------------------------- */
        stage('Verify Containers') {
            steps {
                script {
                    echo "🧐 Checking containers on remote machine..."

                    if (detectedOS == "linux") {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} docker ps -a
                        """
                    } else {
                        sh """
                            sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} \
                            "powershell -Command \\"docker ps -a\\""
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS! PostgreSQL + Redis + Registry remote machine वर चालू आहेत!"
        }
        failure {
            echo "❌ FAILED! Logs तपासा."
        }
    }
}
