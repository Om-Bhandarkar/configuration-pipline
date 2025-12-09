pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Target Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE = ""
        PATH_FIX = "export PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin;"
        COMPOSE_DIR_LINUX  = "/infra"
        COMPOSE_FILE_LINUX = "/infra/docker-compose.yml"
        COMPOSE_DIR_WIN    = "C:/infra"
        COMPOSE_FILE_WIN   = "C:/infra/docker-compose.yml"
    }

    stages {

        /* -------------------------
           1) CHECK CONNECTION
        ------------------------- */
        stage("Check Connection") {
            steps {
                script {
                    echo "🔗 Connecting to remote system: ${params.TARGET_IP}"
                    sh """
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 \
                        ${params.SSH_USER}@${params.TARGET_IP} "echo '✅ connection OK'"
                    """
                }
            }
        }

        /* -------------------------
           2) DETECT OS
        ------------------------- */
        stage("Detect OS") {
            steps {
                script {
                    echo "🔍 Detecting remote OS..."

                    def os = sh(returnStdout: true, script: """
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "uname -s || echo UNKNOWN"
                    """).trim().toLowerCase()

                    if (os.contains("linux")) {
                        env.OS_TYPE = "linux"
                        echo "✅ Linux detected"
                    } else {
                        env.OS_TYPE = "windows"
                        echo "⚠ Windows detected"
                    }
                }
            }
        }

        /* -------------------------
           3) INSTALL DOCKER & COMPOSE
        ------------------------- */
        stage("Install Docker & Compose") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🐳 Installing Docker & Docker Compose on Linux..."

                    sh """
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                            ${PATH_FIX}

                            if ! command -v docker >/dev/null 2>&1; then
                                echo "📦 Installing Docker..."

                                if command -v apt-get >/dev/null 2>&1; then
                                    apt-get update -y
                                    apt-get install -y docker.io
                                elif command -v yum >/dev/null 2>&1; then
                                    yum install -y docker
                                fi

                                systemctl enable docker
                                systemctl start docker
                            fi

                            echo "Docker: \$(docker --version)"

                            # Install Compose v2
                            if ! command -v docker-compose >/dev/null 2>&1; then
                                echo "📦 Installing Docker Compose v2..."
                                curl -sSL https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-linux-x86_64 \
                                    -o /usr/local/bin/docker-compose
                                chmod +x /usr/local/bin/docker-compose
                                ln -sf /usr/local/bin/docker-compose /usr/bin/docker-compose
                            fi

                            echo "Compose: \$(docker-compose --version)"
                        '
                    """
                }
            }
        }

        /* -------------------------
           4) CREATE DOCKER COMPOSE FILE
        ------------------------- */
        stage("Create Compose File") {
            steps {
                script {
                    echo "📄 Creating docker-compose.yml..."

                    def yml = """version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: always

  redis:
    image: redis:7-alpine
    container_name: redis_cache
    command: redis-server --requirepass redis123
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: always

volumes:
  postgres_data:
  redis_data:
"""

                    writeFile file: "docker-compose.yml", text: yml
                    echo "✅ docker-compose.yml created."
                }
            }
        }

        /* -------------------------
           5) DEPLOY STACK
        ------------------------- */
        stage("Upload & Deploy Services") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🚀 Deploying services on Linux..."

                    // Upload compose file
                    sh """
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "mkdir -p ${COMPOSE_DIR_LINUX}/postgres"

                        sshpass -p "${params.SSH_PASS}" \
                        scp -o StrictHostKeyChecking=no docker-compose.yml \
                        ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_DIR_LINUX}/docker-compose.yml
                    """

                    // Upload init.sql
                    sh """
                        sshpass -p "${params.SSH_PASS}" ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} \
                        "echo 'CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";' > ${COMPOSE_DIR_LINUX}/postgres/init.sql"
                    """

                    // RUN Docker Compose
                    sh """
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} "
                            ${PATH_FIX}
                            cd ${COMPOSE_DIR_LINUX}

                            echo '🛑 Stopping old containers...'
                            docker compose down || true

                            echo '📥 Pulling images...'
                            docker compose pull

                            echo '🚀 Starting services...'
                            docker compose up -d

                            echo '⏳ Waiting 5s...'
                            sleep 5

                            echo '📊 Containers running:'
                            docker compose ps
                        "
                    """
                }
            }
        }

        /* -------------------------
           6) VERIFY SERVICES
        ------------------------- */
        stage("Verify Deployment") {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                script {
                    echo "🔍 Verifying remote containers..."

                    sh """
                        sshpass -p "${params.SSH_PASS}" \
                        ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                            ${PATH_FIX}

                            echo "--------- Docker PS ----------"
                            docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"

                            echo ""
                            echo "PostgreSQL running? -> \$(docker ps | grep -c postgres_db)"
                            echo "Redis running?      -> \$(docker ps | grep -c redis_cache)"
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Deployment Successful! Services running 🔥"
        }
        failure {
            echo "❌ Deployment Failed. Please check logs."
        }
    }
}
