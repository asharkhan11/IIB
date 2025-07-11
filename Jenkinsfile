pipeline {
  agent any

  parameters {
    string(name: 'ACE_VERSION', defaultValue: '12.0.12.0', description: 'ACE installation version')
    string(name: 'INTEGRATION_NODE', defaultValue: 'Node_2', description: 'Integration Node Name')
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

    stage('Check Changed Files') {
  steps {
    script {
      def changedFiles = bat(
        script: 'git diff --name-only HEAD~1 HEAD',
        returnStdout: true
      ).trim().split("\r?\n")

      def shouldBuild = changedFiles.any { it == 'applists' }

      if (!shouldBuild) {
        echo "🛑 Skipping pipeline - 'applists' file was not changed."
        currentBuild.result = 'NOT_BUILT'
        error('No relevant changes detected.')
      } else {
        echo "✅ Detected change in applists, continuing build."
      }
    }
  }
}


    stage('Read App List') {
      steps {
        script {
          def raw = readFile(file: 'applists').trim()
          def appList = raw.split("\n").collect { it.trim() }.findAll { it && !it.startsWith('#') }

          if (appList.isEmpty()) {
            error "❌ 'applists' file is empty or invalid."
          }

          env.APPS = appList.join(',')
          echo "📄 Apps from applists file: ${env.APPS}"
        }
      }
    }

    stage('Prepare Output Directory') {
      steps {
        bat "if not exist \"${BAR_OUTPUT_DIR}\" mkdir \"${BAR_OUTPUT_DIR}\""
      }
    }

    stage('Build BARs') {
      steps {
        script {
          for (app in env.APPS.split(',')) {
            def barFile = "${BAR_OUTPUT_DIR}\\${app}.bar"

            echo "📦 Building BAR for ${app}"
            bat """
CALL "${MQSIPROFILE}"
"${CREATEBAR_EXE}" ^
  -data "%WORKSPACE%\\APPS" ^
  -b "${barFile}" ^
  -a "${app}" ^
  -p "${app}"
"""
          }
        }
      }
    }

    stage('Deploy BARs') {
      steps {
        script {
          for (app in env.APPS.split(',')) {
            def barFile = "${BAR_OUTPUT_DIR}\\${app}.bar"

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
      echo "✅ Pipeline completed."
    }
  }
}
