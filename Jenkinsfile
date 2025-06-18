pipeline {
  agent any
  environment {
    REPO_URL           = 'https://github.com/asharkhan11/IIB.git'
    BAR_OUTPUT_DIR     = "${env.WORKSPACE}\\bars"
    ACE_WS             = 'C:\\Users\\Sreenivas Bandaru\\IBM\\ACET13\\workspace\\TEST_SERVER'
    ACE_CREATEBAR_EXE  = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\tools\\mqsicreatebar.exe'
    ACE_DEPLOY_EXE     = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\server\\bin\\mqsideploy.exe'
    INTEGRATION_SERVER = 'TEST_SERVER'
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
          // Fallback to all apps if no diffs
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

    stage('Build & Deploy') {
      steps {
        script {
          bat "if not exist \"${BAR_OUTPUT_DIR}\" mkdir \"${BAR_OUTPUT_DIR}\""

          for (app in env.APPS.split(',')) {
            def bar = "${BAR_OUTPUT_DIR}\\${app}.bar"

            echo "📦 Creating BAR for ${app} from ACE workspace ${ACE_WS}"
            bat """
\"${ACE_CREATEBAR_EXE}\" ^
  -data "${ACE_WS}" ^
  -b "${bar}" ^
  -a "${app}" ^
  -p "${app}"
"""

            echo "🚀 Deploying ${app}.bar to integration server in ${ACE_WS}"
            bat """
\"${ACE_DEPLOY_EXE}\" ^
   node13 --integration-server server13  ^
  --bar-file "${bar}"
"""
          }
        }
      }
    }
  }

  post {
    always { echo "✅ Done." }
  }
}
