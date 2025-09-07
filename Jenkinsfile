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

    stage('Quality Gate') {
      steps {
        script {
          timeout(time: 10, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') { error "Quality Gate failed: ${qg.status}" }
          }
        }
      }
    }

    stage('Publish to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.NEXUS_CRED_ID, usernameVariable: 'NU', passwordVariable: 'NP')]) {
          script {
            // 1) Read project version
            def version = sh(script: "mvn -q -DforceStdout help:evaluate -Dexpression=project.version", returnStdout: true).trim()
            def isSnapshot = version.endsWith('-SNAPSHOT')
            echo "Project version: ${version} (snapshot=${isSnapshot})"

            // 2) Nexus 2 repo IDs and URLs
            def repoId = isSnapshot ? 'maven-snapshots' : 'releases' // MUST match Nexus 2 repo IDs
            def repoUrl = "${env.NEXUS_URL}/nexus/content/repositories/${repoId}/"

            // 3) Write a minimal settings.xml with matching <server id>
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

            // 4) Deploy using Nexus 2 URL pattern
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
      sh 'rm -f jenkins-settings.xml || true'
    }
  }
}
