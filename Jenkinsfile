pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        REGISTRY_URL = "${TARGET_IP}:5000"
        COMPOSE_DIR = "/infra"
        COMPOSE_FILE = "/infra/docker-compose.yml"
    }

    stages {

        stage("Build Images") {
            steps {
                sh """
                    docker build -t infra/postgres:latest postgres/
                    docker build -t infra/redis:latest redis/
                """
            }
        }

        stage("Push Images to Registry") {
            steps {
                sh """
                    docker tag infra/postgres:latest ${REGISTRY_URL}/infra/postgres:latest
                    docker tag infra/redis:latest ${REGISTRY_URL}/infra/redis:latest
                    
                    # Allow insecure registry on remote
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        mkdir -p /etc/docker
                        echo "{\\"insecure-registries\\":[\\"${TARGET_IP}:5000\\"]}" > /etc/docker/daemon.json
                        systemctl restart docker
                    '

                    docker push ${REGISTRY_URL}/infra/postgres:latest
                    docker push ${REGISTRY_URL}/infra/redis:latest
                """
            }
        }

        stage("Upload Compose File") {
            steps {
                script {

                    // Compose template
                    def composeTemplate = """
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
    image: __REGISTRY_IP__:5000/infra/postgres:latest
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - "5432:5432"
    restart: unless-stopped
    depends_on:
      - registry

  redis:
    container_name: infra-redis
    image: __REGISTRY_IP__:5000/infra/redis:latest
    ports:
      - "6379:6379"
    restart: unless-stopped
    depends_on:
      - registry

volumes:
  registry_data:
"""

                    // Replace placeholder with real IP
                    def finalCompose = composeTemplate.replace("__REGISTRY_IP__", TARGET_IP)

                    // Write file locally
                    writeFile file: 'docker-compose.yml', text: finalCompose

                    // Upload to remote
                    sh """
                        sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} 'mkdir -p ${COMPOSE_DIR}'
                        sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${TARGET_IP}:${COMPOSE_FILE}
                    """
                }
            }
        }

        stage("Deploy") {
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
}
