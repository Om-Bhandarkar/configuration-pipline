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
            error "TARGET_IP आवश्यक आहे — Jenkins job parameter मध्ये द्या."
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
          echo "Jenkins SSH Public Key Ready."
        }
      }
    }

    stage('Ping target') {
      steps {
        script {
          if (sh(returnStatus: true, script: "ping -c 2 ${params.TARGET_IP}") != 0) {
            error "Target ${params.TARGET_IP} not reachable."
          }
          echo "Target reachable."
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
            echo "Passwordless SSH works. Skipping bootstrap."
            env.NEED_BOOTSTRAP = "false"

            def linuxTest = sh(returnStatus: true, script: """
              ssh -o StrictHostKeyChecking=no ${params.TARGET_IP} "uname -s"
            """)

            env.DETECTED_OS = (linuxTest == 0) ? "LINUX" : "WINDOWS"
          }
        }
      }
    }

    stage('Password-Based Bootstrap') {
      when { expression { env.NEED_BOOTSTRAP == "true" } }
      steps {
        script {

          def TUSER = params.TARGET_USER
          def TPASS = params.TARGET_PASSWORD

          // Check password SSH
          def canSSH = sh(returnStatus: true, script: """
            sshpass -p "$TPASS" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=8 ${TUSER}@${params.TARGET_IP} "echo OK"
          """)

          if (canSSH != 0) {
            error "Password-based SSH failed."
          }

          // FIXED OS DETECTION LOGIC
          def output = sh(returnStdout:true, script: """
            sshpass -p "$TPASS" ssh -o StrictHostKeyChecking=no ${TUSER}@${params.TARGET_IP} "uname -s" || echo WINDOWS
          """).trim()

          if (output.toLowerCase().contains("linux")) {
            env.DETECTED_OS = "LINUX"
          } else {
            env.DETECTED_OS = "WINDOWS"
          }

          echo "Detected OS: ${env.DETECTED_OS}"

          // LINUX SETUP
          if (env.DETECTED_OS == "LINUX") {

            sh """
              sshpass -p "$TPASS" ssh -o StrictHostKeyChecking=no ${TUSER}@${params.TARGET_IP} '
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

            echo "Linux bootstrap complete."
          }

          // WINDOWS SETUP
          else {

            def ps = sh(returnStatus: true, script: """
              sshpass -p "$TPASS" ssh -o StrictHostKeyChecking=no ${TUSER}@${params.TARGET_IP} powershell.exe -Command "Write-Output OK"
            """)

            if (ps != 0) {
              error "Windows OpenSSH is not installed. Install it manually once."
            }

            // FIX: USED env.SSH_PUB instead of SSH_PUB
            sh """
              sshpass -p "$TPASS" ssh -o StrictHostKeyChecking=no ${TUSER}@${params.TARGET_IP} '
                powershell -Command "
                  \$d = Join-Path \$env:USERPROFILE '.ssh';
                  if (!(Test-Path \$d)) { New-Item -ItemType Directory -Path \$d };
                  Add-Content -Path (Join-Path \$d 'authorized_keys') -Value '${env.SSH_PUB}';
                "
              '
            """

            echo "Windows key setup complete."
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

          if (ok != 0) {
            error "Passwordless SSH still NOT working!"
          }

          echo "Passwordless SSH Verified ✔"
        }
      }
    }

  }

  post {
    success {
      echo "Pipeline Completed Successfully ✔ DETECTED_OS=${env.DETECTED_OS}"
    }
    failure {
      echo "Pipeline Failed ❌ — Please check logs."
    }
  }
}
