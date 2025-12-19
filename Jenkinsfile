pipeline {
    agent any

    parameters {
        choice(
            name: 'REMOTE_OS',
            choices: ['LINUX', 'WINDOWS'],
            description: 'Target machine operating system'
        )
        string(name: 'TARGET_IP', defaultValue: '', description: 'Remote machine IP')
        string(name: 'SSH_USER', defaultValue: 'om', description: 'SSH Username')
        password(name: 'SSH_PASS', defaultValue: '', description: 'SSH Password')
        string(name: 'COMPOSE_FILE', defaultValue: 'docker-compose.yml', description: 'Compose file')

        booleanParam(
            name: 'RESTORE_DB',
            defaultValue: false,
            description: '⚠️ Restore Postgres DB from latest backup (OVERWRITES DATA)'
        )
    }

    stages {

        stage('Validate Inputs') {
            steps {
                script {
                    echo "🔍 Validating inputs"
                    if (!params.TARGET_IP) error "TARGET_IP is required"
                    if (!params.SSH_USER) error "SSH_USER is required"
                    if (!params.SSH_PASS) error "SSH_PASS is required"
                    if (!fileExists(params.COMPOSE_FILE)) error "Compose file not found"
                    echo "✅ Inputs validated"
                }
            }
        }

        stage('SSH Check') {
            steps {
                sh """
                echo "🔐 Checking SSH connectivity"
                which sshpass >/dev/null || exit 2
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} 'echo SSH_OK'
                """
            }
        }

        /* ===================== LINUX ===================== */

        stage('Ensure Docker & Compose (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                echo "🐳 Ensuring Docker on Linux host"
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    set -e
                    if ! command -v docker >/dev/null 2>&1; then
                        echo "⬇️ Installing Docker"
                        curl -fsSL https://get.docker.com | sudo sh
                    fi
                    sudo systemctl enable docker
                    sudo systemctl start docker
                    sudo usermod -aG docker ${params.SSH_USER}
                    docker compose version
                '
                """
            }
        }

        stage('Copy Compose File (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                echo "📦 Copying compose file (Linux)"
                sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                ${params.COMPOSE_FILE} \
                ${params.SSH_USER}@${params.TARGET_IP}:~/docker-compose.yml
                """
            }
        }

        stage('Deploy Containers (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                echo "🚀 Deploying containers (Linux)"
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    cd ~
                    docker compose up -d --remove-orphans
                '
                """
            }
        }

        stage('Verify Containers (Linux)') {
            when { expression { params.REMOTE_OS == 'LINUX' } }
            steps {
                sh """
                echo "🔎 Verifying containers (Linux)"
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} "docker ps"
                """
            }
        }

        /* ===================== WINDOWS (via WSL Docker) ===================== */

        stage('Verify Docker (Windows via WSL)') {
            when { expression { params.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                echo "🐳 Verifying Docker (Windows)"
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    docker version
                    docker compose version
                '
                """
            }
        }
        
        stage('Copy Compose File (Windows via WSL)') {
            when { expression { params.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                echo "📦 Copying compose file (Windows)"
                sshpass -p '${params.SSH_PASS}' scp -o StrictHostKeyChecking=no \
                ${params.COMPOSE_FILE} \
                ${params.SSH_USER}@${params.TARGET_IP}:~/docker-compose.yml
                """
            }
        }
        
        stage('Deploy Containers (Windows via WSL)') {
            when { expression { params.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                echo "🚀 Deploying containers (Windows)"
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} '
                    cd ~
                    docker compose down --remove-orphans
                    docker compose pull
                    docker compose up -d
                '
                """
            }
        }
        
        stage('Verify Containers (Windows via WSL)') {
            when { expression { params.REMOTE_OS == 'WINDOWS' } }
            steps {
                sh """
                echo "🔎 Verifying containers (Windows)"
                sshpass -p '${params.SSH_PASS}' ssh -o StrictHostKeyChecking=no \
                ${params.SSH_USER}@${params.TARGET_IP} 'docker ps'
                """
            }
        }

        /* ===================== BACKUP ===================== */

       stage('Postgres Backup') {
            steps {
                sh '''
                echo "💾 Starting Postgres backup"
                sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no \
                "$SSH_USER@$TARGET_IP" '
                    if docker ps --format "{{.Names}}" | grep -q postgres_db; then
                        FILE=/backup/appdb_$(date +%F_%H-%M).sql
                        docker exec postgres_db sh -c "pg_dump -U admin appdb > $FILE"
                        echo "✅ Backup created: $FILE"
                    else
                        echo "⚠️ Postgres container not running, backup skipped"
                    fi
                '
                '''
            }
        }






        /* ===================== RESTORE ===================== */

        stage('Postgres Restore (Manual)') {
            when { expression { params.RESTORE_DB == true } }
            steps {
                sh '''
                echo "⚠️ RESTORE ENABLED — DATA WILL BE OVERWRITTEN"
                sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no \
                "$SSH_USER@$TARGET_IP" '
                    set -e
                    BACKUP_FILE=$(docker exec postgres_db ls -t /backup/appdb_*.sql | head -n 1)
        
                    if [ -z "$BACKUP_FILE" ]; then
                        echo "❌ No backup file found"
                        exit 1
                    fi
        
                    echo "📂 Restoring from $BACKUP_FILE"
        
                    docker exec postgres_db psql -U admin -d appdb \
                      -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
        
                    docker exec postgres_db sh -c "psql -U admin appdb < $BACKUP_FILE"
        
                    echo "✅ Restore completed successfully"
                '
                '''
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline completed successfully"
        }
        failure {
            echo "❌ Pipeline failed — check logs (data not auto-deleted)"
        }
    }
}
