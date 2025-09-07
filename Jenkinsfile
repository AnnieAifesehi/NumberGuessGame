pipeline {
  agent any
  tools { jdk 'Java-17'; maven 'Maven' }
  options { timestamps(); ansiColor('xterm'); disableConcurrentBuilds() }

  environment {
    REPO_URL         = 'https://github.com/AnnieAifesehi/NumberGuessGame.git'
    REPO_BRANCH      = 'main'

    // --- SonarQube ---
    SONARQUBE_SERVER = 'SonarQube'   // Jenkins global server name (Manage Jenkins ➜ System)

    // --- Nexus (use settings.xml with <server id="nexus-releases">) ---
    NEXUS_URL        = 'http://54.157.3.135:8081'
    NEXUS_REPO_ID    = 'nexus-releases'       // must match <server id> in settings.xml
    NEXUS_REPO_PATH  = 'maven-releases'       // hosted repo name in Nexus
    MVN_SETTINGS_ID  = 'maven-settings-nexus' // Config File Provider ID for settings.xml

    // --- Tomcat target host ---
    TOMCAT_HOST      = '3.210.219.27'
    TOMCAT_USER      = 'ec2-user'
    TOMCAT_SSH_ID    = 'tomcat-ssh'
    TOMCAT_WEBAPPS   = '/opt/tomcat/webapps'
    APP_NAME         = 'NumberGuessGame'

    // optional app check
    HEALTH_URL       = "http://${TOMCAT_HOST}:8080/${APP_NAME}/"
  }

  stages {
    stage('Checkout') {
      steps { git branch: env.REPO_BRANCH, url: env.REPO_URL }
    }

    stage('Build') {
      steps {
        withMaven(maven: 'Maven') {
          sh 'mvn -B -DskipTests clean package'
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv(env.SONARQUBE_SERVER) {
          sh 'mvn -B sonar:sonar'
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          // requires SonarQube webhook back to Jenkins
          timeout(time: 10, unit: 'MINUTES') {
            def qg = waitForQualityGate()  // aborts pipeline on failure
            if (qg.status != 'OK') {
              error "Quality Gate failed: ${qg.status}"
            }
          }
        }
      }
    }

    stage('Publish to Nexus') {
      steps {
        configFileProvider([configFile(fileId: env.MVN_SETTINGS_ID, variable: 'MVN_SETTINGS')]) {
          withMaven(globalMavenSettingsConfig: '', mavenSettingsConfig: env.MVN_SETTINGS_ID) {
            sh """
              mvn -B -s "$MVN_SETTINGS" -DskipTests deploy \
                -DaltDeploymentRepository=${env.NEXUS_REPO_ID}::default::${env.NEXUS_URL}/repository/${env.NEXUS_REPO_PATH}/
            """
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        script {
          // Obtain the built WAR path + version from Maven
          def war = sh(script: "ls -1 target/*.war | head -n1", returnStdout: true).trim()
          echo "WAR: ${war}"

          sshagent([env.TOMCAT_SSH_ID]) {
            sh """
              set -e
              scp -o StrictHostKeyChecking=no "${war}" ${TOMCAT_USER}@${TOMCAT_HOST}:/tmp/app.war

              ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} '
                set -e
                sudo systemctl stop tomcat || true
                sudo rm -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war
                sudo rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME}
                sudo cp /tmp/app.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war
                sudo chown -R tomcat:tomcat ${TOMCAT_WEBAPPS}
                sudo systemctl start tomcat
              '
            """
          }
        }
      }
    }

    stage('Post-Deploy Check') {
      when { expression { return env.HEALTH_URL?.trim() } }
      steps {
        // gentle wait for Tomcat to explode the WAR
        sh """
          echo "Waiting for app to come up: ${HEALTH_URL}"
          for i in {1..30}; do
            if curl -fsSL "${HEALTH_URL}" >/dev/null 2>&1; then
              echo "App is up"; exit 0
            fi
            sleep 2
          done
          echo "Health check failed"; exit 1
        """
      }
    }
  }

  post {
    success { echo 'Build, scan, publish, and deploy completed.' }
    failure { echo 'Pipeline failed. Check logs for the failing stage.' }
    always  { archiveArtifacts artifacts: '**/target/*.war, **/target/site/**', fingerprint: true, allowEmptyArchive: true }
  }
}
