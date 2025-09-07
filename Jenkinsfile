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
    NEXUS_CRED_ID    = 'nexus-cred'   // Jenkins Username/Password credential

    // Tomcat target
    TOMCAT_HOST      = '3.210.219.27'
    TOMCAT_SSH_ID    = 'tomcat-ssh'   // Jenkins sshUserPrivateKey credential
    TOMCAT_WEBAPPS   = '/opt/tomcat/webapps'   // change to /var/lib/tomcat/webapps if using OS package
    TOMCAT_SERVICE   = 'tomcat'                // change if your unit is tomcat9/custom
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

            def repoId  = isSnapshot ? 'maven-snapshots' : 'releases'   // must match Nexus 2 repo IDs
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

        withCredentials([sshUserPrivateKey(credentialsId: env.TOMCAT_SSH_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          // Reachability
          sh '''
            set -euo pipefail
            echo "Testing SSH to ${TOMCAT_HOST} ..."
            ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "$SSH_USER@${TOMCAT_HOST}" 'echo ok'
          '''

          // Copy artifact
          sh '''
            set -euo pipefail
            echo "Copying ${WAR_PATH} to server..."
            scp -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "${WAR_PATH}" "$SSH_USER@${TOMCAT_HOST}:/tmp/app.war"
          '''

          // Remote deploy with service/tarball fallback
          sh """
            set -euo pipefail
            ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "$SSH_USER@${TOMCAT_HOST}" "
              set -e
              sudo install -d -m 0755 ${TOMCAT_WEBAPPS} || true

              # Stop Tomcat (systemd if present, else shutdown.sh)
              if systemctl list-unit-files | grep -q '^${TOMCAT_SERVICE}\\.service'; then
                sudo systemctl stop ${TOMCAT_SERVICE} || true
              else
                if [ -x /opt/tomcat/bin/shutdown.sh ]; then /opt/tomcat/bin/shutdown.sh || true; fi
              fi

              sudo rm -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war
              sudo rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME}
              sudo cp /tmp/app.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war

              # Best-effort ownership if 'tomcat' user exists
              if id tomcat >/dev/null 2>&1; then
                sudo chown -R tomcat:tomcat ${TOMCAT_WEBAPPS}
              fi

              # Start Tomcat (systemd if present, else startup.sh)
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

    stage('Remote Diagnostics (pre-check)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: env.TOMCAT_SSH_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh """
            set -euo pipefail
            echo "Listing remote webapps and checking 8080:"
            ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "$SSH_USER@${TOMCAT_HOST}" '
              ls -la ${TOMCAT_WEBAPPS} || true
              (command -v ss && ss -ltnp | grep :8080) || (command -v netstat && netstat -tulpn | grep :8080) || true
            '
          """
        }
      }
    }

    stage('Post-Deploy Check') {
      when { expression { return env.HEALTH_URL?.trim() } }
      steps {
        sh """
          echo "Waiting for app to come up: ${HEALTH_URL}"
          deadline=\\$((SECONDS+180))
          status=000
          while [ \\$SECONDS -lt \\$deadline ]; do
            status=\\$(curl -sS -o /dev/null -w "%{http_code}" "${HEALTH_URL}" || echo 000)
            echo "HTTP status: \\$status"
            case "\\$status" in
              2??|3??) echo "App is up"; exit 0 ;;
            esac
            sleep 5
          done
          echo "Health check failed after 180s (last HTTP: \\$status)"
          exit 1
        """
      }
      post {
        failure {
          withCredentials([sshUserPrivateKey(credentialsId: env.TOMCAT_SSH_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
            sh """
              set +e
              echo "=== REMOTE DIAGNOSTICS (on failure) ==="
              ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$SSH_KEY" "$SSH_USER@${TOMCAT_HOST}" '
                echo "--- systemctl status (if present) ---"
                (systemctl status ${TOMCAT_SERVICE} --no-pager 2>/dev/null | tail -n 80) || true
                echo "--- open ports ---"
                (command -v ss && ss -ltnp | grep :8080) || (command -v netstat && netstat -tulpn | grep :8080) || true
                echo "--- webapps listing ---"
                ls -la ${TOMCAT_WEBAPPS} || true
                echo "--- last catalina.out ---"
                tail -n 200 /opt/tomcat/logs/catalina.out 2>/dev/null || true
              '
            """
          }
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
