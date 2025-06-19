pipeline {
  agent any

  parameters {
    string(name: 'ACE_VERSION', defaultValue: '12.0.12.0', description: 'ACE installation version')
    string(name: 'INTEGRATION_NODE', defaultValue: 'Node_2', description: 'Integration Node Name')
    string(name: 'INTEGRATION_SERVER', defaultValue: 'Server_1', description: 'Integration Server Name')
    string(name: 'BRANCH', defaultValue: 'master', description: 'Git branch to build')
    choice(name: 'OS_TYPE', choices: ['windows', 'linux'], description: 'Operating System Type')
  }

  environment {
    REPO_URL = 'https://github.com/asharkhan11/IIB.git'
  }

  stages {
    stage('Init') {
      steps {
        script {
          def sep = params.OS_TYPE == 'windows' ? '\\' : '/'
          def aceHome = params.OS_TYPE == 'windows'
                        ? "C:\\Program Files\\IBM\\ACE\\${params.ACE_VERSION}"
                        : "/home/ace/ace-${params.ACE_VERSION}"

          env.WORKSPACE_ROOT = env.WORKSPACE
          env.BAR_OUTPUT_DIR = "${env.WORKSPACE}${sep}bars"
          env.ACE_HOME       = aceHome
          env.CREATEBAR_EXE  = "${aceHome}${params.OS_TYPE == 'windows' ? '\\tools\\mqsicreatebar.exe' : '/tools/mqsicreatebar'}"
          env.DEPLOY_EXE     = "${aceHome}${params.OS_TYPE == 'windows' ? '\\server\\bin\\mqsideploy.exe' : '/server/bin/mqsideploy'}"
          env.MQSIPROFILE    = "${aceHome}${params.OS_TYPE == 'windows' ? '\\server\\bin\\mqsiprofile.cmd' : '/server/bin/mqsiprofile'}"
          
          if (params.OS_TYPE == 'windows') {
            env.OSSL_MODULES = "${aceHome}\\server\\lib\\ossl-modules"
          }

          echo "🔧 OS: ${params.OS_TYPE}"
          echo "🔧 ACE_HOME: ${env.ACE_HOME}"
          echo "🔧 BAR_OUTPUT_DIR: ${env.BAR_OUTPUT_DIR}"
        }
      }
    }

    stage('Checkout') {
      steps {
        git url: "${env.REPO_URL}", branch: "${params.BRANCH}"
      }
    }

    stage('Detect Apps') {
      steps {
        script {
          def changed = []
          if (params.OS_TYPE == 'windows') {
            changed = powershell(returnStdout: true, script: '''
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

          } else {
            changed = sh(returnStdout: true, script: '''
git fetch origin master
git diff --name-only origin/master...HEAD | grep '^APPS/' | cut -d/ -f2 | sort -u
''').trim().split("\n").findAll { it }

            if (!changed) {
              changed = sh(returnStdout: true, script: '''
ls -d APPS/*/ | xargs -n1 basename
''').trim().split("\n").findAll { it }
            }
          }

          changed = changed.findAll { app -> !app.startsWith('.') }

          if (!changed) {
            error "❌ No valid apps found under APPS/"
          }

          env.APPS = changed.join(',')
          echo "🔍 Will build apps: ${env.APPS}"
        }
      }
    }

    stage('Prepare Output Directory') {
      steps {
        script {
          if (params.OS_TYPE == 'windows') {
            bat "if not exist \"${env.BAR_OUTPUT_DIR}\" mkdir \"${env.BAR_OUTPUT_DIR}\""
          } else {
            sh "mkdir -p \"${env.BAR_OUTPUT_DIR}\""
          }
        }
      }
    }

    stage('Build & Deploy') {
      steps {
        script {
          def apps = env.APPS.split(',')
          for (app in apps) {
            def barFile = "${env.BAR_OUTPUT_DIR}${params.OS_TYPE == 'windows' ? '\\' : '/'}${app}.bar"

            echo "📦 Creating BAR for ${app}"

            if (params.OS_TYPE == 'windows') {
              bat """
CALL "${env.MQSIPROFILE}"
"${env.CREATEBAR_EXE}" ^
  -data "%WORKSPACE%\\APPS" ^
  -b "${barFile}" ^
  -a "${app}" ^
  -p "${app}"
"""

              echo "🚀 Deploying ${app}.bar to ${params.INTEGRATION_NODE}/${params.INTEGRATION_SERVER}"

              bat """
CALL "${env.MQSIPROFILE}"
set OPENSSL_MODULES=${env.OSSL_MODULES}
"${env.DEPLOY_EXE}" ^
  "${params.INTEGRATION_NODE}" ^
  -e "${params.INTEGRATION_SERVER}" ^
  -a "${barFile}"
"""
            } else {
              sh """
. "${env.MQSIPROFILE}"
"${env.CREATEBAR_EXE}" \
  -data "$WORKSPACE/APPS" \
  -b "${barFile}" \
  -a "${app}" \
  -p "${app}"
"""

              echo "🚀 Deploying ${app}.bar to ${params.INTEGRATION_NODE}/${params.INTEGRATION_SERVER}"

              sh """
. "${env.MQSIPROFILE}"
"${env.DEPLOY_EXE}" \
  "${params.INTEGRATION_NODE}" \
  -e "${params.INTEGRATION_SERVER}" \
  -a "${barFile}"
"""
            }
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
