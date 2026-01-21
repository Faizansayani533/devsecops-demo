pipeline {
  agent any

  tools {
    maven 'Maven'
  }

  environment {
    SONAR_SCANNER_HOME = tool 'sonar-scanner'
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
      		export SONAR_SCANNER_OPTS="-Xmx2048m"
      		$SONAR_SCANNER_HOME/bin/sonar-scanner \
      		-Dsonar.projectKey=devsecops-demo \
      		-Dsonar.sources=. \
      		-Dsonar.java.binaries=target \
      		-Dsonar.scm.disabled=true
      		'''
        }
      }
    }

  }
}
