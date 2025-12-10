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

        /* ----------------------------------------------------------
           DETECT REMOTE OS (LINUX OR WINDOWS)
        ---------------------------------------------------------- */
        stage("Detect Remote OS") {
            steps {
                script {
                    echo "🔍 Detecting Remote OS..."

                    // Try Linux
                    def linuxCheck = sh(returnStatus: true, script: """
                        sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "uname -s"
                    """)

                    if (linuxCheck == 0) {
                        env.OS_TYPE = "linux"
                        echo "✅ Remote OS Detected: LINUX"
                    } else {
                        // Try Windows
                        def winCheck = sh(returnStatus: true, script: """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                            "powershell -NoProfile -Command \\"(Get-CimInstance Win32_OperatingSystem).Caption\\""
                        """)

                        if (winCheck == 0) {
                            env.OS_TYPE = "windows"
                            echo "🟦 Remote OS Detected: WINDOWS"
                        } else {
                            error "❌ Could not detect remote OS"
                        }
                    }
                }
            }
        }

        /* ----------------------------------------------------------
           PULL → TAG → PUSH IMAGES TO PRIVATE REGISTRY
        ---------------------------------------------------------- */
        stage("Push Images to Registry") {
            steps {
                sh """
                    docker pull postgres:latest
                    docker pull redis:latest

                    docker tag postgres:latest ${REGISTRY_URL}/infra/postgres:latest
                    docker tag redis:latest ${REGISTRY_URL}/infra/redis:latest

                    echo "📤 Pushing images to private registry..."
                    docker push ${REGISTRY_URL}/infra/postgres:latest
                    docker push ${REGISTRY_URL}/infra/redis:latest
                """
            }
        }

        /* ----------------------------------------------------------
           CREATE docker-compose.yml
        ---------------------------------------------------------- */
        stage("Create Compose File") {
            steps {
                script {
                    def composeText = """
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
                    writeFile file: "docker-compose.yml", text: composeText
                }
            }
        }

        /* ----------------------------------------------------------
           UPLOAD docker-compose.yml (WINDOWS + LINUX)
        ---------------------------------------------------------- */
        stage("Upload Compose File") {
            steps {
                script {

                    if (env.OS_TYPE == "linux") {

                        echo "📤 Uploading compose file to LINUX..."
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} "mkdir -p ${LINUX_COMPOSE_DIR}"
                            sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${LINUX_COMPOSE_FILE}
                        """

                    } else {

                        echo "📤 Uploading compose file to WINDOWS..."
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                                "powershell -NoProfile -Command \\"New-Item -ItemType Directory -Force -Path '${WIN_COMPOSE_DIR}'\\""

                            sshpass -p "${SSH_PASS}" scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${WIN_COMPOSE_FILE}
                        """
                    }
                }
            }
        }

        /* ----------------------------------------------------------
           DEPLOY SERVICES (WINDOWS + LINUX)
        ---------------------------------------------------------- */
        stage("Deploy Services") {
            steps {
                script {

                    if (env.OS_TYPE == "linux") {

                        echo "🚀 Deploying on LINUX..."
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                                cd ${LINUX_COMPOSE_DIR}
                                docker compose down || true
                                docker compose pull
                                docker compose up -d
                            '
                        """

                    } else {

                        echo "🚀 Deploying on WINDOWS..."
                        sh """
                            sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                                "powershell -NoProfile -Command \\"cd '${WIN_COMPOSE_DIR}'; docker compose down; docker compose pull; docker compose up -d\\""
                        """
                    }
                }
            }
        }

        /* ----------------------------------------------------------
           VERIFY RUNNING SERVICES
        ---------------------------------------------------------- */
        stage("Verify Deployment") {
            steps {
                sh """
                    sshpass -p "${SSH_PASS}" ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                        "docker ps --format 'table {{.Names}}\\t{{.Image}}\\t{{.Status}}'"
                """
            }
        }
    }

    post {
        always {
            sh "rm -f docker-compose.yml || true"
        }
    }
}
