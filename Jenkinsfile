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
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "uname -s" 2>/dev/null || true
          """).trim()

          env.DETECTED_OS = osCheck?.toLowerCase()?.contains("linux") ? "LINUX" : "WINDOWS"
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
          // WINDOWS SETUP — FULL AUTO INSTALL + RELIABLE KEY INJECTION
          // -----------------------------
          else {

            echo "✔ Windows detected — Auto-installing OpenSSH if required and injecting keys..."

            // Install OpenSSH Server if missing
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  Try {
                    \$cap = Get-WindowsCapability -Online | Where-Object { \$_.Name -like 'OpenSSH.Server*' };
                    if (-not \$cap -or \$cap.State -ne 'Installed') {
                      Write-Output 'Installing OpenSSH.Server...';
                      Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0;
                    } else {
                      Write-Output 'OpenSSH.Server already installed.';
                    }
                  } Catch {
                    Write-Output 'OpenSSH installation check/install failed: ' + \$_;
                    exit 1;
                  }
                \\\"
              "
            """

            // Enable & Start SSHD service
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  Set-Service -Name sshd -StartupType Automatic -ErrorAction SilentlyContinue;
                  Start-Service sshd -ErrorAction SilentlyContinue;
                \\\"
              "
            """

            // Firewall allow port 22
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  if (-not (Get-NetFirewallRule -DisplayName 'OpenSSH Port 22' -ErrorAction SilentlyContinue)) {
                      New-NetFirewallRule -DisplayName 'OpenSSH Port 22' -Direction Inbound -LocalPort 22 -Protocol TCP -Action Allow;
                  }
                \\\"
              "
            """

            // RELIABLE: write to ProgramData administrators_authorized_keys and user's authorized_keys; set permissions
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  Try {
                    \$adminKey = 'C:/ProgramData/ssh/administrators_authorized_keys';
                    if (!(Test-Path 'C:/ProgramData/ssh')) { New-Item -ItemType Directory -Path 'C:/ProgramData/ssh' -Force | Out-Null }

                    # Append key to admin file
                    '${env.SSH_PUB}' | Out-File -FilePath \$adminKey -Encoding ascii -Append

                    # Fix admin file permissions
                    icacls \$adminKey /inheritance:r /grant 'Administrators:F' /grant 'SYSTEM:F' | Out-Null

                    # User-level authorized_keys
                    \$userSsh = Join-Path \$env:USERPROFILE '.ssh';
                    if (!(Test-Path \$userSsh)) { New-Item -ItemType Directory -Path \$userSsh -Force | Out-Null }
                    '${env.SSH_PUB}' | Out-File -FilePath (Join-Path \$userSsh 'authorized_keys') -Encoding ascii -Append

                    # Fix user authorized_keys permissions
                    icacls (Join-Path \$userSsh 'authorized_keys') /inheritance:r /grant \$env:USERNAME':F' | Out-Null
                  } Catch {
                    Write-Output 'Failed to write key or set permissions: ' + \$_;
                    exit 1;
                  }
                \\\"
              "
            """

            echo "✔ Windows OpenSSH auto-install + reliable key injection complete."
          }
        }
      }
    }

    stage('Verify Passwordless SSH') {
      steps {
        script {
          // try several times briefly to allow service start
          def ok = sh(returnStatus: true, script: """
            for i in 1 2 3; do
              ssh -o BatchMode=yes -o StrictHostKeyChecking=no -o ConnectTimeout=4 ${params.TARGET_IP} "echo OK" && exit 0 || sleep 2
            done
            exit 1
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
