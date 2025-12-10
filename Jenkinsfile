pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        REGISTRY_URL = "${TARGET_IP}:5000"

        LINUX_DIR  = "/infra"
        LINUX_COMPOSE = "/infra/docker-compose.yml"

        WIN_DIR    = "C:/infra"
        WIN_COMPOSE = "C:/infra/docker-compose.yml"

        OS_TYPE = ""
    }

    stages {

        /* ----------------------------------------------------------
           1) DETECT REMOTE OS
        ---------------------------------------------------------- */
        stage("Detect Remote OS") {
            steps {
                script {
                    echo "🔍 Detecting Remote OS..."

                    def linuxCheck = sh(returnStatus: true, script: """
                        sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} uname -s
                    """)

                    if (linuxCheck == 0) {
                        env.OS_TYPE = "linux"
                        echo "✅ Detected: LINUX"
                    } else {
                        def winCheck = sh(returnStatus: true, script: """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                            powershell -NoProfile -Command "(Get-CimInstance Win32_OperatingSystem).Caption"
                        """)

                        if (winCheck == 0) {
                            env.OS_TYPE = "windows"
                            echo "🟦 Detected: WINDOWS"
                        } else {
                            error "❌ Could not detect OS for ${TARGET_IP}"
                        }
                    }
                }
            }
        }

        /* ----------------------------------------------------------
           2) PUSH IMAGES TO PRIVATE REGISTRY
        ---------------------------------------------------------- */
        stage("Push Images to Registry") {
            steps {
                sh """
                    docker pull postgres:latest
                    docker pull redis:latest

                    docker tag postgres:latest ${REGISTRY_URL}/infra/postgres:latest
                    docker tag redis:latest    ${REGISTRY_URL}/infra/redis:latest

                    echo "📤 Pushing images to private registry..."
                    docker push ${REGISTRY_URL}/infra/postgres:latest
                    docker push ${REGISTRY_URL}/infra/redis:latest
                """
            }
        }

        /* ----------------------------------------------------------
           3) CREATE docker-compose.yml
        ---------------------------------------------------------- */
        stage("Create Compose File") {
            steps {
                script {
                    def compose = """
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
                }
            }
        }

        /* ----------------------------------------------------------
           4) UPLOAD COMPOSE FILE
        ---------------------------------------------------------- */
        stage("Upload Compose File") {
            steps {
                script {

                    if (env.OS_TYPE == "linux") {

                        echo "📤 Uploading compose to Linux..."

                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                                "mkdir -p ${LINUX_DIR}"

                            sshpass -p '${SSH_PASS}' scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${LINUX_COMPOSE}
                        """

                    } else {

                        echo "📤 Uploading compose to Windows & preparing Docker config..."

                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                              powershell -NoProfile -Command "
                                New-Item -ItemType Directory -Force -Path 'C:/infra' | Out-Null;
                                New-Item -ItemType Directory -Force -Path 'C:/docker-config' | Out-Null;
                                Set-Content -Path 'C:/docker-config/config.json' -Value '{}';
                              "

                            sshpass -p '${SSH_PASS}' scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${WIN_COMPOSE}
                        """
                    }
                }
            }
        }

        /* ----------------------------------------------------------
           5) DEPLOY (LINUX + WINDOWS FIXED)
        ---------------------------------------------------------- */
        stage("Deploy Services") {
            steps {
                script {

                    if (env.OS_TYPE == "linux") {

                        echo "🚀 Deploying on Linux..."

                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                                cd ${LINUX_DIR}
                                docker compose down || true
                                docker compose pull
                                docker compose up -d
                            '
                        """

                    } else {

                        echo "🚀 Creating deploy.ps1 for Windows..."

                        writeFile file: "deploy.ps1", text: """
\$Env:DOCKER_CONFIG = 'C:/docker-config'

Set-Content -Path 'C:/docker-config/config.json' -Value '{
  "credsStore": "",
  "auths": {
    "${REGISTRY_URL}": { "auth": "" }
  }
}'

Write-Host "Using DOCKER_CONFIG: \$Env:DOCKER_CONFIG"

Set-Location 'C:/infra'
docker logout ${REGISTRY_URL} 2>\$null
docker compose down
docker compose pull
docker compose up -d
"""

                        sh """
                            sshpass -p '${SSH_PASS}' scp -o StrictHostKeyChecking=no deploy.ps1 \
                                ${SSH_USER}@${TARGET_IP}:'C:/infra/deploy.ps1'
                        """

                        echo "🚀 Running deploy.ps1..."

                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                                powershell -NoProfile -ExecutionPolicy Bypass -File 'C:/infra/deploy.ps1'
                        """
                    }
                }
            }
        }

        /* ----------------------------------------------------------
           6) VERIFY (WINDOWS FORMAT FIXED)
        ---------------------------------------------------------- */
        stage("Verify Deployment") {
            steps {
                script {

                    if (env.OS_TYPE == "windows") {

                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                              powershell -NoProfile -Command "docker ps --format \\\"table {{.Names}}\\t{{.Image}}\\t{{.Status}}\\\""
                        """

                    } else {

                        sh """
                            sshpass -p '${SSH_PASS}' ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                              "docker ps --format 'table {{.Names}}\\t{{.Image}}\\t{{.Status}}'"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh "rm -f docker-compose.yml || true"
            sh "rm -f deploy.ps1 || true"
        }
    }
}
