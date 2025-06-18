pipeline {
  agent any
  environment {
    REPO_URL           = 'https://github.com/asharkhan11/IIB.git'
    BAR_OUTPUT_DIR     = "${env.WORKSPACE}\\bars"
    ACE_DEPLOY_EXE     = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\server\\bin\\mqsideploy.exe'
    ACE_CREATEBAR_EXE  = 'C:\\Program Files\\IBM\\ACE\\13.0.3.0\\tools\\mqsicreatebar.exe'
    INTEGRATION_SERVER = 'default'
  }

  stages {
    stage('Checkout') {
      steps {
        git url: "${REPO_URL}", branch: 'master'
      }
    }

    stage('Detect Apps to Build') {
      steps {
        script {
          // 1. Try to detect changed apps
          def changed = powershell(
            returnStdout: true,
            script: '''
              # fetch remote so diff works
              git fetch origin master

              # list unique subfolders under APPS\ changed in this push
              git diff --name-only origin/master...HEAD |
                Where-Object { $_ -like 'APPS/*' } |
                ForEach-Object { ($_ -split '/')[1] } |
                Sort-Object -Unique
            '''
          ).trim().split("\r\n").findAll { it }

          if (changed.isEmpty()) {
            echo "No changes detected under APPS\\; building ALL apps."
            // fallback: list every directory under APPS\
            changed = powershell(
              returnStdout: true,
              script: '''
                Get-ChildItem -Path 'APPS' -Directory |
                  ForEach-Object { $_.Name }
              '''
            ).trim().split("\r\n").findAll { it }
          }

          echo "🔍 Apps to build: ${changed}"
          // stash into a Groovy variable for later stages
          env.APPS_TO_BUILD = changed.join(',')
        }
      }
    }

    stage('Prepare Output Directory') {
      steps {
        bat "if not exist \"${BAR_OUTPUT_DIR}\" mkdir \"${BAR_OUTPUT_DIR}\""
      }
    }

    stage('Build & Deploy BARs') {
      steps {
        script {
          def apps = env.APPS_TO_BUILD.split(',')
          for (app in apps) {
            def appDir  = "APPS\\${app}"
            def barFile = "${BAR_OUTPUT_DIR}\\${app}.bar"

            echo "📦 Building BAR for ${app}: ${barFile}"
            bat """
              "\"${ACE_CREATEBAR_EXE}\"" ^
                -data "%WORKSPACE%" ^
                -b "${barFile}" ^
                -a "${appDir}" ^
                -k "${appDir}"
            """

            echo "🚀 Deploying ${app}.bar to server '${INTEGRATION_SERVER}'"
            bat """
              "\"${ACE_DEPLOY_EXE}\"" ^
                -i localhost ^
                -e ${INTEGRATION_SERVER} ^
                -a "${barFile}" ^
                -m
            """
          }
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
