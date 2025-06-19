pipeline {
  agent any

  parameters {
    string(name: 'ACE_VERSION', defaultValue: '12.0.12.0', description: 'ACE installation version')
    string(name: 'INTEGRATION_NODE', defaultValue: 'Node_1', description: 'Integration Node Name')
    string(name: 'INTEGRATION_SERVER', defaultValue: 'Server_1', description: 'Integration Server Name')
    string(name: 'BRANCH', defaultValue: 'master', description: 'Git branch to build')
  }

  environment {
    REPO_URL       = 'https://github.com/asharkhan11/IIB.git'
    WORKSPACE_ROOT = "${env.WORKSPACE}"
    BAR_OUTPUT_DIR = "${WORKSPACE_ROOT}\\bars"

    ACE_HOME       = "C:\\Program Files\\IBM\\ACE\\${params.ACE_VERSION}"
    CREATEBAR_EXE  = "${ACE_HOME}\\tools\\mqsicreatebar.exe"
    DEPLOY_EXE     = "${ACE_HOME}\\server\\bin\\mqsideploy.exe"
    MQSIPROFILE    = "${ACE_HOME}\\server\\bin\\mqsiprofile.cmd"
    OSSL_MODULES   = "${ACE_HOME}\\server\\lib\\ossl-modules"
  }

  stages {
    stage('Checkout') {
      steps {
        git url: "${REPO_URL}", branch: "${params.BRANCH}"
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

    stage('Build & Deploy') {
      steps {
        script {
          for (app in env.APPS.split(',')) {
            def barFile = "${BAR_OUTPUT_DIR}\\${app}.bar"

            echo "📦 Creating BAR for ${app}"
            bat """
CALL "${MQSIPROFILE}"
"${CREATEBAR_EXE}" ^
  -data "%WORKSPACE%\\APPS" ^
  -b "${barFile}" ^
  -a "${app}" ^
  -p "${app}"
"""

            echo "🚀 Deploying ${app}.bar to ${params.INTEGRATION_NODE}/${params.INTEGRATION_SERVER}"
            bat """
CALL "${MQSIPROFILE}"
set OPENSSL_MODULES=${OSSL_MODULES}
"${DEPLOY_EXE}" ^
  "${params.INTEGRATION_NODE}" ^
  -e "${params.INTEGRATION_SERVER}" ^
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
