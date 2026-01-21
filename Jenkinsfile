pipeline {
  agent any

  tools {
    maven 'Maven'
  }

  stages {

    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh '''
          mvn sonar:sonar \
          -Dsonar.projectKey=devsecops-demo \
          -Dsonar.projectName=devsecops-demo \
          -Dsonar.scm.disabled=true
          '''
        }
      }
    }

  }
}
