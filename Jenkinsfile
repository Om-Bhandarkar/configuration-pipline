pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP')
        string(name: 'REMOTE_USER', defaultValue: '', description: 'SSH Username')
        password(name: 'REMOTE_PASS', defaultValue: '', description: 'SSH Password')
    }

    environment {
        COMPOSE_FILE = "docker-compose.yml"
        REGISTRY = "your-registry.com"
    }

    stages {

        /* ----------------------- 1. TEST SSH ----------------------- */
        stage('Validate SSH Connection') {
            steps {
                script {
                    sh """
                        sshpass -p '${params.REMOTE_PASS}' \
                        ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "echo connected"
                    """
                }
            }
        }

        /* ----------------------- 2. DETECT OS ---------------------- */
        stage('Detect OS') {
            steps {
                script {
                    def os = sh(
                        script: """
                            sshpass -p '${params.REMOTE_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "uname || ver"
                        """,
                        returnStdout: true
                    ).trim()

                    if (os.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "LINUX"
                    } else {
                        env.OS_TYPE = "WINDOWS"
                    }

                    echo "OS Detected = ${env.OS_TYPE}"
                }
            }
        }

        /* ----------------------- 3. OS SETUP ----------------------- */
        stage('Setup Remote OS') {
            steps {
                script {
                    if (env.OS_TYPE == "LINUX") {

                        sh """
                            sshpass -p '${params.REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                                sudo apt-get update -y &&
                                sudo apt-get install -y openssh-server curl ufw &&
                                sudo ufw allow 22 &&
                                curl -fsSL https://get.docker.com | sudo sh &&
                                sudo usermod -aG docker ${params.REMOTE_USER}
                            '
                        """

                    } else {

                        sh """
                            sshpass -p '${params.REMOTE_PASS}' ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
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

        /* ---------------------- 4. BUILD + PUSH ---------------------- */
        stage('Build & Push Images') {
            steps {
                sh """
                    docker-compose -f ${COMPOSE_FILE} build
                    echo '${params.REMOTE_PASS}' | docker login ${REGISTRY} --username ${params.REMOTE_USER} --password-stdin
                    docker-compose -f ${COMPOSE_FILE} push
                """
            }
        }

        /* ---------------------- 5. COPY COMPOSE FILE ----------------- */
        stage('Copy Compose File') {
            steps {
                sh """
                    sshpass -p '${params.REMOTE_PASS}' \
                    scp -o StrictHostKeyChecking=no ${COMPOSE_FILE} \
                    ${params.REMOTE_USER}@${params.REMOTE_IP}:/home/${params.REMOTE_USER}/${COMPOSE_FILE}
                """
            }
        }

        /* ---------------------- 6. DEPLOY REMOTE --------------------- */
        stage('Deploy Containers') {
            steps {
                sh """
                    sshpass -p '${params.REMOTE_PASS}' \
                    ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "
                        echo '${params.REMOTE_PASS}' | docker login ${REGISTRY} --username ${params.REMOTE_USER} --password-stdin;
                        docker compose -f ${COMPOSE_FILE} pull;
                        docker compose -f ${COMPOSE_FILE} up -d
                    "
                """
            }
        }

        /* ---------------------- 7. VERIFY ---------------------------- */
        stage('Verify Services') {
            steps {
                script {
                    def output = sh(
                        script: """
                            sshpass -p '${params.REMOTE_PASS}' \
                            ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} \
                            "docker ps"
                        """,
                        returnStdout: true
                    ).trim()

                    echo output

                    if (!output.contains("postgres") || !output.contains("redis")) {
                        error "PostgreSQL OR Redis not running!"
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
