pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP')
        string(name: 'REMOTE_USER', defaultValue: '', description: 'SSH Username')
        credentials(name: 'SSH_CRED', description: 'SSH Password or SSH Private Key')
    }

    environment {
        COMPOSE_FILE = "docker-compose.yml"
        REGISTRY = "your-docker-registry.com"
        PROJECT = "infra-services"
    }

    stages {

        /* -------------------------- 1. SSH CONNECTIVITY TEST -------------------------- */
        stage('Validate SSH Connection') {
            steps {
                script {
                    echo "Testing SSH connection to ${params.REMOTE_IP}..."

                    def result = sshCommand(
                        remote: [
                            host: params.REMOTE_IP,
                            user: params.REMOTE_USER,
                            credentialsId: 'SSH_CRED'
                        ],
                        command: "echo connected"
                    )

                    if (!result.contains("connected")) {
                        error "SSH failed! पुढे pipeline चालू शकत नाही."
                    }

                    echo "SSH OK ✓"
                }
            }
        }

        /* -------------------------- 2. DETECT REMOTE OS -------------------------- */
        stage('Detect Remote Operating System') {
            steps {
                script {
                    echo "Detecting OS..."

                    def osCheck = sshCommand(
                        remote: [host: params.REMOTE_IP, user: params.REMOTE_USER, credentialsId: 'SSH_CRED'],
                        command: "uname || ver"
                    )

                    if (osCheck.toLowerCase().contains("linux")) {
                        env.OS_TYPE = "LINUX"
                    } else if (osCheck.toLowerCase().contains("windows")) {
                        env.OS_TYPE = "WINDOWS"
                    } else {
                        error "Unknown OS detected!"
                    }

                    echo "Remote OS = ${env.OS_TYPE}"
                }
            }
        }

        /* -------------------------- 3. OS-SPECIFIC SETUP -------------------------- */
        stage('Setup Remote OS for Docker & SSH') {
            steps {
                script {

                    if (env.OS_TYPE == "LINUX") {
                        echo "Running Linux setup..."

                        sshCommand remote: remoteConfig(), command: """
                            sudo apt-get update -y
                            sudo apt-get install -y openssh-server ufw curl
                            sudo ufw allow 22
                            curl -fsSL https://get.docker.com | sudo sh
                            sudo usermod -aG docker ${params.REMOTE_USER}
                            sudo systemctl enable docker
                            sudo systemctl start docker
                        """

                    } else {
                        echo "Running Windows setup..."

                        sshCommand remote: remoteConfig(), command: """
                            powershell Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
                            powershell Start-Service sshd
                            powershell Set-Service -Name sshd -StartupType 'Automatic'
                            powershell New-NetFirewallRule -Name sshd -DisplayName 'SSH' -Protocol TCP -LocalPort 22 -Action Allow
                            
                            choco install docker-desktop -y
                            powershell Start-Service com.docker.service
                        """
                    }
                }
            }
        }

        /* -------------------------- 4. Build & Push Images -------------------------- */
        stage('Build & Push Containers') {
            steps {
                script {
                    sh """
                        docker-compose -f ${COMPOSE_FILE} build
                        docker login ${REGISTRY}
                        docker-compose -f ${COMPOSE_FILE} push
                    """
                }
            }
        }

        /* -------------------------- 5. Transfer Compose File -------------------------- */
        stage('Transfer Compose File to Remote') {
            steps {
                sshPut(
                    remote: remoteConfig(),
                    from: "${COMPOSE_FILE}",
                    into: "/home/${params.REMOTE_USER}/${COMPOSE_FILE}"
                )
            }
        }

        /* -------------------------- 6. Remote Deployment -------------------------- */
        stage('Deploy on Remote Machine') {
            steps {
                script {
                    sshCommand remote: remoteConfig(), command: """
                        docker login ${REGISTRY}
                        docker compose -f ${COMPOSE_FILE} pull
                        docker compose -f ${COMPOSE_FILE} up -d
                    """
                }
            }
        }

        /* -------------------------- 7. Verify Deployment -------------------------- */
        stage('Verify Services') {
            steps {
                script {
                    def ps = sshCommand(remote: remoteConfig(), command: "docker ps")
                    echo ps
                    if (!ps.contains("postgres") || !ps.contains("redis")) {
                        error "Containers NOT running properly!"
                    }
                }
            }
        }
    }

    /* -------------------------- POST ACTIONS -------------------------- */
    post {
        success {
            echo "Pipeline SUCCESSFUL 🎉"
        }
        failure {
            echo "Pipeline FAILED ❌ — logs पाहा."
        }
    }
}

/* -------------------------- Helper Function -------------------------- */
def remoteConfig() {
    return [
        host: params.REMOTE_IP,
        user: params.REMOTE_USER,
        credentialsId: 'SSH_CRED'
    ]
}
