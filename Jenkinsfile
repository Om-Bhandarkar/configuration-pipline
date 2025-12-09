pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Target Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE = ""
        LINUX_DIR = "/infra"
        LINUX_COMPOSE = "/infra/docker-compose.yml"

        WIN_DIR = "C:/infra"
        WIN_COMPOSE = "C:/infra/docker-compose.yml"
        
        REMOTE_PATH='export PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin'
    }

    stages {

        /* ----------------------------------------------------
           1) CHECK CONNECTIVITY
        ---------------------------------------------------- */
        stage("Check SSH Connection") {
            steps {
                script {
                    echo "🔗 Testing SSH connection..."

                    sh """
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 \
                        ${params.SSH_USER}@${params.TARGET_IP} "echo 'SSH OK'"
                    """
                }
            }
        }

        /* ----------------------------------------------------
           2) OS DETECTION
        ---------------------------------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting remote OS..."

                    def osName = sh(
                        returnStdout: true,
                        script: """
                            sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                            ${params.SSH_USER}@${params.TARGET_IP} "uname 2>/dev/null || echo WINDOWS"
                        """
                    ).trim().toLowerCase()

                    if (osName.contains("linux")) {
                        env.OS_TYPE = "linux"
                        echo "✅ Remote OS: Linux"
                    } 
                    else if (osName.contains("windows")) {
                        env.OS_TYPE = "windows"
                        echo "✅ Remote OS: Windows"
                    }
                    else {
                        error "❌ Unable to detect OS. Response: ${osName}"
                    }
                }
            }
        }

        /* ----------------------------------------------------
           3) INSTALL DOCKER + COMPOSE (LINUX ONLY)
        ---------------------------------------------------- */
        stage("Install Docker & Docker Compose (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🐳 Ensuring Docker & Docker Compose exist..."

                    sh """
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} '
                            set -e

                            if ! command -v docker >/dev/null 2>&1; then
                                echo "📦 Installing Docker..."
                                apt-get update -y || yum update -y
                                if command -v apt-get >/dev/null 2>&1; then
                                    apt-get install -y docker.io
                                else
                                    yum install -y docker
                                fi
                            fi

                            systemctl start docker || service docker start || true

                            echo "Docker Version: \$(docker --version)"

                            # Install compose v2
                            if ! command -v docker-compose >/dev/null 2>&1; then
                                echo "📦 Installing Docker Compose..."
                                curl -L "https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-linux-x86_64" \
                                    -o /usr/local/bin/docker-compose
                                chmod +x /usr/local/bin/docker-compose
                            fi

                            echo "Compose Version: \$(docker-compose --version)"
                        '
                    """
                }
            }
        }

        /* ----------------------------------------------------
           4) CREATE DOCKER-COMPOSE.YML LOCALLY
        ---------------------------------------------------- */
        stage("Generate Compose File") {
            steps {
                script {
                    def compose = """
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass redis123
    ports:
      - "6379:6379"
    restart: unless-stopped
"""

                    writeFile file: "docker-compose.yml", text: compose
                    echo "✅ docker-compose.yml created"
                }
            }
        }

        /* ----------------------------------------------------
           5) UPLOAD COMPOSE + DEPLOY (LINUX)
        ---------------------------------------------------- */
        stage("Upload & Deploy (Linux)") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🚀 Deploying to Linux..."

                    sh """
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${LINUX_DIR}"

                        sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no \
                        docker-compose.yml \
                        ${params.SSH_USER}@${params.TARGET_IP}:${LINUX_COMPOSE}

                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "
                            ${REMOTE_PATH}
                            cd ${LINUX_DIR}
                            echo '📥 Pulling images...'
                            docker-compose pull

                            echo '🚀 Starting containers...'
                            docker-compose up -d || { echo '❌ Compose failed'; exit 1; }

                            echo '📊 Running Containers:'
                            docker ps
                        "
                    """
                }
            }
        }

        /* ----------------------------------------------------
           6) VERIFY SERVICES
        ---------------------------------------------------- */
        stage("Verify Running Containers") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🔎 Checking service status..."

                    sh """
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "
                            docker ps --format 'table {{.Names}}\\t{{.Status}}\\t{{.Ports}}'
                        "
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Deployment Success! Containers are LIVE on remote machine."
        }
        failure {
            echo "❌ Deployment Failed. Check logs above."
        }
        always {
            sh "rm -f docker-compose.yml"
        }
    }
}
