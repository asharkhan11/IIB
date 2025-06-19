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
    REPO_URL       = 'https://github.com/asharkhan11/IIB.git'
    WORKSPACE_ROOT = "${env.WORKSPACE}"
    BAR_OUTPUT_DIR = "${params.OS_TYPE}" == 'windows' ? "${env.WORKSPACE}\\bars" : "${env.WORKSPACE}/bars"

    ACE_HOME       = "${params.OS_TYPE}" == 'windows' ? "C:\\Program Files\\IBM\\ACE\\${params.ACE_VERSION}" : "/home/ace/ace-${params.ACE_VERSION}"
    CREATEBAR_EXE  = "${ACE_HOME}${params.OS_TYPE == 'windows' ? '\\tools\\mqsicreatebar.exe' : '/tools/mqsicreatebar'}"
    DEPLOY_EXE     = "${ACE_HOME}${params.OS_TYPE == 'windows' ? '\\server\\bin\\mqsideploy.exe' : '/server/bin/mqsideploy'}"
    MQSIPROFILE    = "${ACE_HOME}${params.OS_TYPE == 'windows' ? '\\server\\bin\\mqsiprofile.cmd' : '/server/bin/mqsiprofile'}"
    OSSL_MODULES   = "${ACE_HOME}${params.OS_TYPE == 'windows' ? '\\server\\lib\\ossl-modules' : '/server/lib/ossl-modules'}"
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
          def changed = ""
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
git diff --name-only origin/master...HEAD | \
  grep '^APPS/' | cut -d/ -f2 | sort -u
''').trim().split("\n").findAll { it }

            if (!changed) {
              changed = sh(returnStdout: true, script: '''
ls -d APPS/*/ | xargs -n1 basename
''').trim().split("\n").findAll { it }
            }
          }

          changed = changed.findAll { app -> !app.startsWith('.') }

          if (!changed) {
            error "No valid apps found under APPS/"
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
            bat "if not exist \"${BAR_OUTPUT_DIR}\" mkdir \"${BAR_OUTPUT_DIR}\""
          } else {
            sh "mkdir -p \"${BAR_OUTPUT_DIR}\""
          }
        }
      }
    }

    stage('Build & Deploy') {
      steps {
        script {
          def apps = env.APPS.split(',')
          for (app in apps) {
            def barFile = "${BAR_OUTPUT_DIR}${params.OS_TYPE == 'windows' ? '\\' : '/'}${app}.bar"

            echo "📦 Creating BAR for ${app}"

            if (params.OS_TYPE == 'windows') {
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
            } else {
              sh """
. "${MQSIPROFILE}"
"${CREATEBAR_EXE}" \
  -data "$WORKSPACE/APPS" \
  -b "${barFile}" \
  -a "${app}" \
  -p "${app}"
"""

              echo "🚀 Deploying ${app}.bar to ${params.INTEGRATION_NODE}/${params.INTEGRATION_SERVER}"

              sh """
. "${MQSIPROFILE}"
export OPENSSL_MODULES="${OSSL_MODULES}"
"${DEPLOY_EXE}" \
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

