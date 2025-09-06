pipeline {
  agent any
  tools { jdk 'Java-17'; maven 'Maven' }
  options { timestamps() }

  environment {
    REPO_URL        = 'https://github.com/AnnieAifesehi/NumberGuessGame.git'
    REPO_BRANCH     = 'main'
    SONARQUBE_SERVER= 'SonarQube'

    NEXUS_URL       = 'http://54.157.3.135:8081'
    NEXUS_REPO      = 'maven-releases'
    NEXUS_CRED_ID   = 'nexus-cred'

    TOMCAT_HOST     = '3.210.219.27'
    TOMCAT_USER     = 'ec2-user'
    TOMCAT_SSH_ID   = 'tomcat-ssh'
    TOMCAT_WEBAPPS  = '/opt/tomcat/webapps'
    APP_NAME        = 'NumberGuessGame'
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
          sh """
            mvn -B -DskipTests deploy \
              -DaltDeploymentRepository=${NEXUS_REPO}::default::${NEXUS_URL}/repository/${NEXUS_REPO}/ \
              -DrepositoryId=${NEXUS_REPO} \
              -Durl=${NEXUS_URL}/repository/${NEXUS_REPO}/ \
              -Dusername=$NU -Dpassword=$NP
          """
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        script {
          def war = sh(script: "ls -1 target/*.war | head -n1", returnStdout: true).trim()
          sshagent([env.TOMCAT_SSH_ID]) {
            sh """
              scp -o StrictHostKeyChecking=no "${war}" ${TOMCAT_USER}@${TOMCAT_HOST}:/tmp/app.war
              ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} '
                set -e
                sudo rm -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war
                sudo rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME}
                sudo cp /tmp/app.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war
              '
            """
          }
        }
      }
    }
  }

  post {
    success { echo 'Build, scan, publish, and deploy completed.' }
    failure { echo 'Pipeline failed. Check logs for the failing stage.' }
  }
}
