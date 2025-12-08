pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Target server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        COMPOSE_DIR = "/infra"
        COMPOSE_FILE = "/infra/docker-compose.yml"
    }

    stages {

        stage('Check Connection') {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "echo Connected"
                """
            }
        }

        stage('Install Docker & docker-compose') {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        if ! command -v docker >/dev/null; then
                            apt-get update -y || yum update -y
                            apt-get install -y docker.io || yum install -y docker
                            systemctl start docker || true
                            systemctl enable docker || true
                        fi

                        if ! command -v docker-compose >/dev/null; then
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\$(uname -s)-\\$(uname -m)" -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        fi
                    '
                """
            }
        }

        stage('Upload docker-compose.yml') {
            steps {
                script {
                    writeFile file: "docker-compose.yml", text: """
version: "3.9"

services:

  app:
    image: localhost:5000/react-app:v1
    container_name: react_app
    ports:
      - "8081:80"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: always

  postgres:
    image: postgres:latest
    container_name: postgres_db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: root
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: always

  redis:
    image: redis:latest
    container_name: redis_server
    ports:
      - "6479:6479"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 4s
      timeout: 3s
      retries: 5
    restart: always

volumes:
  pgdata:
"""
                    sh """
                        sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "mkdir -p ${COMPOSE_DIR}"
                        sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${TARGET_IP}:${COMPOSE_FILE}
                    """
                }
            }
        }

        stage('Run docker-compose') {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        cd ${COMPOSE_DIR}
                        docker-compose down || true
                        docker-compose up -d
                    '
                """
            }
        }
    }

    post {
        success { echo "Deployment Successful 🎉" }
        failure { echo "Deployment Failed ❌" }
    }
}
