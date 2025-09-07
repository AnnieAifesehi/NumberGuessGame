pipeline {
  agent any
  tools { jdk 'Java-17'; maven 'Maven' }
  options { timestamps(); disableConcurrentBuilds() }

  environment {
    REPO_URL         = 'https://github.com/AnnieAifesehi/NumberGuessGame.git'
    REPO_BRANCH      = 'main'

    // SonarQube
    SONARQUBE_SERVER = 'SonarQube'

    // Nexus 2 base (NO trailing slash; path is added below)
    NEXUS_URL        = 'http://54.157.3.135:8081'

    // Jenkins creds for Nexus (Username/Password)
    NEXUS_CRED_ID    = 'nexus-cred'

    // Tomcat target
    TOMCAT_HOST      = '3.210.219.27'
    TOMCAT_USER      = 'ec2-user'
    TOMCAT_SSH_ID    = 'tomcat-ssh'
    TOMCAT_WEBAPPS   = '/opt/tomcat/webapps'
    TOMCAT_SERVICE   = 'tomcat'
    APP_NAME         = 'NumberGuessGame'

    HEALTH_URL       = "http://${TOMCAT_HOST}:8080/${APP_NAME}/"
  }

  stages {
    stage('Checkout') {
      steps { git branch: env.REPO_BRANCH, url: env.REPO_URL }
    }

    stage('Build') {
      steps { sh 'mvn -B -DskipTests clean package' }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv(env.SONARQUBE_SERVER) {
          sh 'mvn -B sonar:sonar'
        }
      }
    }

    stage('Publish to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.NEXUS_CRED_ID, usernameVariable: 'NU', passwordVariable: 'NP')]) {
          script {
            def version = sh(script: "mvn -q -DforceStdout help:evaluate -Dexpression=project.version", returnStdout: true).trim()
            def isSnapshot = version.endsWith('-SNAPSHOT')
            echo "Project version: ${version} (snapshot=${isSnapshot})"

            def repoId = isSnapshot ? 'maven-snapshots' : 'releases'
            def repoUrl = "${env.NEXUS_URL}/nexus/content/repositories/${repoId}/"

            writeFile file: 'jenkins-settings.xml', text: """
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>${repoId}</id>
      <username>${NU}</username>
      <password>${NP}</password>
    </server>
  </servers>
</settings>
""".stripIndent()

            sh """
              echo "Deploying to ${repoUrl}"
              mvn -B -s jenkins-settings.xml -DskipTests deploy \
                -DaltDeploymentRepository=${repoId}::default::${repoUrl}
            """
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        script {
          env.WAR_PATH = sh(script: 'ls -1 target/*.war | head -n1', returnStdout: true).trim()
          echo "WAR: ${env.WAR_PATH}"
        }

        withCredentials([sshUserPrivateKey(
          credentialsId: env.TOMCAT_SSH_ID,
          keyFileVariable: 'SSH_KEY',
          usernameVariable: 'SSH_USER'
        )]) {
          sh '''
            set -euo pipefail
            echo "Testing SSH to ${TOMCAT_HOST} ..."
            ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "$SSH_USER@${TOMCAT_HOST}" 'echo ok'
          '''

          sh '''
            set -euo pipefail
            echo "Copying ${WAR_PATH} to server..."
            scp -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "${WAR_PATH}" "$SSH_USER@${TOMCAT_HOST}:/tmp/app.war"
          '''

          sh """
            set -euo pipefail
            ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "$SSH_USER@${TOMCAT_HOST}" "
              set -e
              sudo install -d -m 0755 ${TOMCAT_WEBAPPS} || true

              if systemctl list-unit-files | grep -q '^${TOMCAT_SERVICE}\\.service'; then
                sudo systemctl stop ${TOMCAT_SERVICE} || true
              else
                if [ -x /opt/tomcat/bin/shutdown.sh ]; then /opt/tomcat/bin/shutdown.sh || true; fi
              fi

              sudo rm -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war
              sudo rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME}
              sudo cp /tmp/app.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war

              if id tomcat >/dev/null 2>&1; then
                sudo chown -R tomcat:tomcat ${TOMCAT_WEBAPPS}
              fi

              if systemctl list-unit-files | grep -q '^${TOMCAT_SERVICE}\\.service'; then
                sudo systemctl start ${TOMCAT_SERVICE}
              else
                if [ -x /opt/tomcat/bin/startup.sh ]; then /opt/tomcat/bin/startup.sh; fi
              fi

              sleep 3
              (command -v ss && ss -ltnp | grep :8080) || (command -v netstat && netstat -tulpn | grep :8080) || true
              ls -la ${TOMCAT_WEBAPPS} || true
            "
          """
        }
      }
    }
  }

  post {
    success { echo '✅ Build, scan, publish, and deploy completed.' }
    failure { echo '❌ Pipeline failed. Check logs for the failing stage.' }
    always  {
      archiveArtifacts artifacts: '**/target/*.war, **/target/site/**', fingerprint: true, allowEmptyArchive: true
      sh 'rm -f jenkins-settings.xml || true'
    }
  }
}
