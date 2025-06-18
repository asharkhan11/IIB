pipeline {
  agent any
  environment {
    REPO_URL           = 'https://github.com/asharkhan11/IIB.git'
    BAR_OUTPUT_DIR     = "${env.WORKSPACE}\\bars"

    // ACE workspace isn't needed for deploy if you're pointing directly at the MQSI component folders:
    ACE_CREATEBAR_EXE  = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\tools\\mqsicreatebar.exe'
    ACE_DEPLOY_EXE     = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\server\\bin\\mqsideploy.exe'

    // Integration node & server names
    INTEGRATION_NODE   = 'node13'
    INTEGRATION_SERVER = 'server13'

    // Paths for your integration node & server
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
          // Try to detect changed apps; fallback to all
          def changed = powershell(returnStdout: true, script: '''
git fetch origin master
git diff --name-only origin/master...HEAD |
  Where-Object { $_ -like 'APPS/*' } |
  ForEach-Object { ($_ -split '/')[1] } |
  Sort-Object -Unique
''').trim().split("\r\n").findAll{ it }

          if (!changed) {
            changed = powershell(returnStdout: true, script: '''
Get-ChildItem -Path 'APPS' -Directory | ForEach-Object { $_.Name }
''').trim().split("\r\n").findAll{ it }
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

    stage('Build & Deploy') {
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
"${ACE_DEPLOY_EXE}" ^
  --integration-node "${INTEGRATION_NODE}" ^
  --integration-node-path "${NODE_PATH}" ^
  --integration-server "${INTEGRATION_SERVER}" ^
  --integration-server-path "${SERVER_PATH}" ^
  --bar-file "${barFile}"
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
