// Works ONLY for Windows (Docker Desktop + OpenSSH)

pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', defaultValue: '', description: 'Remote Windows machine IP')
        string(name: 'SSH_USER', defaultValue: 'om', description: 'Windows SSH Username')
        password(name: 'SSH_PASS', defaultValue: '', description: 'Windows SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file')
    }

    stages {

        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.TARGET_IP) error "TARGET_IP is required"
                    if (!params.SSH_USER) error "SSH_USER is required"
                    if (!params.SSH_PASS) error "SSH_PASS is required"
                    if (!fileExists(params.COMPOSE_FILE)) error "Compose file not found"
                }
            }
        }

        stage('SSH Check') {
            steps {
                sh """
                which sshpass >/dev/null || exit 2
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} "echo SSH_OK"
                """
            }
        }

        /* ========= OS DETECTION (WINDOWS EXPECTED) ========= */
        stage('Detect Remote OS') {
            steps {
                script {
                    def os = sh(
                        script: """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                        ${params.SSH_USER}@${params.TARGET_IP} "ver"
                        """,
                        returnStdout: true
                    ).trim()

                    env.REMOTE_OS = "WINDOWS"
                    echo "✅ Detected OS: WINDOWS"
                }
            }
        }
        /* ================================================ */

        stage('Verify Docker Desktop (Windows)') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                powershell -NoProfile -Command "& {
                    if (!(Get-Command docker -ErrorAction SilentlyContinue)) {
                        Write-Error 'Docker Desktop not installed or not running'
                        exit 1
                    }
                    docker compose version
                }"
                """
            }
        }

        stage('Copy Compose File') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                ${params.COMPOSE_FILE} \
                ${params.SSH_USER}@${params.TARGET_IP}:C:/Users/${params.SSH_USER}/docker-compose.yml
                """
            }
        }

        stage('Deploy Containers') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                powershell -NoProfile -Command "& {
                    cd C:/Users/${params.SSH_USER}
                    docker compose down --remove-orphans
                    docker compose up -d
                }"
                """
            }
        }

        stage('Verify Containers') {
            steps {
                sh """
                sshpass -p '${params.SSH_PASS}' ssh ${params.SSH_USER}@${params.TARGET_IP} \
                powershell -NoProfile -Command "& {
                    docker ps
                }"
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful on WINDOWS"
        }
        failure {
            echo "❌ Deployment failed"
        }
    }
}
