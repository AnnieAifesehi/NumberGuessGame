pipeline {
  agent any
  tools { jdk 'Java-17'; maven 'Maven' }
  options { timestamps(); disableConcurrentBuilds() }

  environment {
    REPO_URL         = 'https://github.com/AnnieAifesehi/NumberGuessGame.git'
    REPO_BRANCH      = 'main'

    // --- SonarQube ---
    SONARQUBE_SERVER = 'SonarQube'   // Jenkins global server name (Manage Jenkins ➜ System)

    // --- Nexus (server id + repo path) ---
    NEXUS_URL        = 'http://54.157.3.135:8081'
    NEXUS_REPO_ID    = 'nexus-releases'       // <server id> used in settings.xml
    NEXUS_REPO_PATH  = 'maven-releases'       // hosted repo name in Nexus
    NEXUS_CRED_ID    = 'nexus-cred'           // Jenkins Username/Password creds for Nexus

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
      steps {
        git branch: env.REPO_BRANCH, url: env.REPO_URL
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests clean package'
      }
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
        def version   = sh(script: "mvn -q -DforceStdout help:evaluate -Dexpression=project.version", returnStdout: true).trim()
        def isSnapshot = version.endsWith('-SNAPSHOT')

        // Nexus 2: repo *IDs* and Nexus 2 URL pattern
        def repoId   = isSnapshot ? 'maven-snapshots' : 'releases'     // <--- match Nexus 2 repo IDs
        def repoPath = repoId                                          // path segment equals ID in Nexus 2
        def url      = "${env.NEXUS_URL}/nexus/content/repositories/${repoPath}/"

        // temp settings.xml with credentials for the chosen server <id>
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
          mvn -B -s jenkins-settings.xml -DskipTests deploy \
            -DaltDeploymentRepository=${repoId}::default::${url}
        """
      }
    }
  }
}

stage('Deploy to Tomcat') {
      steps {
        script {
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
    success { echo '✅ Build, scan, publish, and deploy completed.' }
    failure { echo '❌ Pipeline failed. Check logs for the failing stage.' }
    always  {
      archiveArtifacts artifacts: '**/target/*.war, **/target/site/**', fingerprint: true, allowEmptyArchive: true
      // Clean up the temporary settings file if present
      sh 'rm -f jenkins-settings.xml || true'
    }
  }
}
