pipeline {
  agent any

  parameters {
    string(name: 'TARGET_IP', defaultValue: '', description: 'Target machine IP')
    string(name: 'TARGET_USER', defaultValue: 'admin', description: 'Target username')
    password(name: 'TARGET_PASSWORD', defaultValue: '', description: 'Target password')
  }

  environment {
    SSH_PUB = ''
    NEED_BOOTSTRAP = 'true'
    DETECTED_OS = ''
  }

  stages {

    stage('Validate input') {
      steps {
        script {
          if (!params.TARGET_IP?.trim()) {
            error "TARGET_IP आवश्यक आहे."
          }
        }
      }
    }

    stage('Generate Jenkins SSH key') {
      steps {
        script {
          sh """
            mkdir -p ${env.WORKSPACE}/.ssh
            if [ ! -f ${env.WORKSPACE}/.ssh/id_rsa ]; then
              ssh-keygen -t rsa -b 4096 -N "" -f ${env.WORKSPACE}/.ssh/id_rsa
            fi
          """
          env.SSH_PUB = sh(returnStdout:true, script: "cat ${env.WORKSPACE}/.ssh/id_rsa.pub").trim()
          echo "✔ Jenkins SSH Public Key Ready."
        }
      }
    }

    stage('Ping Target') {
      steps {
        script {
          if (sh(returnStatus: true, script: "ping -c 2 ${params.TARGET_IP}") != 0) {
            error "❌ Target not reachable."
          }
          echo "✔ Ping successful."
        }
      }
    }

    stage('Try Passwordless SSH') {
      steps {
        script {
          env.NEED_BOOTSTRAP = "true"

          def ok = sh(returnStatus: true, script: """
            ssh -o BatchMode=yes -o StrictHostKeyChecking=no -o ConnectTimeout=3 ${params.TARGET_IP} "echo OK"
          """)

          if (ok == 0) {
            echo "✔ Passwordless SSH already works. Skipping bootstrap."
            env.NEED_BOOTSTRAP = "false"

            def isLinux = sh(returnStatus: true, script: """
              ssh -o StrictHostKeyChecking=no ${params.TARGET_IP} "uname -s"
            """)

            env.DETECTED_OS = (isLinux == 0) ? "LINUX" : "WINDOWS"
          }
        }
      }
    }

    stage('Password-Based Bootstrap') {
      when { expression { env.NEED_BOOTSTRAP == "true" } }
      steps {
        script {
          def USER = params.TARGET_USER
          def PASS = params.TARGET_PASSWORD

          // Check SSH password login
          def canSSH = sh(returnStatus: true, script: """
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=8 ${USER}@${params.TARGET_IP} "echo OK"
          """)

          if (canSSH != 0) error "❌ Password SSH failed."

          // Detect OS
          def osCheck = sh(returnStdout:true, script: """
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "uname -s" 2>/dev/null
          """).trim()

          env.DETECTED_OS = osCheck.toLowerCase().contains("linux") ? "LINUX" : "WINDOWS"
          echo "✔ Detected OS = ${env.DETECTED_OS}"

          // -----------------------------
          // LINUX SETUP
          // -----------------------------
          if (env.DETECTED_OS == "LINUX") {

            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} '
                sudo apt-get update -y || sudo yum -y makecache fast;
                sudo apt-get install -y openssh-server ufw || sudo yum install -y openssh-server;
                sudo systemctl enable --now ssh || sudo systemctl enable --now sshd;
                sudo ufw allow 22/tcp || true;
                sudo ufw reload || true;
                mkdir -p ~/.ssh;
                echo "${env.SSH_PUB}" >> ~/.ssh/authorized_keys;
                chmod 600 ~/.ssh/authorized_keys;
              '
            """

            echo "✔ Linux bootstrap completed."
          }

          // -----------------------------
          // WINDOWS SETUP — FULL AUTO INSTALL
          // -----------------------------
          else {

            echo "✔ Windows detected — Auto-installing OpenSSH..."

            // Install OpenSSH Server if missing
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  if (-not (Get-WindowsCapability -Online | Where-Object { \$_.Name -like 'OpenSSH.Server*' -and \$_.State -eq 'Installed' })) {
                      Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0;
                  }
                \\\"
              "
            """

            // Enable & Start SSHD
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  Set-Service -Name sshd -StartupType Automatic;
                  Start-Service sshd;
                \\\"
              "
            """

            // Firewall allow port 22
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  if (!(Get-NetFirewallRule -DisplayName 'OpenSSH Port 22' -ErrorAction SilentlyContinue)) {
                      New-NetFirewallRule -DisplayName 'OpenSSH Port 22' -Direction Inbound -LocalPort 22 -Protocol TCP -Action Allow;
                  }
                \\\"
              "
            """

            // Add SSH Public Key
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  \$d = Join-Path \$env:USERPROFILE '.ssh';
                  if (!(Test-Path \$d)) { New-Item -ItemType Directory -Path \$d -Force };
                  Add-Content -Path (Join-Path \$d 'authorized_keys') -Value '${env.SSH_PUB}';
                \\\"
              "
            """

            echo "✔ Windows OpenSSH auto-install + key setup complete."
          }
        }
      }
    }

    stage('Verify Passwordless SSH') {
      steps {
        script {
          def ok = sh(returnStatus: true, script: """
            ssh -o BatchMode=yes -o StrictHostKeyChecking=no ${params.TARGET_IP} "echo OK"
          """)

          if (ok != 0) error "❌ Passwordless SSH still NOT working!"

          echo "✔ Passwordless SSH Verified!"
        }
      }
    }
  }

  post {
    success {
      echo "🎉 Pipeline Finished Successfully! OS = ${env.DETECTED_OS}"
    }
    failure {
      echo "❌ Pipeline Failed — Check logs."
    }
  }
}
