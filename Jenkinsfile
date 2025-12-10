pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        REGISTRY_URL = "${TARGET_IP}:5000"

        LINUX_COMPOSE_DIR = "/infra"
        LINUX_COMPOSE_FILE = "/infra/docker-compose.yml"

        WIN_COMPOSE_DIR = "C:/infra"
        WIN_COMPOSE_FILE = "C:/infra/docker-compose.yml"

        OS_TYPE = ""
    }

    stages {

        /* Detect OS */
        stage("Detect Remote OS") {
            steps {
                script {
                    def linuxCheck = sh(returnStatus: true, script: """
                        sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "uname -s"
                    """)

                    if (linuxCheck == 0) {
                        env.OS_TYPE = "linux"
                    } else {
                        def winCheck = sh(returnStatus: true, script: """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                            "powershell -Command \\"(Get-CimInstance Win32_OperatingSystem).Caption\\""
                        """)

                        if (winCheck == 0) env.OS_TYPE = "windows"
                        else error "Cannot detect OS"
                    }
                }
            }
        }

        /* Pull + Tag + Push */
        stage("Push Images To Registry") {
            steps {
                sh """
                    # Pull from Docker Hub
                    docker pull postgres:latest
                    docker pull redis:latest

                    # Tag for private registry
                    docker tag postgres:latest ${REGISTRY_URL}/infra/postgres:latest
                    docker tag redis:latest ${REGISTRY_URL}/infra/redis:latest

                    # Allow insecure registry on remote
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                        mkdir -p /etc/docker
                        echo "{\\"insecure-registries\\":[\\"${REGISTRY_URL}\\"]}" > /etc/docker/daemon.json
                        systemctl restart docker || true
                    '

                    # Push to private registry
                    docker push ${REGISTRY_URL}/infra/postgres:latest
                    docker push ${REGISTRY_URL}/infra/redis:latest
                """
            }
        }

        /* Upload compose file */
        stage("Upload Compose File") {
            steps {
                script {

                    def compose = """
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
    image: ${TARGET_IP}:5000/infra/postgres:latest
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - "5432:5432"
    restart: unless-stopped

  redis:
    container_name: infra-redis
    image: ${TARGET_IP}:5000/infra/redis:latest
    ports:
      - "6379:6379"
    restart: unless-stopped

volumes:
  registry_data:
"""

                    writeFile file: "docker-compose.yml", text: compose

                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "mkdir -p ${LINUX_COMPOSE_DIR}"
                            sshpass -p "${SSH_PASS}" scp docker-compose.yml ${SSH_USER}@${TARGET_IP}:${LINUX_COMPOSE_FILE}
                        """
                    } else {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} ^
                                "powershell -Command \\"New-Item -ItemType Directory -Force -Path '${WIN_COMPOSE_DIR}'\\""
                            sshpass -p "${SSH_PASS}" scp docker-compose.yml ${SSH_USER}@${TARGET_IP}:${WIN_COMPOSE_FILE}
                        """
                    }
                }
            }
        }

        /* Deploy */
        stage("Deploy Services") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                                cd ${LINUX_COMPOSE_DIR}
                                docker compose down || true
                                docker compose pull
                                docker compose up -d
                            '
                        """
                    } else {
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} ^
                                "powershell -Command \\"cd '${WIN_COMPOSE_DIR}'; docker compose down; docker compose pull; docker compose up -d\\""
                        """
                    }
                }
            }
        }

        /* Verify */
        stage("Verify") {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "docker ps"
                """
            }
        }
    }
}
