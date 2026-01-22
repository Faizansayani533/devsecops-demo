pipeline {
  agent { label 'kaniko' }

  tools {
    maven 'Maven'
  }

  environment {
    AWS_REGION = "eu-north-1"
    ECR_REPO = "079662785620.dkr.ecr.eu-north-1.amazonaws.com/devsecops-app"
    IMAGE_TAG = "${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
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
          sh '''
          mvn sonar:sonar \
          -Dsonar.projectKey=devsecops-demo \
          -Dsonar.projectName=devsecops-demo
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Kaniko Build & Push') {
      steps {
        container('kaniko') {
          sh """
          /kaniko/executor \
            --dockerfile=Dockerfile \
            --context=dir:///home/jenkins/agent/workspace/${JOB_NAME} \
            --destination=$ECR_REPO:$IMAGE_TAG \
            --destination=$ECR_REPO:latest
          """
        }
      }
    }

    stage('Trivy Scan (ECR Image)') {
      steps {
        sh '''
        trivy image --exit-code 0 --severity HIGH,CRITICAL $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh 'kubectl apply -f k8s/'
      }
    }
  }

  post {
    success {
      echo "✅ DevSecOps Pipeline Completed Successfully"
    }
    failure {
      echo "❌ Pipeline Failed"
    }
  }
}
