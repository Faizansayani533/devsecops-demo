pipeline {
  agent {
    kubernetes {
      label 'kaniko'
      defaultContainer 'jnlp'
    }
  }

  tools {
    maven 'Maven'
  }

  environment {
    AWS_REGION = "eu-north-1"
    ECR_REPO   = "079662785620.dkr.ecr.eu-north-1.amazonaws.com/devsecops-app"
    IMAGE_TAG  = "${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Unit Test') {
      steps {
        sh 'mvn clean package'
      }
    }

    stage('SonarQube Analysis (SAST)') {
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

    stage('Quality Gate') {
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('OWASP Dependency Check (SCA)') {
      steps {
        dependencyCheck additionalArguments: '--scan .', odcInstallation: 'Default'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }

    stage('Kaniko Build & Push (ECR)') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context `pwd` \
              --dockerfile Dockerfile \
              --destination=$ECR_REPO:$IMAGE_TAG \
              --destination=$ECR_REPO:latest \
              --skip-tls-verify \
              --cache=true
          '''
        }
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh '''
          trivy image --exit-code 1 --severity CRITICAL,HIGH $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          sed -i "s|IMAGE_PLACEHOLDER|$ECR_REPO:$IMAGE_TAG|g" k8s/petclinic.yml
          kubectl apply -f k8s/
        '''
      }
    }
  }

  post {
    success {
      echo "✅ DevSecOps Pipeline Completed Successfully"
    }
    failure {
      echo "❌ Pipeline Failed – Check Jenkins Logs"
    }
  }
}
