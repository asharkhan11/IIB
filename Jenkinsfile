pipeline {
  agent any
  environment {
    REPO_URL           = 'https://github.com/asharkhan11/IIB.git'
    BAR_OUTPUT_DIR     = "${env.WORKSPACE}\\bars"
    ACE_BIN            = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\server\\bin\\mqsideploy.exe'
    ACE_TOOLS          = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\tools\\mqsicreatebar.exe'
    INTEGRATION_SERVER = 'default'
  }

  stages {
    stage('Checkout') {
      steps {
        git url: "${REPO_URL}", branch: 'main'
      }
    }

    stage('Detect Changed App') {
      steps {
        script {
          // Run a PowerShell snippet to list unique subfolders under APPS\ changed in this push
          def changed = powershell(
            returnStdout: true,
            script: '''
              git fetch origin main
              git diff --name-only origin/main...HEAD |
                Where-Object { $_ -like 'APPS/*' } |
                ForEach-Object { ($_ -split '/')[1] } |
                Sort-Object -Unique
            '''
          ).trim().split("\r\n")

          if (changed.size() == 0 || changed[0] == '') {
            error "No changes detected under APPS\\ – nothing to build"
          }
          env.APP_NAME = changed[0]
          echo "🔍 Detected changed app: ${env.APP_NAME}"
        }
      }
    }

    stage('Prepare Output Directory') {
      steps {
        bat "if not exist \"${BAR_OUTPUT_DIR}\" mkdir \"${BAR_OUTPUT_DIR}\""
      }
    }

    stage('Build BAR') {
      steps {
        script {
          def appDir  = "APPS\\${env.APP_NAME}"
          def barFile = "${BAR_OUTPUT_DIR}\\${env.APP_NAME}.bar"

          echo "📦 Building BAR: ${barFile}"
          bat """
            "\"${ACE_TOOLS}\"" ^
              -data "%WORKSPACE%" ^
              -b "${barFile}" ^
              -a "${appDir}" ^
              -k "${appDir}"
          """
        }
      }
    }

    stage('Deploy to ACE') {
      steps {
        script {
          def barFile = "${BAR_OUTPUT_DIR}\\${env.APP_NAME}.bar"
          echo "🚀 Deploying ${barFile} to server '${INTEGRATION_SERVER}'"
          bat """
            "\"${ACE_BIN}\"" ^
              -i localhost ^
              -e ${INTEGRATION_SERVER} ^
              -a "${barFile}" ^
              -m
          """
        }
      }
    }
  }

  post {
    always {
      echo "✅ Pipeline finished."
    }
  }
}
