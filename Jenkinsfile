pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', description: 'Target server IP')
        string(name: 'SSH_USER', defaultValue: 'root', description: 'SSH Username')
        password(name: 'SSH_PASS', description: 'SSH Password')
    }

    environment {
        COMPOSE_DIR  = "/infra"
        COMPOSE_FILE = "/infra/docker-compose.yml"
    }

    stages {

        /* 1. Check SSH Connection */
        stage('Check Connection') {
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                    "echo Connected Successfully"
                """
            }
        }

        /* 2. Install Docker & docker-compose */
        stage('Install Docker & docker-compose') {
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '

                        # Install Docker
                        if ! command -v docker >/dev/null 2>&1; then
                            echo "Installing Docker..."
                            apt-get update -y || yum update -y
                            apt-get install -y docker.io || yum install -y docker
                            systemctl start docker || true
                            systemctl enable docker || true
                        fi

                        # Install docker-compose
                        if ! command -v docker-compose >/dev/null 2>&1; then
                            echo "Installing docker-compose..."
                            curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \
                                -o /usr/local/bin/docker-compose
                            chmod +x /usr/local/bin/docker-compose
                        fi

                    '
                """
            }
        }

        /* 3. Upload External docker-compose.yml file */
        stage('Upload docker-compose.yml') {
            steps {
                sh """
                    # Create folder on remote server
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} \
                    "mkdir -p ${COMPOSE_DIR}"

                    # Upload your local docker-compose.yml file
                    sshpass -p "${params.SSH_PASS}" \
                    scp -o StrictHostKeyChecking=no docker-compose.yml \
                    ${params.SSH_USER}@${params.TARGET_IP}:${COMPOSE_FILE}
                """
            }
        }

        /* 4. Run docker-compose on server */
        stage('Run docker-compose') {
            steps {
                sh """
                    sshpass -p "${params.SSH_PASS}" \
                    ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                        cd ${COMPOSE_DIR}
                        docker-compose down || true
                        docker-compose up -d
                    '
                """
            }
        }
    }

    post {
        success {
            echo "🎉 Deployment Successful"
        }
        failure {
            echo "❌ Deployment Failed"
        }
    }
}
