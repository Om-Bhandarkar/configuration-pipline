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

        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.REMOTE_IP?.trim()) error "REMOTE_IP required"
                    if (!params.SSH_USER?.trim()) error "SSH_USER required"
                    if (!params.SSH_PASS) error "SSH_PASS required"   // FIXED
                    if (!fileExists(env.COMPOSE_FILE)) error "docker-compose.yml not found"
                }
            }
        }

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

        stage('Tag & Push Images') {
            steps {
                script {
                    def images = readFile('images.list').readLines()

                    for (img in images) {
                        def name = img.contains('/') ? img.substring(img.indexOf('/')+1) : img
                        def tag = img.contains(":") ? img.split(':')[-1] : "latest"
                        def target = "${env.REGISTRY}/${name}:${tag}"

                        sh """
                            docker pull ${img} || true
                            docker tag ${img} ${target}
                            docker push ${target}
                        """
                    }
                }
            }
        }

        stage('Generate Remote Script') {
            steps {
                script {
                    def scriptText = """
                    #!/bin/bash
                    set -e

                    REGISTRY="${env.REGISTRY}"
                    COMPOSE="${env.COMPOSE_FILE}"

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
                    cd /opt/jenkins_app

                    echo "Pulling images..."
                    docker-compose -f ${env.COMPOSE_FILE} pull || true

                    echo "Starting containers..."
                    docker-compose -f ${env.COMPOSE_FILE} up -d --remove-orphans

                    echo "Running containers:"
                    docker ps
                    """

                    writeFile file: "remote.sh", text: scriptText
                    sh "chmod +x remote.sh"
                }
            }
        }

        stage('Copy Files to Remote') {
            steps {
                script {
                    sh """
                        sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no ${env.COMPOSE_FILE} ${params.SSH_USER}@${params.REMOTE_IP}:/opt/jenkins_app/
                        sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no remote.sh ${params.SSH_USER}@${params.REMOTE_IP}:/tmp/
                    """
                }
            }
        }

        stage('Execute Remote Script') {
            steps {
                script {
                    sh """
                        sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no ${params.SSH_USER}@${params.REMOTE_IP} "sudo bash /tmp/remote.sh"
                    """
                }
            }
        }

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
