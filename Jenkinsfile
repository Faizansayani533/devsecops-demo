pipeline {
  agent any

  tools {
    maven 'Maven'
  }

  environment {
    SONAR_SCANNER_HOME = tool 'sonar-scanner'
  }

  stages {

    stage('Checkout') {
      steps {
        git 'https://github.com/Faizansayani533/devsecops-demo.git'
      }
    }

    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh """
          $SONAR_SCANNER_HOME/bin/sonar-scanner \
          -Dsonar.projectKey=devsecops-demo \
          -Dsonar.sources=. \
          -Dsonar.java.binaries=target
          """
        }
      }
    }
  }
}
