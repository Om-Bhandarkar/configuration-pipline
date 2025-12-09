pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote machine IP')
        string(name: 'USERNAME', description: 'Remote username')
        password(name: 'PASSWORD', description: 'Remote password')
    }

    environment {
        IMAGE_REGISTRY = "your-private-registry.com"
        POSTGRES_IMAGE = "postgres:latest"
        REDIS_IMAGE    = "redis:latest"
    }

    stages {

        stage('Detect OS over SSH') {
            steps {
                script {
                    sh """
                    sshpass -p '${PASSWORD}' ssh -o StrictHostKeyChecking=no ${USERNAME}@${TARGET_IP} 'uname -s' > os.txt
                    """
                    def os = readFile('os.txt').trim()

                    if (os == "Linux") {
                        env.OS_TYPE = "linux"
                    } else {
                        env.OS_TYPE = "windows"
                    }

                    echo "Detected OS: ${OS_TYPE}"
                }
            }
        }

        stage('Install Docker & Docker Compose if missing') {
            when { expression { env.OS_TYPE == "linux" } }
            steps {
                sh """
                sshpass -p '${PASSWORD}' ssh -o StrictHostKeyChecking=no ${USERNAME}@${TARGET_IP} '
                    if ! command -v docker >/dev/null; then
                        echo "Installing Docker..."
                        curl -fsSL https://get.docker.com | sudo sh
                    fi

                    if ! command -v docker-compose >/dev/null; then
                        echo "Installing Docker Compose..."
                        sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-\\\$(uname -s)-\\\$(uname -m)" -o /usr/local/bin/docker-compose
                        sudo chmod +x /usr/local/bin/docker-compose
                    fi
                '
                """
            }
        }

        stage('Pull & Push Images to Private Registry') {
            steps {
                sh """
                # Pull official images
                docker pull ${POSTGRES_IMAGE}
                docker pull ${REDIS_IMAGE}

                # Retag for private registry
                docker tag ${POSTGRES_IMAGE} ${IMAGE_REGISTRY}/postgres:latest
                docker tag ${REDIS_IMAGE}    ${IMAGE_REGISTRY}/redis:latest

                # Push to private registry
                docker push ${IMAGE_REGISTRY}/postgres:latest
                docker push ${IMAGE_REGISTRY}/redis:latest
                """
            }
        }

        stage('Deploy on Remote Machine') {
            steps {
                sh """
                sshpass -p '${PASSWORD}' ssh -o StrictHostKeyChecking=no ${USERNAME}@${TARGET_IP} '
                    docker pull ${IMAGE_REGISTRY}/postgres:latest
                    docker pull ${IMAGE_REGISTRY}/redis:latest

                    mkdir -p ~/deploy
                '

                sshpass -p '${PASSWORD}' scp docker-compose.yml ${USERNAME}@${TARGET_IP}:~/deploy/

                sshpass -p '${PASSWORD}' ssh ${USERNAME}@${TARGET_IP} '
                    cd ~/deploy
                    docker-compose up -d
                '
                """
            }
        }
    }

    post {
        success { echo "🚀 Deployment Successful!" }
        failure { echo "❌ Deployment Failed!" }
    }
}
