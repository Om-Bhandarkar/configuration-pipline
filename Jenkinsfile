pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote Server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        OS_TYPE = ""
        REGISTRY_URL = "${TARGET_IP}:5000"
        
        LINUX_DIR  = "/infra"
        LINUX_COMPOSE = "/infra/docker-compose.yml"

        WIN_DIR    = "C:/infra"
        WIN_COMPOSE = "C:/infra/docker-compose.yml"
    }

    stages {

        /* -------------------------------------------------------------
           DETECT OS
        ------------------------------------------------------------- */
        stage("Detect Remote OS") {
            steps {
                script {
                    echo "🔍 Detecting Remote OS..."

                    def linuxCheck = sh(returnStatus: true, script: """
                        sshpass -p '${SSH_PASS}' \
                        ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} uname -s
                    """)

                    if (linuxCheck == 0) {
                        env.OS_TYPE = "linux"
                        echo "🐧 OS Detected: Linux"

                    } else {
                        def winCheck = sh(returnStatus: true, script: """
                            sshpass -p '${SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                            powershell -NoProfile -Command "(Get-CimInstance Win32_OperatingSystem).Caption"
                        """)

                        if (winCheck == 0) {
                            env.OS_TYPE = "windows"
                            echo "🟦 OS Detected: Windows"
                        } else {
                            error "❌ Could not detect remote OS"
                        }
                    }
                }
            }
        }

        /* -------------------------------------------------------------
           PUSH IMAGES INTO PRIVATE REGISTRY
        ------------------------------------------------------------- */
        stage("Push Images to Registry") {
            steps {
                sh """
                    docker pull postgres:latest
                    docker pull redis:latest

                    docker tag postgres:latest ${REGISTRY_URL}/infra/postgres:latest
                    docker tag redis:latest    ${REGISTRY_URL}/infra/redis:latest

                    docker push ${REGISTRY_URL}/infra/postgres:latest
                    docker push ${REGISTRY_URL}/infra/redis:latest
                """
            }
        }

        /* -------------------------------------------------------------
           CREATE docker-compose.yml
        ------------------------------------------------------------- */
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
    image: ${REGISTRY_URL}/infra/postgres:latest
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - "5432:5432"
    restart: unless-stopped

  redis:
    container_name: infra-redis
    image: ${REGISTRY_URL}/infra/redis:latest
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

        /* -------------------------------------------------------------
           UPLOAD COMPOSE FILE (LINUX & WINDOWS)
        ------------------------------------------------------------- */
        stage("Upload Compose File") {
            steps {
                script {
                    if (env.OS_TYPE == "linux") {

                        echo "📤 Uploading compose to Linux..."

                        sh """
                            sshpass -p '${SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                                "mkdir -p ${LINUX_DIR}"

                            sshpass -p '${SSH_PASS}' \
                            scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${LINUX_COMPOSE}
                        """

                    } else {

                        echo "📤 Uploading to Windows..."

                        sh """
                            sshpass -p '${SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                              powershell -NoProfile -Command "
                                New-Item -ItemType Directory -Force -Path 'C:/infra' | Out-Null;
                                New-Item -ItemType Directory -Force -Path 'C:/docker-config' | Out-Null;
                              "

                            sshpass -p '${SSH_PASS}' \
                            scp -o StrictHostKeyChecking=no docker-compose.yml \
                                ${SSH_USER}@${TARGET_IP}:${WIN_COMPOSE}
                        """
                    }
                }
            }
        }

        /* -------------------------------------------------------------
           DEPLOY SERVICES (LINUX & WINDOWS FIXED)
        ------------------------------------------------------------- */
        stage("Deploy Services") {
            steps {
                script {

                    if (env.OS_TYPE == "linux") {

                        echo "🚀 Deploying on Linux..."

                        sh """
                            sshpass -p '${SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} '
                                cd ${LINUX_DIR}
                                docker compose down || true
                                docker compose pull
                                docker compose up -d
                            '
                        """

                    } else {

                        echo "🟦 Preparing Windows deploy.ps1..."

                        writeFile file: "deploy.ps1", text: """
Write-Host "🔧 Setting Docker Config..."

\$Env:DOCKER_CONFIG = "C:/docker-config"

@"
{
  "auths": {
    "${REGISTRY_URL}": { }
  },
  "credsStore": ""
}
"@ | Set-Content -Path "C:/docker-config/config.json"

Write-Host "✔ DOCKER_CONFIG loaded"

Stop-Service com.docker.service
Start-Service com.docker.service
Start-Sleep -Seconds 5

Write-Host "🚀 Deploying..."

Set-Location "C:/infra"

docker compose down --remove-orphans
docker compose pull --ignore-pull-failures
docker compose up -d

Write-Host "✔ Deployment Completed"
"""

                        sh """
                            sshpass -p '${SSH_PASS}' \
                            scp -o StrictHostKeyChecking=no deploy.ps1 \
                                ${SSH_USER}@${TARGET_IP}:'C:/infra/deploy.ps1'
                        """

                        sh """
                            sshpass -p '${SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                              powershell -NoProfile -ExecutionPolicy Bypass -File 'C:/infra/deploy.ps1'
                        """
                    }
                }
            }
        }

        /* -------------------------------------------------------------
           VERIFY DEPLOYMENT
        ------------------------------------------------------------- */
        stage("Verify Deployment") {
            steps {
                script {
                    if (env.OS_TYPE == "windows") {

                        sh """
                            sshpass -p '${SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
                              powershell -NoProfile -Command "docker ps --format \\"table {{.Names}},{{.Image}},{{.Status}}\\""
                        """

                    } else {

                        sh """
                            sshpass -p '${SSH_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${SSH_USER}@${TARGET_IP} \
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
