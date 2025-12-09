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
    }

    stages {

        /* ----------------------------------------------------
           1) CHECK SSH CONNECTION
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
           2) DETECT OS
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
                    } else {
                        error "❌ Only Linux supported right now!"
                    }
                }
            }
        }

        /* ----------------------------------------------------
           3) INSTALL DOCKER + COMPOSE (LINUX)
        ---------------------------------------------------- */
        stage("Install Docker & Docker Compose") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🐳 Installing Docker & Compose if missing..."

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
version: "3.8"

services:
  postgres:
    container_name: infra-postgres
    image: postgres:latest
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - "5432:5432"
    restart: unless-stopped

  redis:
    container_name: infra-redis
    image: redis:latest
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
           5) UPLOAD COMPOSE + DEPLOY
        ---------------------------------------------------- */
        stage("Upload & Deploy") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🚀 Deploying Compose to Linux..."

                    sh """
                        # create remote dir
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${LINUX_DIR}"

                        # upload compose file
                        sshpass -p "${params.SSH_PASS}" scp -o StrictHostKeyChecking=no \
                        docker-compose.yml \
                        ${params.SSH_USER}@${params.TARGET_IP}:${LINUX_COMPOSE}

                        # run compose
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "
                            cd ${LINUX_DIR}
                            docker-compose pull
                            docker-compose up -d
                        "
                    """
                }
            }
        }

        /* ----------------------------------------------------
           6) VERIFY SERVICES
        ---------------------------------------------------- */
        stage("Verify Postgres & Redis Containers") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🔎 Checking Postgres & Redis..."

                    sh """
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "
                            echo '--- Running Containers ---'
                            docker ps --format 'table {{.Names}}\\t{{.Status}}\\t{{.Ports}}'

                            echo '\\nChecking infra-postgres...'
                            docker ps | grep infra-postgres && echo '✅ Postgres Running' || echo '❌ Postgres NOT Running!'

                            echo '\\nChecking infra-redis...'
                            docker ps | grep infra-redis && echo '✅ Redis Running' || echo '❌ Redis NOT Running!'
                        "
                    """
                }
            }
        }
    }

    post {
        success { echo "🎉 Deployment Successful! Postgres & Redis are running." }
        failure { echo "❌ Deployment Failed. Check logs above." }
        always { sh "rm -f docker-compose.yml" }
    }
}
