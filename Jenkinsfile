pipeline {
  agent any
  environment {
    REPO_URL           = 'https://github.com/asharkhan11/IIB.git'
    BAR_OUTPUT_DIR     = "${env.WORKSPACE}\\bars"

    // ACE 13 paths for bar creation and deploy
    ACE_CREATEBAR_EXE  = 'C:\\Program Files\\IBM\\ACE\\12.0.2.0\\tools\\mqsicreatebar.exe'
    ACE_DEPLOY_EXE     = 'C:\\Program Files\\IBM\\ACE\\12.0.2.0\\server\\bin\\mqsideploy.exe'

    INTEGRATION_NODE   = 'Node_2'
    INTEGRATION_SERVER = 'Server_1'

    NODE_PATH          = 'C:\\ProgramData\\IBM\\MQSI\\components\\node13'
    SERVER_PATH        = 'C:\\ProgramData\\IBM\\MQSI\\components\\node13\\servers\\server13'
  }

  stages {
    stage('Checkout') {
      steps {
        git url: "${REPO_URL}", branch: 'master'
      }
    }

    stage('Detect Apps') {
      steps {
        script {
          def changed = powershell(returnStdout: true, script: '''
git fetch origin master
git diff --name-only origin/master...HEAD |
  Where-Object { $_ -like 'APPS/*' } |
  ForEach-Object { ($_ -split '/')[1] } |
  Sort-Object -Unique
''').trim().split("\r\n").findAll { it }

          if (!changed) {
            changed = powershell(returnStdout: true, script: '''
Get-ChildItem -Path 'APPS' -Directory |
  Where-Object { -not $_.Name.StartsWith('.') } |
  ForEach-Object { $_.Name }
''').trim().split("\r\n").findAll { it }
          }

          changed = changed.findAll { app -> !app.startsWith('.') }

          if (!changed) {
            error "No valid apps found under APPS\\"
          }

          env.APPS = changed.join(',')
          echo "🔍 Will build apps: ${env.APPS}"
        }
      }
    }

    stage('Prepare Output Directory') {
      steps {
        bat "if not exist \"${BAR_OUTPUT_DIR}\" mkdir \"${BAR_OUTPUT_DIR}\""
      }
    }

    stage('Build & Deploy (ACE 12)') {
      steps {
        script {
          for (app in env.APPS.split(',')) {
            def barFile = "${BAR_OUTPUT_DIR}\\${app}.bar"

            echo "📦 Creating BAR for ${app}"
            bat """
"${ACE_CREATEBAR_EXE}" ^
  -data "%WORKSPACE%\\APPS" ^
  -b "${barFile}" ^
  -a "${app}" ^
  -p "${app}"
"""

 echo "🚀 Deploying ${app}.bar to ${INTEGRATION_NODE}/${INTEGRATION_SERVER}"
  bat """
  CALL "C:\\Program Files\\IBM\\ACE\\12.0.12.0\\server\\bin\\mqsiprofile.cmd"
  set OPENSSL_MODULES=C:\\Program Files\\IBM\\ACE\\12.0.12.0\\server\\lib\\ossl-modules
    
"${ACE_DEPLOY_EXE}" ^
  "${INTEGRATION_NODE}" ^
  -e "${INTEGRATION_SERVER}" ^
  -a "${barFile}"
"""
          }
        }
      }
    }

  }

  post {
    always {
      echo "✅ Done."
    }
  }
}
