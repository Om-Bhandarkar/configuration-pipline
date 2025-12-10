pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote Machine IP')
        string(name: 'REMOTE_USER', defaultValue: '', description: 'SSH Username')
        string(name: 'REMOTE_PASS', defaultValue: '', description: 'SSH Password')   // Visible password field
    }

    environment {
        REGISTRY = "192.168.1.10:5000"   // <-- Your Jenkins + Registry machine IP
        COMPOSE_FILE = "docker-compose.yml"
    }

    stages {

        /* ---------- 1. SSH VALIDATION ---------- */
        stage('Validate SSH Connection') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "echo CONNECTED"
                    """
                }
            }
        }

        /* ---------- 2. DETECT OS ---------- */
        stage('Detect Remote OS') {
            steps {
                script {
                    def output = sh(
                        script: """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "uname || ver"
                        """,
                        returnStdout: true
                    ).trim()

                    if (output.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "LINUX"
                    } else {
                        env.OS_TYPE = "WINDOWS"
                    }

                    echo "Remote OS detected → ${env.OS_TYPE}"
                }
            }
        }

        /* ---------- 3. FIX HOME DIR ---------- */
        stage('Prepare Remote User Home Directory') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} '
                            if [ ! -d "/home/${REMOTE_USER}" ]; then
                                sudo mkdir -p /home/${REMOTE_USER};
                                sudo chown ${REMOTE_USER}:${REMOTE_USER} /home/${REMOTE_USER};
                            fi
                        '
                    """
                }
            }
        }

        /* ---------- 4. SETUP DOCKER (Linux only) ---------- */
        stage('Setup Docker on Remote Machine') {
            steps {
                script {
                    if (env.OS_TYPE == "LINUX") {
                        sh """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} '
                                sudo apt-get update -y &&
                                sudo apt-get install -y curl &&
                                curl -fsSL https://get.docker.com | sudo sh &&
                                sudo usermod -aG docker ${REMOTE_USER} &&
                                sudo systemctl restart docker
                            '
                        """
                    }
                }
            }
        }

        /* ---------- 5. BUILD + TAG + PUSH ---------- */
        stage('Build & Push Docker Images') {
            steps {
                script {
                    sh """
                        docker-compose -f ${COMPOSE_FILE} build

                        # Tag + push each image from compose
                        docker-compose -f ${COMPOSE_FILE} images --quiet | while read IMG; do
                            NAME=\$(echo \$IMG | awk -F '/' '{print \$NF}')
                            docker tag \$IMG ${REGISTRY}/\$NAME
                            docker push ${REGISTRY}/\$NAME
                        done
                    """
                }
            }
        }

        /* ---------- 6. COPY FILE ---------- */
        stage('Copy Compose File') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' scp -o StrictHostKeyChecking=no ${COMPOSE_FILE} \
                        ${REMOTE_USER}@${REMOTE_IP}:/home/${REMOTE_USER}/${COMPOSE_FILE}
                    """
                }
            }
        }

        /* ---------- 7. DEPLOY ---------- */
        stage('Deploy Services') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                            docker compose -f /home/${REMOTE_USER}/${COMPOSE_FILE} pull;
                            docker compose -f /home/${REMOTE_USER}/${COMPOSE_FILE} up -d
                        "
                    """
                }
            }
        }

        /* ---------- 8. VERIFY ---------- */
        stage('Verify Deployment') {
            steps {
                script {
                    def ps = sh(
                        script: """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "docker ps"
                        """,
                        returnStdout: true
                    )

                    echo ps

                    if (!ps.contains("redis") || !ps.contains("postgres")) {
                        error "❌ Services are not running!"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline Successful! Remote PostgreSQL + Redis deployed via Registry 192.168.1.10:5000."
        }
        failure {
            echo "❌ Pipeline Failed — कृपया logs तपासा."
        }
    }
}
