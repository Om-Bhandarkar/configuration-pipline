

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
    stage('Validate params') {
      steps {
        script {
          if (!params.TARGET_IP?.trim()) {
            error "TARGET_IP is required"
          }
          if (!params.TARGET_USER?.trim()) {
            error "TARGET_USER is required"
          }
          echo "Will run pipeline against ${params.TARGET_USER}@${params.TARGET_IP}"
        }
      }
    }

    stage('Prepare remote (OS detect & basic tools)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} 'uname -s || true' | sed -n '1p'"
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} 'sudo apt-get update -y && sudo apt-get install -y ufw curl ca-certificates'"
        }
      }
    }

    stage('Firewall & SSH hardening') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          // Allow default SSH (22) and registry (5000) first to avoid locking out
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"sudo ufw allow 22/tcp && sudo ufw allow 5000/tcp && sudo ufw --force enable\""

          // Example of minimal sshd hardening: disable root login and protocol tweaks
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"sudo sed -i -e 's/^#PermitRootLogin.*/PermitRootLogin no/' -e 's/^#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config || true && sudo systemctl restart sshd || true\""
        }
      }
    }

    stage('Install Docker & Docker Compose on remote') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"curl -fsSL https://get.docker.com | sudo sh\""
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"sudo curl -L 'https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)' -o /usr/local/bin/docker-compose && sudo chmod +x /usr/local/bin/docker-compose || true\""
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} 'sudo usermod -aG docker ${params.TARGET_USER} || true'"
        }
      }
    }

    stage('Deploy docker-compose (registry, postgres, redis)') {
      steps {
        // Create docker-compose.yml locally and copy to remote
        writeFile file: 'docker-compose.yml', text: '''version: "3.9"
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
'''

        withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh "scp -i ${SSH_KEY} -o StrictHostKeyChecking=no docker-compose.yml ${SSH_USER}@${params.TARGET_IP}:~/docker-compose.yml"
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} 'docker-compose -f ~/docker-compose.yml up -d'"
        }
      }
    }

    stage('Push images into remote private registry (built on remote)') {
      steps {
        // We'll perform pull/tag/push on remote itself to avoid insecure-registry issues
        withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          // Pull official images on remote, tag to registry, push
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"docker pull postgres:16 && docker tag postgres:16 localhost:5000/postgres-custom:latest && docker push localhost:5000/postgres-custom:latest\""
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"docker pull redis:latest && docker tag redis:latest localhost:5000/redis-custom:latest && docker push localhost:5000/redis-custom:latest\""
        }
      }
    }

    stage('Run containers from registry on remote') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          // login not required for default registry; but show how if credentials existed
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"docker pull localhost:5000/postgres-custom:latest && docker pull localhost:5000/redis-custom:latest\""

          // Stop any previous containers (safe restart)
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"docker rm -f pg-custom || true && docker rm -f redis-custom || true\""

          // Run new containers with basic options
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"docker run -d --name pg-custom -e POSTGRES_PASSWORD=example -e POSTGRES_USER=admin -p 5432:5432 localhost:5000/postgres-custom:latest\""
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} \"docker run -d --name redis-custom -p 6379:6379 localhost:5000/redis-custom:latest\""
        }
      }
    }

    stage('Verify remote containers') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'remote_ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${params.TARGET_IP} 'docker ps --format \"{{.Names}}\t{{.Image}}\t{{.Status}}\"'"
        }
      }
    }
  }

  post {
    success {
      echo 'Pipeline completed successfully.'
    }
    failure {
      echo 'Pipeline failed — check logs.'
    }
  }
}
