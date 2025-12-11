pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP')
        string(name: 'REMOTE_USER', defaultValue: 'administrator', description: 'Remote username')
        password(name: 'REMOTE_PASSWORD', defaultValue: '', description: 'Remote password')
        string(name: 'REGISTRY_PORT', defaultValue: '5000', description: 'Private registry port')
    }

    environment {
        DETECTED_OS = ""
        REGISTRY_NAME = "local-registry"
        POSTGRES_IMAGE = "postgres:latest"
        REDIS_IMAGE = "redis:latest"
    }

    stages {

        /* -------------------------------------------------------
         * 1️⃣ INPUT VALIDATION
         * -------------------------------------------------------*/
        stage("Input Validation") {
            steps {
                script {
                    if (!params.REMOTE_IP) 
                        error("❌ Remote IP आवश्यक आहे!")

                    echo "✅ Using Remote IP: ${params.REMOTE_IP}"
                }
            }
        }

        /* -------------------------------------------------------
         * 2️⃣ DETECT OS
         * -------------------------------------------------------*/
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting remote OS..."

                    try {
                        def osCheck = sh(script: """
                            sshpass -p '${params.REMOTE_PASSWORD}' \
                            ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 \
                            ${params.REMOTE_USER}@${params.REMOTE_IP} "uname -s" 2>/dev/null || echo FAILED
                        """, returnStdout: true).trim()

                        env.DETECTED_OS = osCheck.contains("Linux") ? "linux" : "windows"
                    } catch (e) {
                        env.DETECTED_OS = "windows"
                    }

                    echo "🎯 Detected OS: ${env.DETECTED_OS}"
                }
            }
        }

        /* -------------------------------------------------------
         * 3️⃣ LINUX SETUP
         * -------------------------------------------------------*/
        stage("Linux Setup") {
            when { expression { env.DETECTED_OS == "linux" } }
            stages {

                stage("Install Docker (Linux)") {
                    steps {
                        script {
                            def installed = sh(script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} "command -v docker >/dev/null && echo yes || echo no"
                            """, returnStdout: true).trim()

                            if (installed == "no") {
                                echo "⚠️ Installing Docker on Linux..."
                                sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    sudo apt-get update
                                    sudo apt-get install -y docker.io
                                    sudo systemctl enable docker
                                    sudo systemctl start docker
                                '
                                """
                            }
                            echo "✅ Docker ready on Linux!"
                        }
                    }
                }

                stage("Install Docker Compose (Linux)") {
                    steps {
                        script {
                            def installed = sh(script: """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} "command -v docker-compose >/dev/null && echo yes || echo no"
                            """, returnStdout: true).trim()

                            if (installed == "no") {
                                echo "⚠️ Installing Docker Compose..."
                                sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                    sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-\\$(uname -s)-\\$(uname -m)" \
                                    -o /usr/local/bin/docker-compose
                                    sudo chmod +x /usr/local/bin/docker-compose
                                '
                                """
                            }
                            echo "✅ Docker Compose ready!"
                        }
                    }
                }

                stage("Upload docker-compose.yml") {
                    steps {
                        script {
                            echo "📤 Uploading compose file to Linux..."
                            sh """
                                sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no \
                                ${params.REMOTE_USER}@${params.REMOTE_IP} 'mkdir -p ~/docker-deployment'

                                sshpass -p '${params.REMOTE_PASSWORD}' scp \
                                -o StrictHostKeyChecking=no \
                                docker-compose.yml \
                                ${params.REMOTE_USER}@${params.REMOTE_IP}:~/docker-deployment/docker-compose.yml
                            """

                            echo "✅ Uploaded docker-compose.yml to Linux."
                        }
                    }
                }
            }
        }

        /* -------------------------------------------------------
         * 4️⃣ WINDOWS SETUP
         * -------------------------------------------------------*/
        stage("Windows Setup") {
            when { expression { env.DETECTED_OS == "windows" } }
            stages {

                stage("Check Docker Desktop") {
                    steps {
                        script {
                            echo "🔎 Checking Docker on Windows..."

                            def installed = bat(script: """
                                @echo off
                                sshpass -p ${params.REMOTE_PASSWORD} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "docker --version" >nul 2>&1 && echo yes || echo no
                            """, returnStdout: true).trim()

                            if (installed == "no") {
                                error("❌ Docker Desktop Windows machine वर manually install करावे लागेल!")
                            }
                            echo "✅ Docker available on Windows."
                        }
                    }
                }

                stage("Upload docker-compose.yml (Windows)") {
                    steps {
                        script {
                            echo "📤 Uploading compose file to Windows..."

                            bat """
                                sshpass -p ${params.REMOTE_PASSWORD} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "mkdir C:/docker-deployment" 2>nul

                                sshpass -p ${params.REMOTE_PASSWORD} scp docker-compose.yml \
                                ${params.REMOTE_USER}@${params.REMOTE_IP}:/c:/docker-deployment/docker-compose.yml
                            """

                            echo "✅ docker-compose.yml uploaded to Windows!"
                        }
                    }
                }
            }
        }

        /* -------------------------------------------------------
         * 5️⃣ COMMON STAGES (Both OS)
         * -------------------------------------------------------*/
        stage("Setup Docker Registry") {
            steps {
                script {
                    echo "🔧 Setting up private registry..."

                    def cmd = env.DETECTED_OS == "linux" ?
                        "docker ps -a | grep -q ${REGISTRY_NAME} || docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${REGISTRY_NAME} registry:2"
                        :
                        "docker ps -a | findstr ${REGISTRY_NAME} || docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${REGISTRY_NAME} registry:2"

                    shOrBat(cmd)
                    echo "✅ Registry ready."
                }
            }
        }

        stage("Pull, Tag & Push Images") {
            steps {
                script {
                    echo "📦 Handling images..."

                    def cmd = """
                        docker pull ${POSTGRES_IMAGE}
                        docker pull ${REDIS_IMAGE}
                        docker tag ${POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:latest
                        docker tag ${REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:latest
                        docker push localhost:${params.REGISTRY_PORT}/postgres:latest
                        docker push localhost:${params.REGISTRY_PORT}/redis:latest
                    """

                    shOrBatRemote(cmd)
                    echo "✅ Images pushed to private registry."
                }
            }
        }

        stage("Deploy Stack") {
            steps {
                script {
                    echo "🚀 Deploying containers..."

                    def deploy = """
                        cd ~/docker-deployment || cd C:/docker-deployment
                        docker-compose down || true
                        docker-compose up -d
                        docker-compose ps
                    """

                    shOrBatRemote(deploy)
                }
            }
        }
    }

    post {
        success {
            echo """
🎉 SUCCESS! Environment deployed successfully.

PostgreSQL → ${params.REMOTE_IP}:5432  
Redis      → ${params.REMOTE_IP}:6379  
Registry   → ${params.REMOTE_IP}:${params.REGISTRY_PORT}
"""
        }
        failure {
            echo "❌ Deployment Failed! Logs तपासा."
        }
    }
}

/* -------------------------------------------------------
   UTIL FUNCTIONS to reduce duplicate code
--------------------------------------------------------*/

def shOrBatRemote(cmd) {
    if (env.DETECTED_OS == "linux") {
        sh """sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '${cmd}'"""
    } else {
        bat """sshpass -p ${params.REMOTE_PASSWORD} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "${cmd}" """
    }
}

def shOrBat(cmd) {
    if (env.DETECTED_OS == "linux") {
        sh """sshpass -p '${params.REMOTE_PASSWORD}' ssh ${params.REMOTE_USER}@${params.REMOTE_IP} '${cmd}'"""
    } else {
        bat """sshpass -p ${params.REMOTE_PASSWORD} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "${cmd}" """
    }
}
