pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', defaultValue: '', description: 'Remote machine IP')
        string(name: 'SSH_USER', defaultValue: 'Administrator', description: 'SSH Username')
        password(name: 'SSH_PASS', defaultValue: '', description: 'SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file to deploy')
    }

    stages {

        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.TARGET_IP) error "IP address is required"
                    if (!params.SSH_USER) error "Username is required"
                    if (!params.SSH_PASS) error "Password is required"
                    if (!fileExists(params.COMPOSE_FILE)) error "Compose file not found in workspace"
                }
            }
        }

        stage('SSH Connectivity Check') {
            steps {
                sh """
                which sshpass >/dev/null 2>&1 || (echo 'ERROR: sshpass not installed on Jenkins agent!' && exit 2)

                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                    ${params.SSH_USER}@${params.TARGET_IP} 'echo SSH_OK'
                """
                echo "SSH connection successful."
            }
        }

        stage('Detect Remote OS') {
            steps {
                script {
                    def os = sh(
                        script: """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} 'uname -s' 2>/dev/null || echo WINDOWS
                        """,
                        returnStdout: true
                    ).trim()

                    env.REMOTE_OS = os.contains("Linux") ? "LINUX" : "WINDOWS"
                    echo "Remote OS detected: ${env.REMOTE_OS}"
                }
            }
        }

        stage('Install Docker & Compose (Linux only)') {
            when { expression { env.REMOTE_OS == "LINUX" } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.TARGET_IP} '
                    if ! command -v docker >/dev/null 2>&1; then
                        echo "Installing Docker..."
                        curl -fsSL https://get.docker.com | sudo sh
                    fi

                    if ! command -v docker-compose >/dev/null 2>&1; then
                        echo "Installing docker-compose..."
                        sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m)" \
                        -o /usr/local/bin/docker-compose
                        sudo chmod +x /usr/local/bin/docker-compose
                    fi
                '
                """
                echo "Docker & docker-compose installed for Linux."
            }
        }

        stage('Copy docker-compose.yml to Remote') {
            steps {
                script {
                    def remotePath = (env.REMOTE_OS == "WINDOWS") ?
                        "/Users/${params.SSH_USER}/docker-compose.yml" :
                        "~/docker-compose.yml"

                    sh """
                    sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                        ${params.COMPOSE_FILE} ${params.SSH_USER}@${params.TARGET_IP}:${remotePath}
                    """

                    echo "Compose file copied to: ${remotePath}"
                }
            }
        }

        /* ---------------------- WINDOWS FLOW ------------------------ */
        stage('Start Containers (Windows)') {
            when { expression { env.REMOTE_OS == "WINDOWS" } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} powershell -Command "
                    cd C:/Users/${params.SSH_USER};
                    if (!(docker compose)) { Write-Host 'Docker Desktop not installed!'; exit 1 }
                    docker compose up -d;
                "
                """
                echo "Windows deployment done using 'docker compose'"
            }
        }

        /* ---------------------- LINUX FLOW ------------------------ */
        stage('Start Containers (Linux)') {
            when { expression { env.REMOTE_OS == "LINUX" } }
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    cd ~;
                    docker-compose up -d --remove-orphans;
                '
                """
                echo "Linux deployment done using docker-compose"
            }
        }

        stage('Verify Running Containers') {
            steps {
                script {
                    def cmd = (env.REMOTE_OS == "WINDOWS") ?
                        "docker ps --format \\\"CONTAINER: {{.Names}} IMAGE: {{.Image}} STATUS: {{.Status}}\\\"" :
                        "docker ps --format \\\"CONTAINER: {{.Names}} IMAGE: {{.Image}} STATUS: {{.Status}}\\\""

                    def output = sh(
                        script: """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "${cmd}"
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Container Status:\n${output}"
                    if (!output) error("No containers running!")
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
