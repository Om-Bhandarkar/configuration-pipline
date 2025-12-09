pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Remote machine IP')
        string(name: 'TARGET_USER', description: 'Remote machine username')
        password(name: 'TARGET_PASSWORD', description: 'Remote machine password')
    }

    stages {

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
                    } else {
                        error "Unsupported OS. Only Linux supported in Case 1."
                    }

                    echo "Remote OS detected: ${env.REMOTE_OS}"
                }
            }
        }

        stage('Install Docker on Remote') {
            steps {
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
            }
        }

        stage('Copy docker-compose.yml to Remote') {
            steps {
                sh """
                sshpass -p '${TARGET_PASSWORD}' \
                  scp -o StrictHostKeyChecking=no docker-compose.yml \
                      ${TARGET_USER}@${TARGET_IP}:/tmp/docker-compose.yml
                """
            }
        }

        stage('Deploy Containers') {
            steps {
                sh """
                sshpass -p '${TARGET_PASSWORD}' \
                  ssh -o StrictHostKeyChecking=no ${TARGET_USER}@${TARGET_IP} '
                    docker-compose -f /tmp/docker-compose.yml up -d
                '
                """
            }
        }
    }

    post {
        success {
            echo "🎉 Remote machine वर Postgres आणि Redis यशस्वीपणे deploy झाले!"
        }
        failure {
            echo "❌ Pipeline failed – console log तपासा."
        }
    }
}
