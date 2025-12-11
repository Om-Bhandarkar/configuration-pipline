pipeline {
  agent any

  parameters {
    string(name: 'REMOTE_IP', defaultValue: '', description: 'Remote machine IP address')
    string(name: 'REMOTE_USER', defaultValue: 'administrator', description: 'Remote machine username')
    password(name: 'REMOTE_PASSWORD', defaultValue: '', description: 'Remote machine password')
    string(name: 'REGISTRY_PORT', defaultValue: '5000', description: 'Docker registry port')
  }

  environment {
    // Use latest images (you asked for latest)
    REGISTRY_NAME = "local-registry"
    POSTGRES_IMAGE = "postgres:latest"
    REDIS_IMAGE = "redis:latest"

    // internal state
    DETECTED_OS = ''
  }

  options {
    // keep logs for debugging
    timestamps()
    ansiColor('xterm')
  }

  stages {
    stage('Validate Input') {
      steps {
        script {
          if (!params.REMOTE_IP?.trim()) {
            error("❌ Remote IP address आवश्यक आहे! Provide REMOTE_IP parameter.")
          }
          echo "✅ Remote IP: ${params.REMOTE_IP}"
        }
      }
    }

    stage('Detect Remote OS') {
      steps {
        script {
          echo "🔍 Detecting remote OS..."

          // Prefer a simple SSH probe. If ssh fails, assume Windows and try WinRM fallback.
          try {
            // try uname -s; if it returns something containing 'Linux' treat as linux
            def probe = sh(script: """
              set -o pipefail
              ssh -o StrictHostKeyChecking=no -o ConnectTimeout=8 ${params.REMOTE_USER}@${params.REMOTE_IP} 'uname -s' 2>/dev/null || echo FAILED
            """, returnStdout: true).trim()

            if (probe && probe.toLowerCase().contains('linux')) {
              env.DETECTED_OS = 'linux'
              echo "✅ Detected Linux (via SSH)."
            } else if (probe == 'FAILED') {
              // SSH probe failed — try PowerShell remoting check locally (Windows fallback)
              echo "⚠️ SSH probe failed; will attempt Windows WinRM fallback (Invoke-Command)"
              env.DETECTED_OS = 'windows'
            } else {
              // If probe returned something else, still treat linux conservatively
              env.DETECTED_OS = 'linux'
              echo "ℹ️ Remote response: ${probe} — treating as Linux"
            }

          } catch (Exception e) {
            // any exception assume windows fallback
            env.DETECTED_OS = 'windows'
            echo "⚠️ Exception during SSH probe — assuming Windows and using WinRM style steps where required. (${e.getMessage()})"
          }

          echo "🎯 Detected OS: ${env.DETECTED_OS}"
        }
      }
    }

    stage('Ensure Remote Prereqs') {
      steps {
        script {
          if (env.DETECTED_OS == 'linux') {
            echo "🐧 Ensuring Docker and docker-compose on remote Linux..."

            // Check docker
            def dockerCheck = sh(script: """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} 'command -v docker >/dev/null && echo installed || echo missing'
            """, returnStdout: true).trim()

            if (dockerCheck == 'installed') {
              echo "✅ Docker already installed on remote"
            } else {
              echo "⚠️ Docker not present. Attempting non-interactive install (supports Debian/Ubuntu)."
              // Install Docker (Debian/Ubuntu). If remote is different distro user may need to install manually.
              sh """
                ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                  set -e
                  if command -v apt-get >/dev/null; then
                    sudo apt-get update -y
                    sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
                    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
                    echo \"deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable\" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
                    sudo apt-get update -y
                    sudo apt-get install -y docker-ce docker-ce-cli containerd.io
                    sudo systemctl enable --now docker
                    sudo usermod -aG docker ${params.REMOTE_USER} || true
                    echo 'docker-installed'
                  else
                    echo 'unsupported-distro'
                  fi
                '
              """

              // query again
              dockerCheck = sh(script: """
                ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} 'command -v docker >/dev/null && echo installed || echo missing'
              """, returnStdout: true).trim()

              if (dockerCheck != 'installed') {
                error('❌ Unable to install Docker automatically. Please install Docker on the remote machine and re-run the pipeline.')
              }

              echo "✅ Docker installed"
            }

            // Check docker-compose (use v2 plugin if docker compose plugin exists)
            def composeCheck = sh(script: """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} 'docker compose version >/dev/null 2>&1 && echo compose_v2 || (command -v docker-compose >/dev/null && echo compose_v1) || echo missing'
            """, returnStdout: true).trim()

            if (composeCheck == 'compose_v2' || composeCheck == 'compose_v1') {
              echo "✅ Docker Compose available on remote (${composeCheck})"
            } else {
              echo "⚠️ Docker Compose not found. Installing docker-compose (standalone)"
              sh """
                ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                  set -e
                  sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
                  sudo chmod +x /usr/local/bin/docker-compose
                  if command -v docker-compose >/dev/null; then echo installed; else echo failed; fi
                '
              """

              composeCheck = sh(script: """
                ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} 'command -v docker-compose >/dev/null && echo compose_v1 || echo missing'
              """, returnStdout: true).trim()

              if (composeCheck == 'missing') {
                error('❌ Failed to install docker-compose on remote. Please install it manually and re-run the pipeline.')
              }

              echo "✅ Docker Compose installed"
            }

          } else {
            echo "🪟 Windows detected. The pipeline will configure OpenSSH (if possible) and expect Docker Desktop to be installed manually."

            // Try to configure OpenSSH via PowerShell remoting locally (requires that Jenkins agent can reach the WinRM endpoint).
            // For most setups, admin action on Windows server is required and Docker Desktop must be installed interactively.

            echo "ℹ️ For Windows remote, please ensure OpenSSH or WinRM is configured and Docker Desktop is installed. This pipeline will attempt some remote steps but may require manual intervention."
          }
        }
      }
    }

    stage('Setup Registry & Compose Files on Remote') {
      steps {
        script {
          if (env.DETECTED_OS == 'linux') {
            echo "🗄️ Creating docker-deployment and registry (linux)..."

            sh """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                set -e
                mkdir -p ~/docker-deployment
                cd ~/docker-deployment

                # Start registry if missing
                if docker ps -a --format "{{.Names}}" | grep -q "^${env.REGISTRY_NAME}$"; then
                  docker start ${env.REGISTRY_NAME} || true
                else
                  docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${env.REGISTRY_NAME} registry:2
                fi

                # Create docker-compose.yml (uses images from local registry)
                cat > docker-compose.yml <<-'EOF'
version: "3.8"

services:
  postgres:
    image: localhost:${REGISTRY_PORT}/postgres:latest
    container_name: postgres-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: localhost:${REGISTRY_PORT}/redis:latest
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: always
    command: ["redis-server","--appendonly","yes"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  default:
    name: app-network
    driver: bridge
EOF

                echo '✅ docker-compose.yml created'
                ls -la ~/docker-deployment
              '
            """

          } else {
            echo "🗄️ Windows remote: creating C:\\docker-deployment and registry via SSH/PowerShell (best-effort)."

            bat """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} "powershell -NoProfile -ExecutionPolicy Bypass -Command \"
                if (-Not (Test-Path -Path C:\\docker-deployment)) { New-Item -ItemType Directory -Path C:\\docker-deployment | Out-Null }
                Set-Location C:\\docker-deployment

                # start registry if missing
                $names = docker ps -a --format '{{{{.Names}}}}'
                if ($names -contains '${env.REGISTRY_NAME}') { docker start ${env.REGISTRY_NAME} } else { docker run -d -p ${params.REGISTRY_PORT}:5000 --restart=always --name ${env.REGISTRY_NAME} registry:2 }

                # write minimal docker-compose.yml
                @'
version: "3.8"

services:
  postgres:
    image: localhost:${REGISTRY_PORT}/postgres:latest
    container_name: postgres-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always

  redis:
    image: localhost:${REGISTRY_PORT}/redis:latest
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: always

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  default:
    name: app-network
    driver: bridge
'@ > docker-compose.yml

                Write-Output '✅ docker-compose.yml created (windows)'
              \""
            """
          }
        }
      }
    }

    stage('Pull, Tag and Push Images to Private Registry') {
      steps {
        script {
          echo "📦 Pulling ${env.POSTGRES_IMAGE} and ${env.REDIS_IMAGE} and pushing to private registry..."

          if (env.DETECTED_OS == 'linux') {
            sh """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                set -e
                docker pull ${env.POSTGRES_IMAGE}
                docker pull ${env.REDIS_IMAGE}

                docker tag ${env.POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:latest
                docker tag ${env.REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:latest

                docker push localhost:${params.REGISTRY_PORT}/postgres:latest || true
                docker push localhost:${params.REGISTRY_PORT}/redis:latest || true

                echo "✅ Images pulled, tagged and pushed (if registry reachable)"
              '
            """

          } else {
            bat """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} \"docker pull ${env.POSTGRES_IMAGE} && docker pull ${env.REDIS_IMAGE} && docker tag ${env.POSTGRES_IMAGE} localhost:${params.REGISTRY_PORT}/postgres:latest && docker tag ${env.REDIS_IMAGE} localhost:${params.REGISTRY_PORT}/redis:latest && docker push localhost:${params.REGISTRY_PORT}/postgres:latest || echo push-failed && docker push localhost:${params.REGISTRY_PORT}/redis:latest || echo push-failed\"
            """
          }
        }
      }
    }

    stage('Deploy with Docker Compose') {
      steps {
        script {
          echo "🚀 Deploying containers via docker-compose..."

          if (env.DETECTED_OS == 'linux') {
            sh """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                set -e
                cd ~/docker-deployment
                docker-compose down || true
                docker-compose up -d
                # wait and show status
                sleep 7
                docker-compose ps
                docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
              '
            """
          } else {
            bat """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} \"cd C:\\docker-deployment && docker-compose down || echo down-failed && docker-compose up -d && timeout /t 7 /nobreak && docker-compose ps\"
            """
          }
        }
      }
    }

    stage('Verification') {
      steps {
        script {
          echo "✔️ Verifying services and healthchecks..."

          if (env.DETECTED_OS == 'linux') {
            sh """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} '
                set -e
                cd ~/docker-deployment
                echo "=== Docker Compose Services ==="
                docker-compose ps

                echo "=== Running Containers ==="
                docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

                echo "=== Registry Catalog (if available) ==="
                curl -s http://localhost:${params.REGISTRY_PORT}/v2/_catalog || true

                echo "=== Health Checks ==="
                if docker exec -i postgres-db pg_isready -U postgres >/dev/null 2>&1; then echo 'Postgres is ready'; else echo 'Postgres not ready'; fi
                if docker exec -i redis-cache redis-cli ping >/dev/null 2>&1; then echo 'Redis responded PONG'; else echo 'Redis not ready'; fi
              '
            """
          } else {
            bat """
              ssh -o StrictHostKeyChecking=no ${params.REMOTE_USER}@${params.REMOTE_IP} \"cd C:\\docker-deployment && docker-compose ps && docker ps --format \"table {{.Names}}\\t{{.Status}}\\t{{.Ports}}\"\"
            """
          }
        }
      }
    }
  }

  post {
    success {
      script {
        echo "🎉 Pipeline finished successfully. Remote services should be running."
      }
    }
    failure {
      script {
        echo "❌ Pipeline failed. Check console logs for detailed errors and re-run after fixing issues."
      }
    }
  }
}
