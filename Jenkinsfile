pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP')
        string(name: 'REMOTE_USER', defaultValue: '', description: 'SSH Username')
        string(name: 'REMOTE_PASS', defaultValue: '', description: 'SSH Password')
    }

    environment {
        REGISTRY = "192.168.1.10:5000"   // <-- इथे तुझ्या Jenkins मशीनचा IP टाक
        COMPOSE_FILE = "docker-compose.yml"
    }

    stages {

        /* -------------------- 1. Test SSH -------------------- */
        stage('Validate SSH Connection') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' \
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "echo connected"
                    """
                }
            }
        }

        /* -------------------- 2. Detect OS -------------------- */
        stage('Detect OS') {
            steps {
                script {
                    def result = sh(
                        script: """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "uname || ver"
                        """,
                        returnStdout: true
                    ).trim()

                    if (result.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "LINUX"
                    } else {
                        env.OS_TYPE = "WINDOWS"
                    }
                    echo "Detected Remote OS = ${env.OS_TYPE}"
                }
            }
        }

        /* -------------------- 3. Setup OS -------------------- */
        stage('Setup Remote System') {
            steps {
                script {

                    if (env.OS_TYPE == "LINUX") {
                        sh """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} '
                                sudo apt update -y &&
                                sudo apt install -y curl ufw &&
                                sudo ufw allow 22 &&
                                curl -fsSL https://get.docker.com | sudo sh &&
                                sudo usermod -aG docker ${REMOTE_USER}
                            '
                        """
                    } else {
                        sh """
                            sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                powershell Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0;
                                powershell Start-Service sshd;
                                powershell New-NetFirewallRule -Name sshd -Protocol TCP -LocalPort 22 -Action Allow;
                                choco install docker-desktop -y;
                                powershell Start-Service com.docker.service
                            "
                        """
                    }
                }
            }
        }

        /* -------------------- 4. Build & Push Images -------------------- */
        stage('Build & Push Images') {
            steps {
                script {
                    sh """
                        echo "Building images..."
                        docker-compose -f ${COMPOSE_FILE} build

                        echo "Tagging & Pushing to Registry..."
                        docker-compose -f ${COMPOSE_FILE} images --quiet | while read IMG; do
                            NAME=\$(echo \$IMG | awk -F '/' '{print \$NF}')
                            docker tag \$IMG ${REGISTRY}/\$NAME
                            docker push ${REGISTRY}/\$NAME
                        done
                    """
                }
            }
        }

        /* -------------------- 5. Copy Compose File -------------------- */
        stage('Transfer Compose File to Remote') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' \
                        scp -o StrictHostKeyChecking=no ${COMPOSE_FILE} \
                        ${REMOTE_USER}@${REMOTE_IP}:/home/${REMOTE_USER}/${COMPOSE_FILE}
                    """
                }
            }
        }

        /* -------------------- 6. Deploy on Remote -------------------- */
        stage('Deploy Services on Remote') {
            steps {
                script {
                    sh """
                        sshpass -p '${REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                            sudo systemctl restart docker;
                            docker compose -f ${COMPOSE_FILE} pull;
                            docker compose -f ${COMPOSE_FILE} up -d
                        "
                    """
                }
            }
        }

        /* -------------------- 7. Verify Deployment -------------------- */
        stage('Verify Services') {
            steps {
                script {
                    def output = sh(
                        script: """
                            sshpass -p '${REMOTE_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "docker ps"
                        """,
                        returnStdout: true
                    ).trim()

                    echo output

                    if (!output.contains("postgres") || !output.contains("redis")) {
                        error "Postgres किंवा Redis चालू नाहीत!"
                    }
                }
            }
        }
    }

    post {
        success { echo "Pipeline SUCCESSFUL 🎉" }
        failure { echo "Pipeline FAILED ❌" }
    }
}
