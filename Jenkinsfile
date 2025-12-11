pipeline {
    agent any

    parameters {
        string(name: 'TARGET_IP', defaultValue: '', description: 'Remote host IP')
        string(name: 'TARGET_USER', defaultValue: 'ubuntu', description: 'Remote SSH user')
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Validate Params') {
            steps {
                script {
                    if (!params.TARGET_IP?.trim()) {
                        error "TARGET_IP is required"
                    }
                    if (!params.TARGET_USER?.trim()) {
                        error "TARGET_USER is required"
                    }
                    echo "Running pipeline for ${params.TARGET_USER}@${params.TARGET_IP}"
                }
            }
        }

        stage('OS Detect + Basic Setup') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', 
                                                   keyFileVariable: 'SSH_KEY', 
                                                   usernameVariable: 'SSH_USER')]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "uname -s"
                    '''

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "sudo apt-get update -y && sudo apt-get install -y ufw curl ca-certificates"
                    '''
                }
            }
        }

        stage('Firewall + SSH Hardening') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "sudo ufw allow 22/tcp && sudo ufw allow 5000/tcp && sudo ufw --force enable"
                    '''

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "
                            sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config;
                            sudo sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config;
                            sudo systemctl restart sshd || true
                        "
                    '''
                }
            }
        }

        stage('Install Docker + Compose') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "curl -fsSL https://get.docker.com | sudo sh"
                    '''

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "
                            sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) \
                                 -o /usr/local/bin/docker-compose;
                            sudo chmod +x /usr/local/bin/docker-compose
                        "
                    '''

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "sudo usermod -aG docker $TARGET_USER"
                    '''
                }
            }
        }

        stage('Copy docker-compose.yml & Start Services') {
            steps {
                script {
                    writeFile file: "docker-compose.yml", text: """
version: "3.9"
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
      POSTGRES_USER: admin
    ports:
      - "5432:5432"

  redis:
    image: redis:latest
    ports:
      - "6379:6379"

  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry_volume:/var/lib/registry

volumes:
  registry_volume:
"""
                }

                withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {

                    sh '''
                        scp -o StrictHostKeyChecking=no -i $SSH_KEY docker-compose.yml $SSH_USER@$TARGET_IP:~/docker-compose.yml
                    '''

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "docker-compose -f ~/docker-compose.yml up -d"
                    '''
                }
            }
        }

        stage('Push Images to Private Registry') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "
                            docker pull postgres:16 &&
                            docker tag postgres:16 localhost:5000/postgres-custom:latest &&
                            docker push localhost:5000/postgres-custom:latest
                        "
                    '''

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "
                            docker pull redis:latest &&
                            docker tag redis:latest localhost:5000/redis-custom:latest &&
                            docker push localhost:5000/redis-custom:latest
                        "
                    '''
                }
            }
        }

        stage('Run Custom Containers From Registry') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "
                            docker pull localhost:5000/postgres-custom:latest;
                            docker pull localhost:5000/redis-custom:latest;
                        "
                    '''

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "
                            docker rm -f pg-custom redis-custom || true;
                            docker run -d --name pg-custom -p 5432:5432 localhost:5000/postgres-custom:latest;
                            docker run -d --name redis-custom -p 6379:6379 localhost:5000/redis-custom:latest;
                        "
                    '''
                }
            }
        }

        stage('Verify Containers') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_USER@$TARGET_IP "docker ps --format 'CONTAINER: {{.Names}} | IMAGE: {{.Image}} | {{.Status}}'"
                    '''
                }
            }
        }

    }

    post {
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed — check logs."
        }
    }
}
