pipeline {
    agent any

    parameters {
        string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP address')
        string(name: 'SSH_USER', defaultValue: '', description: 'SSH username')
        password(name: 'SSH_PASS', defaultValue: '', description: 'SSH password')
    }

    environment {
        REGISTRY = "192.168.1.8:5000"
        COMPOSE_FILE = "docker-compose.yml"
    }

    stages {

        /* -----------------------------
         * Stage 1: Validate Inputs
         * ----------------------------- */
        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.REMOTE_IP?.trim())  error "REMOTE_IP required"
                    if (!params.SSH_USER?.trim())   error "SSH_USER required"
                    if (!params.SSH_PASS)           error "SSH_PASS required"
                    if (!fileExists(env.COMPOSE_FILE)) error "docker-compose.yml not found"
                }
            }
        }

        /* -----------------------------
         * Stage 2: Find Images from Compose
         * ----------------------------- */
        stage('Find Images from Compose') {
            steps {
                script {
                    def images = sh(
                        script: "grep -E 'image:' ${env.COMPOSE_FILE} | awk '{print \$2}'",
                        returnStdout: true
                    ).trim().readLines()

                    if (images.isEmpty()) error "No images found in compose"

                    writeFile file: 'images.list', text: images.join("\n")
                    echo "Images:\n" + images.join("\n")
                }
            }
        }

        /* -----------------------------
         * Stage 3: Tag & Push Images (FIXED)
         * ----------------------------- */
        stage('Tag & Push Images') {
            steps {
                script {
                    def images = readFile('images.list').readLines()

                    for (img in images) {

                        // Split image:tag → redis + latest
                        def parts = img.split(":")
                        def name = parts[0]
                        def tag = parts.size() > 1 ? parts[1] : "latest"

                        def target = "${env.REGISTRY}/${name}:${tag}"

                        echo "Tagging & pushing → ${img} → ${target}"

                        sh """
                            docker pull ${img} || true
                            docker tag ${img} ${target}
                            docker push ${target}
                        """
                    }
                }
            }
        }

        /* -----------------------------
         * Stage 4: Create Remote Script
         * ----------------------------- */
        stage('Generate Remote Script') {
            steps {
                script {
                    def scriptText = """
                    #!/bin/bash
                    set -e

                    REGISTRY="${env.REGISTRY}"

                    echo "Installing Docker if missing..."
                    if ! command -v docker >/dev/null; then
                        if [ -f /etc/debian_version ]; then
                            apt update -y
                            apt install -y docker.io
                        elif [ -f /etc/redhat-release ]; then
                            yum install -y docker
                        fi
                        systemctl enable --now docker
                    fi

                    echo "Installing docker-compose if missing..."
                    if ! command -v docker-compose >/dev/null; then
                        curl -L https://github.com/docker/compose/releases/download/1.29.2/docker-compose-\\\$(uname -s)-\\\$(uname -m) -o /usr/local/bin/docker-compose
                        chmod +x /usr/local/bin/docker-compose
                    fi

                    mkdir -p /opt/jenkins_app
                    cp /tmp/docker-compose.yml /opt/jenkins_app/docker-compose.yml

                    cd /opt/jenkins_app
                    docker-compose pull || true
                    docker-compose up -d --remove-orphans

                    echo "Running containers:"
                    docker ps
                    """

                    writeFile file: "remote.sh", text: scriptText
                    sh "chmod +x remote.sh"
                }
            }
        }

        /* -----------------------------
         * Stage 5: Copy Files to Remote
         * ----------------------------- */
        stage('Copy Files to Remote') {
            steps {
                script {
                    sh """
                        sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no ${env.COMPOSE_FILE} ${params.SSH_USER}@${params.REMOTE_IP}:/tmp/docker-compose.yml
                        sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no remote.sh ${params.SSH_USER}@${params.REMOTE_IP}:/tmp/remote.sh
                    """
                }
            }
        }

        /* -----------------------------
         * Stage 6: Execute Remote Script
         * ----------------------------- */
        stage('Execute Remote Script') {
            steps {
                script {
                    sh """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.REMOTE_IP} "sudo bash /tmp/remote.sh"
                    """
                }
            }
        }

        /* -----------------------------
         * Stage 7: Verify Containers
         * ----------------------------- */
        stage('Verify Containers') {
            steps {
                script {
                    sh """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.REMOTE_IP} "docker ps"
                    """
                }
            }
        }

    }

    post {
        success { echo "Pipeline Successful!" }
        failure { echo "Pipeline Failed!" }
    }
}
