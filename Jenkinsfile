pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        COMPOSE_DIR = "/infra"
        COMPOSE_FILE = "/infra/docker-compose.yml"
    }

    stages {

        stage("Upload Compose File") {
            steps {
                script {
                    def composeContent = """
version: "3.8"

services:

  registry:
    container_name: private-registry
    image: registry:2
    ports:
      - "5000:5000"
    restart: unless-stopped
    volumes:
      - registry_data:/var/lib/registry

  postgres:
    container_name: infra-postgres
    image: postgres:latest
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - "5432:5432"
    restart: unless-stopped
    depends_on:
      - registry

  redis:
    container_name: infra-redis
    image: redis:latest
    ports:
      - "6379:6379"
    restart: unless-stopped
    depends_on:
      - registry

volumes:
  registry_data:
"""

                    writeFile file: 'docker-compose.yml', text: composeContent

                    sh """
                        sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "mkdir -p ${COMPOSE_DIR}"
                        sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${TARGET_IP}:${COMPOSE_FILE}
                    """
                }
            }
        }

        stage("Deploy Services") {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        cd ${COMPOSE_DIR}
                        docker compose down || true
                        docker compose pull
                        docker compose up -d
                    '
                """
            }
        }

        stage("Verify") {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "docker ps"
                """
            }
        }
    }

    post {
        always {
            sh "rm -f docker-compose.yml"
        }
    }
}
