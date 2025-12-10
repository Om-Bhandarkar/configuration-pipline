pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', description: 'Remote machine IP address')
        string(name: 'REMOTE_USER', description: 'SSH username')
        password(name: 'REMOTE_PASS', description: 'SSH password')
    }

    stages {

        /* --------------------------------------------------
           1) SSH Connectivity Test
        -------------------------------------------------- */
        stage('Validate SSH Connection') {
            steps {
                sh """
                    echo "🔍 Checking SSH connectivity..."
                    sshpass -p ${params.REMOTE_PASS} ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "echo connected"
                """
            }
        }

        /* --------------------------------------------------
           2) Detect Remote OS
        -------------------------------------------------- */
        stage('Detect Remote OS') {
            steps {
                script {
                    def osName = sh(script: """
                        sshpass -p ${params.REMOTE_PASS} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "uname || cmd /c ver"
                    """, returnStdout: true).trim()

                    if (osName.contains("Linux")) {
                        env.REMOTE_OS = "LINUX"
                    } else if (osName.contains("Windows") || osName.contains("Microsoft")) {
                        env.REMOTE_OS = "WINDOWS"
                    } else {
                        error("❌ Unsupported OS detected")
                    }

                    echo "Remote OS Detected: ${env.REMOTE_OS}"
                }
            }
        }

        /* --------------------------------------------------
           3) Linux Setup
        -------------------------------------------------- */
        stage('Remote OS Setup - Linux') {
            when { expression { env.REMOTE_OS == "LINUX" } }
            steps {
                sh """
                    sshpass -p ${params.REMOTE_PASS} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "
                        sudo apt update &&
                        sudo apt install -y openssh-server ufw &&
                        sudo ufw allow 22 &&
                        curl -fsSL https://get.docker.com | sh &&
                        sudo usermod -aG docker ${params.REMOTE_USER} &&
                        sudo apt install -y docker-compose-plugin
                    "
                """
            }
        }

        /* --------------------------------------------------
           4) Windows Setup
        -------------------------------------------------- */
        stage('Remote OS Setup - Windows') {
            when { expression { env.REMOTE_OS == "WINDOWS" } }
            steps {
                sh """
                    sshpass -p ${params.REMOTE_PASS} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "
                        powershell Install-WindowsFeature OpenSSH-Server ;
                        powershell New-NetFirewallRule -DisplayName 'SSH' -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow ;
                        powershell winget install Docker.DockerDesktop --silent ;
                    "
                """
            }
        }

        /* --------------------------------------------------
           5) Check Existing Containers
        -------------------------------------------------- */
        stage('Check Existing Containers') {
            steps {
                script {
                    def result = sh(script: """
                        sshpass -p ${params.REMOTE_PASS} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "
                            docker ps -a --format '{{.Image}}'
                        "
                    """, returnStdout: true)

                    env.POSTGRES_EXISTS = result.contains("postgres") ? "YES" : "NO"
                    env.REDIS_EXISTS = result.contains("redis") ? "YES" : "NO"

                    echo "PostgreSQL Exists? ${env.POSTGRES_EXISTS}"
                    echo "Redis Exists? ${env.REDIS_EXISTS}"
                }
            }
        }

        /* --------------------------------------------------
           6) Build Docker Images if Missing
        -------------------------------------------------- */
        stage('Build Missing Docker Images') {
            when { expression { env.POSTGRES_EXISTS == "NO" || env.REDIS_EXISTS == "NO" } }
            steps {
                sh """
                    docker-compose build
                    docker login -u myuser -p mypass myregistry.com
                    docker-compose push
                """
            }
        }

        /* --------------------------------------------------
           7) Copy docker-compose.yml to Remote
        -------------------------------------------------- */
        stage('Transfer Compose File') {
            steps {
                sh """
                    sshpass -p ${params.REMOTE_PASS} scp docker-compose.yml ${params.REMOTE_USER}@${params.REMOTE_IP}:/home/${params.REMOTE_USER}/
                """
            }
        }

        /* --------------------------------------------------
           8) Deploy Services
        -------------------------------------------------- */
        stage('Deploy Docker Services') {
            steps {
                sh """
                    sshpass -p ${params.REMOTE_PASS} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "
                        docker login -u myuser -p mypass myregistry.com &&
                        docker compose -f docker-compose.yml pull &&
                        docker compose -f docker-compose.yml up -d
                    "
                """
            }
        }

        /* --------------------------------------------------
           9) Verify Containers
        -------------------------------------------------- */
        stage('Verify Services') {
            steps {
                sh """
                    sshpass -p ${params.REMOTE_PASS} ssh ${params.REMOTE_USER}@${params.REMOTE_IP} "docker ps"
                """
            }
        }
    }

    post {
        success { echo "🎉 Pipeline executed successfully!" }
        failure { echo "❌ Pipeline failed. Check logs." }
    }
}
