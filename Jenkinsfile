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
                    if (!params.REMOTE_IP) error("❌ Remote IP missing!")
                    echo "➡ Remote: ${params.REMOTE_IP}"
                }
            }
        }

        /* -------------------------------------------------------
         * 2️⃣ DETECT OS
         * -------------------------------------------------------*/
        stage("Detect OS") {
            steps {
                script {
                    def out = sh(script: """
                        sshpass -p '${params.REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=5 ${params.REMOTE_USER}@${params.REMOTE_IP} "uname -s" 2>/dev/null || echo FAILED
                    """, returnStdout:true).trim()

                    env.DETECTED_OS = out.contains("Linux") ? "linux" : "windows"
                    echo "🎯 Detected OS: ${env.DETECTED_OS}"
                }
            }
        }

        /* -------------------------------------------------------
         * 3️⃣ INSTALL DOCKER & COMPOSE (LINUX)
         * -------------------------------------------------------*/
        stage("Linux Setup") {
            when { expression { env.DETECTED_OS == "linux" } }
            steps {
                script {
                    echo "🔧 Ensuring Docker + Compose on Linux..."

                    shOrBatRemote("""
                        if ! command -v docker >/dev/null; then
                            sudo apt-get update -y &&
                            sudo apt-get install -y docker.io &&
                            sudo systemctl enable docker &&
                            sudo systemctl start docker
                        fi

                        if ! command -v docker-compose >/dev/null; then
                            sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-\\$(uname -s)-\\$(uname -m)" \
                            -o /usr/local/bin/docker-compose &&
                            sudo chmod +x /usr/local/bin/docker-compose
                        fi
                    """)

                    echo "✅ Linux ready!"
                }
            }
        }

        /* -------------------------------------------------------
         * 4️⃣ WINDOWS DOCKER CHECK
         * -------------------------------------------------------*/
        stage("Windows Setup") {
            when { expression { env.DETECTED_OS == "windows" } }
            steps {
                script {
                    echo "🔍 Checking Docker on Windows..."

                    def installed = bat(script: """
                        @echo off
                        sshpass -p ${params.REMOTE_PASSWORD} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "docker --version" >nul 2>&1 && echo yes || echo no
                    """, returnStdout:true).trim()

                    if (installed == "no") error("❌ Install Docker Desktop manually!")
                    echo "✅ Docker Desktop found!"
                }
            }
        }

        /* -------------------------------------------------------
         * 5️⃣ UPLOAD COMPOSE FILE
         * -------------------------------------------------------*/
        stage("Upload docker-compose.yml") {
            steps {
                script {
                    echo "📤 Uploading compose file..."

                    shOrBat("""
                        sshpass -p '${params.REMOTE_PASSWORD}' scp -o StrictHostKeyChecking=no \
                        docker-compose.yml \
                        ${params.REMOTE_USER}@${params.REMOTE_IP}:/tmp/docker-compose.yml
                    """)

                    shOrBatRemote("""
                        mkdir -p ~/docker-deployment &&
                        mv /tmp/docker-compose.yml ~/docker-deployment/docker-compose.yml
                    """)

                    echo "✅ Uploaded!"
                }
            }
        }

        /* -------------------------------------------------------
         * 6️⃣ START REGISTRY FIRST
         * -------------------------------------------------------*/
        stage("Start Registry") {
            steps {
                script {
                    echo "🗄️ Starting registry first..."

                    shOrBatRemote("""
                        cd ~/docker-deployment
                        docker-compose up -d registry
                    """)

                    sleep 5

                    echo "🔍 Registry health check..."

                    retry(5) {
                        shOrBatRemote("""
                            curl -s http://localhost:${params.REGISTRY_PORT}/v2/_catalog || exit 1
                        """)
                    }

                    echo "✅ Registry healthy!"
                }
            }
        }

        /* -------------------------------------------------------
         * 7️⃣ PUSH IMAGES TO REGISTRY
         * -------------------------------------------------------*/
        stage("Push Images") {
            steps {
                script {
                    echo "📦 Pull → Tag → Push images..."

                    shOrBatRemote("""
                        docker pull ${POSTGRES_IMAGE}
                        docker pull ${REDIS_IMAGE}

                        docker tag ${POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:latest
                        docker tag ${REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:latest

                        docker push localhost:${params.REGISTRY_PORT}/postgres:latest
                        docker push localhost:${params.REGISTRY_PORT}/redis:latest
                    """)

                    echo "✅ Images available in private registry!"
                }
            }
        }

        /* -------------------------------------------------------
         * 8️⃣ START POSTGRES & REDIS
         * -------------------------------------------------------*/
        stage("Start Infra Services") {
            steps {
                script {
                    echo "🚀 Starting PostgreSQL + Redis..."

                    shOrBatRemote("""
                        cd ~/docker-deployment
                        docker-compose up -d postgres redis
                    """)

                    echo "⏳ Checking PostgreSQL health..."

                    retry(10) {
                        shOrBatRemote("docker exec infra-postgres pg_isready -U postgres || exit 1")
                        sleep 2
                    }

                    echo "⏳ Checking Redis health..."

                    retry(10) {
                        shOrBatRemote("docker exec infra-redis redis-cli ping | grep PONG || exit 1")
                        sleep 2
                    }

                    echo "✅ All services healthy!"
                }
            }
        }
    }

    post {
        success {
            echo """
🎉 DEPLOY SUCCESSFUL!

PostgreSQL → ${params.REMOTE_IP}:5432  
Redis      → ${params.REMOTE_IP}:6379  
Registry   → ${params.REMOTE_IP}:${params.REGISTRY_PORT}

All health checks passed ✔✔✔
"""
        }
        failure {
            echo "❌ Deployment Failed! Check logs."
        }
    }
}

/* -------------------------------------------------------
   UTIL FUNCTIONS (CLEAN & OPTIMIZED)
--------------------------------------------------------*/

def shOrBatRemote(cmd) {
    if (env.DETECTED_OS == "linux") {
        sh """sshpass -p '${params.REMOTE_PASSWORD}' \
        ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '${cmd}'"""
    } else {
        bat """sshpass -p ${params.REMOTE_PASSWORD} \
        ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "${cmd}" """
    }
}

def shOrBat(cmd) {
    if (env.DETECTED_OS == "linux") sh(cmd) else bat(cmd)
}
