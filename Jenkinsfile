pipeline {
    agent any

    // 1) User inputs
    parameters {
        string(name: 'TARGET_IP', description: 'Remote machine IP')
        string(name: 'TARGET_USER', description: 'Remote machine username')
        password(name: 'TARGET_PASSWORD', description: 'Remote machine password')
        string(name: 'REGISTRY_URL', defaultValue: 'myregistry.example.com', description: 'Private Docker registry (e.g. myregistry.example.com)')
    }

    environment {
        REGISTRY       = "${REGISTRY_URL}"
        POSTGRES_IMAGE = "${REGISTRY_URL}/custom-postgres:latest"
        REDIS_IMAGE    = "${REGISTRY_URL}/custom-redis:latest"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2) Build & push images to private registry
        stage('Build & Push Docker Images') {
            steps {
                sh '''
                echo "Building and pushing Postgres & Redis images..."

                # Example: adjust paths / Dockerfiles as per your repo
                docker build -t $POSTGRES_IMAGE -f docker/postgres/Dockerfile .
                docker build -t $REDIS_IMAGE    -f docker/redis/Dockerfile    .

                docker push $POSTGRES_IMAGE
                docker push $REDIS_IMAGE
                '''
            }
        }

        // 3) Detect remote OS (Linux / Windows)
        stage('Detect Remote OS') {
            steps {
                script {
                    def osInfo = sh(
                        script: """
                        sshpass -p '${TARGET_PASSWORD}' \
                          ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} \
                          "uname 2>/dev/null || ver 2>/dev/null"
                        """,
                        returnStdout: true
                    ).trim().toLowerCase()

                    if (osInfo.contains('linux')) {
                        env.REMOTE_OS = 'linux'
                    } else if (osInfo.contains('windows')) {
                        env.REMOTE_OS = 'windows'
                    } else {
                        error "Remote OS detect करू शकलो नाही. Output: ${osInfo}"
                    }

                    echo "Remote OS detected: ${env.REMOTE_OS}"
                }
            }
        }

        // 4) Check native Postgres & Redis on remote
        stage('Check Native Postgres & Redis on Remote') {
            steps {
                script {
                    if (env.REMOTE_OS == 'linux') {
                        sh """
                        sshpass -p '${TARGET_PASSWORD}' \
                          ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} '
                            echo "Checking PostgreSQL..."
                            if command -v psql >/dev/null 2>&1; then
                              echo "PostgreSQL already installed on host";
                            else
                              echo "PostgreSQL NOT installed on host";
                            fi

                            echo "Checking Redis..."
                            if command -v redis-server >/dev/null 2>&1; then
                              echo "Redis already installed on host";
                            else
                              echo "Redis NOT installed on host";
                            fi
                          '
                        """
                    } else if (env.REMOTE_OS == 'windows') {
                        // Windows check (PowerShell) – adjust as per your setup
                        sh """
                        sshpass -p '${TARGET_PASSWORD}' \
                          ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} "
                            powershell -Command \\
                              \\"Write-Output 'Checking PostgreSQL...'; \\
                                if (Get-Command psql -ErrorAction SilentlyContinue) { 'PostgreSQL already installed' } else { 'PostgreSQL NOT installed' }; \\
                                Write-Output 'Checking Redis...'; \\
                                if (Get-Command redis-server -ErrorAction SilentlyContinue) { 'Redis already installed' } else { 'Redis NOT installed' }\\" "
                        """
                    }
                }
            }
        }

        // 5) Install Docker + Docker Compose on remote (if missing)
        stage('Install Docker on Remote') {
            steps {
                script {
                    if (env.REMOTE_OS == 'linux') {
                        sh """
                        sshpass -p '${TARGET_PASSWORD}' \
                          ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} '
                            set -e

                            if ! command -v docker >/dev/null 2>&1; then
                              echo "Installing Docker..."
                              curl -fsSL https://get.docker.com | sh
                            else
                              echo "Docker already installed"
                            fi

                            if ! command -v docker-compose >/dev/null 2>&1; then
                              echo "Installing Docker Compose..."
                              sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" -o /usr/local/bin/docker-compose
                              sudo chmod +x /usr/local/bin/docker-compose
                            else
                              echo "Docker Compose already installed"
                            fi
                          '
                        """
                    } else if (env.REMOTE_OS == 'windows') {
                        // NOTE: ही script फक्त example आहे. तुझ्या env नुसार installer बदला (Chocolatey, winget इ.)
                        sh """
                        sshpass -p '${TARGET_PASSWORD}' \
                          ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} "
                            powershell -Command \\
                              \\"if (-not (Get-Command docker -ErrorAction SilentlyContinue)) { \\
                                  Write-Output 'Installing Docker (Windows)...'; \\
                                  choco install docker-cli -y \\
                                } else { \\
                                  Write-Output 'Docker already installed' \\
                                }\\"
                        "
                        """
                    }
                }
            }
        }

        // 6) Copy docker-compose.yml to remote and deploy containers
        stage('Deploy via Docker Compose on Remote') {
            steps {
                script {
                    // docker-compose.yml Jenkins repo मधून SCP ने remote वर पाठवतो
                    sh """
                    sshpass -p '${TARGET_PASSWORD}' \
                      scp -o StrictHostKeyChecking=no docker-compose.yml \
                          ${TARGET_USER}@${TARGET_IP}:/tmp/docker-compose.yml
                    """

                    if (env.REMOTE_OS == 'linux') {
                        sh """
                        sshpass -p '${TARGET_PASSWORD}' \
                          ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} '
                            set -e
                            echo "Logging into registry $REGISTRY ..."
                            docker login $REGISTRY

                            echo "Pulling latest images..."
                            docker-compose -f /tmp/docker-compose.yml pull

                            echo "Starting containers..."
                            docker-compose -f /tmp/docker-compose.yml up -d
                          '
                        """
                    } else if (env.REMOTE_OS == 'windows') {
                        // Windows साठीही same idea – फक्त paths/commands Windows-friendly
                        sh """
                        sshpass -p '${TARGET_PASSWORD}' \
                          ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} "
                            docker login $REGISTRY && \\
                            docker-compose -f /tmp/docker-compose.yml pull && \\
                            docker-compose -f /tmp/docker-compose.yml up -d
                          "
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Remote machine (${TARGET_IP}) वर Postgres & Redis कंटेनर यशस्वीपणे deploy झाले."
        }
        failure {
            echo "❌ Pipeline failed – Jenkins console log मध्ये तपशील बघ."
        }
    }
}
