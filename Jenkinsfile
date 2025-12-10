pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote Machine IP')
        string(name: 'REMOTE_USER', defaultValue: '', description: 'SSH Username')
        string(name: 'REMOTE_PASS', defaultValue: '', description: 'SSH Password')
    }

    environment {
        REGISTRY = "192.168.1.10:5000"   // <-- Your Jenkins + Registry IP
        COMPOSE_FILE = "docker-compose.yml"
        REMOTE_DEPLOY_DIR = "deploy"     // stored inside remote user's HOME
    }

    stages {

        /* ---------- 1. SSH VALIDATION ---------- */
        stage('Validate SSH Connection') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_IP} "echo CONNECTED"
                    """
                }
            }
        }

        /* ---------- 2. DETECT OS ---------- */
        stage('Detect Remote OS') {
            steps {
                script {
                    def out = sh(
                        script: """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no \
                            ${REMOTE_USER}@${REMOTE_IP} "uname || ver"
                        """,
                        returnStdout: true
                    ).trim()

                    if (out.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "LINUX"
                    } else {
                        env.OS_TYPE = "WINDOWS"
                    }

                    echo "Remote OS Detected → ${env.OS_TYPE}"
                }
            }
        }

        /* ---------- 3. CREATE REMOTE DEPLOY DIRECTORY ---------- */
        stage('Create Remote Deploy Directory') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_IP} '
                            mkdir -p ~/${REMOTE_DEPLOY_DIR};
                            echo "Using remote deploy directory: ~/${REMOTE_DEPLOY_DIR}";
                        '
                    """
                }
            }
        }

        /* ---------- 4. INSTALL DOCKER (LINUX ONLY) ---------- */
        stage('Install Docker on Remote (Linux)') {
            when { expression { env.OS_TYPE == "LINUX" } }
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_IP} '
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

        /* ---------- 5. BUILD + TAG + PUSH IMAGES ---------- */
        stage('Build & Push Docker Images') {
            steps {
                script {
                    sh """
                        docker-compose -f ${COMPOSE_FILE} build

                        # Tag and push all images to private registry
                        IMAGES=\$(docker-compose -f ${COMPOSE_FILE} images --quiet)

                        for IMG in \$IMAGES; do
                            NAME=\$(echo \$IMG | awk -F '/' '{print \$NF}')
                            echo "Pushing: ${REGISTRY}/\$NAME"

                            docker tag \$IMG ${REGISTRY}/\$NAME
                            docker push ${REGISTRY}/\$NAME
                        done
                    """
                }
            }
        }

        /* ---------- 6. COPY COMPOSE FILE TO REMOTE ---------- */
        stage('Copy Compose File to Remote') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' scp -o StrictHostKeyChecking=no \
                        ${COMPOSE_FILE} \
                        ${REMOTE_USER}@${REMOTE_IP}:~/${REMOTE_DEPLOY_DIR}/${COMPOSE_FILE}
                    """
                }
            }
        }

        /* ---------- 7. REMOTE DEPLOY VIA DOCKER COMPOSE ---------- */
        stage('Deploy Services on Remote') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_IP} "
                            docker compose -f ~/${REMOTE_DEPLOY_DIR}/${COMPOSE_FILE} pull &&
                            docker compose -f ~/${REMOTE_DEPLOY_DIR}/${COMPOSE_FILE} up -d
                        "
                    """
                }
            }
        }

        /* ---------- 8. VERIFY CONTAINERS ---------- */
        stage('Verify Deployment') {
            steps {
                script {
                    def status = sh(
                        script: """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no \
                            ${REMOTE_USER}@${REMOTE_IP} "docker ps"
                        """,
                        returnStdout: true
                    )

                    echo status

                    if (!status.contains("postgres") || !status.contains("redis")) {
                        error "❌ Postgres किंवा Redis चालू नाहीत."
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS: Remote PostgreSQL + Redis deployed successfully using registry 192.168.1.10:5000"
        }
        failure {
            echo "❌ Pipeline Failed — logs तपासा."
        }
    }
}
