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
      	export SONAR_SCANNER_OPTS="-Xmx1024m"

      	$SONAR_SCANNER_HOME/bin/sonar-scanner \
      	-Dsonar.projectKey=devsecops-demo \
      	-Dsonar.projectName=devsecops-demo \
      	-Dsonar.sources=src \
      	-Dsonar.java.binaries=target \
      	-Dsonar.scm.disabled=true \
      	-Dsonar.javascript.enabled=false \
      	-Dsonar.iac.enabled=false \
      	-Dsonar.docker.enabled=false
      	'''
        }
      }
    }

  }
}
