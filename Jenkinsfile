pipeline {
  agent any

  parameters {
    string(name: 'TARGET_IP', defaultValue: '', description: 'Target machine IP')
    string(name: 'TARGET_USER', defaultValue: 'admin', description: 'Username to login')
    password(name: 'TARGET_PASSWORD', defaultValue: '', description: 'Password for SSH/PowerShell')
  }

  environment {
    DETECTED_OS = ''
  }

  stages {

    stage('Validate Input') {
      steps {
        script {
          if (!params.TARGET_IP?.trim()) {
            error "TARGET_IP आवश्यक आहे."
          }
        }
      }
    }

    stage('Ping Target') {
      steps {
        script {
          if (sh(returnStatus: true, script: "ping -c 2 ${params.TARGET_IP}") != 0) {
            error "❌ Target ping होत नाही."
          }
          echo "✔ Target reachable."
        }
      }
    }

    stage('Check SSH with Password') {
      steps {
        script {
          def USER = params.TARGET_USER
          def PASS = params.TARGET_PASSWORD

          def ok = sh(returnStatus: true, script: """
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 ${USER}@${params.TARGET_IP} "echo OK" 2>/dev/null
          """)

          if (ok == 0) {
            echo "✔ SSH password login working."
          } else {
            echo "⚠ SSH not responding — checking if Windows without OpenSSH..."
          }
        }
      }
    }

    stage('Detect OS') {
      steps {
        script {
          def USER = params.TARGET_USER
          def PASS = params.TARGET_PASSWORD

          def os = sh(returnStdout:true, script: """
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "uname -s" 2>/dev/null || echo WINDOWS
          """).trim()

          env.DETECTED_OS = os.toLowerCase().contains("linux") ? "LINUX" : "WINDOWS"

          echo "✔ Detected OS: ${env.DETECTED_OS}"
        }
      }
    }

    stage('Windows: Install OpenSSH If Missing') {
      when { expression { env.DETECTED_OS == "WINDOWS" } }
      steps {
        script {
          def USER = params.TARGET_USER
          def PASS = params.TARGET_PASSWORD

          echo "🔍 Checking if Windows OpenSSH is installed..."

          // Check & Install OpenSSH Server
          sh """
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
              powershell.exe -Command \\\"
                \$pkg = Get-WindowsCapability -Online | Where-Object { \$_.Name -like 'OpenSSH.Server*' };

                if (\$pkg.State -ne 'Installed') {
                  Write-Output '➡ Installing OpenSSH Server...';
                  Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0;
                } else {
                  Write-Output '✔ OpenSSH already installed.';
                }
              \\\"
            "
          """

          // Enable & Start sshd
          sh """
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
              powershell.exe -Command \\\"
                Set-Service sshd -StartupType Automatic;
                Start-Service sshd;
              \\\"
            "
          """

          // Open firewall port 22
          sh """
            sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
              powershell.exe -Command \\\"
                if (-not (Get-NetFirewallRule -DisplayName 'OpenSSH Port 22' -ErrorAction SilentlyContinue)) {
                  New-NetFirewallRule -DisplayName 'OpenSSH Port 22' -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow;
                }
              \\\"
            "
          """

          echo "✔ Windows OpenSSH installation complete!"
        }
      }
    }

    stage('Run OS Specific Commands') {
      steps {
        script {
          def USER = params.TARGET_USER
          def PASS = params.TARGET_PASSWORD

          if (env.DETECTED_OS == "LINUX") {

            echo "➡ Running Linux Commands..."
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                echo 'Connected to Linux via password!'
              "
            """

          } else {

            echo "➡ Running Windows Commands..."
            sh """
              sshpass -p '${PASS}' ssh -o StrictHostKeyChecking=no ${USER}@${params.TARGET_IP} "
                powershell.exe -Command \\\"
                  Write-Output 'Connected to Windows using password!'
                \\\"
              "
            """

          }
        }
      }
    }

  }

  post {
    success {
      echo "🎉 Pipeline Completed Successfully (Password Only Mode)"
    }
    failure {
      echo "❌ Pipeline Failed — तपशील वर पाहा."
    }
  }
}
