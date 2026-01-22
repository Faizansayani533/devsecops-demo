pipeline {
  agent any

  tools {
    maven 'Maven'
  }

  environment {
    AWS_REGION = "eu-north-1"
    ECR_REPO = "079662785620.dkr.ecr.eu-north-1.amazonaws.com/devsecops-app"
    IMAGE_TAG = "latest"
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
          -Dsonar.projectName=devsecops-demo \
          -Dsonar.scm.disabled=true
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

    stage('Docker Build') {
      steps {
        sh '''
        docker build -t devsecops-app .
        docker tag devsecops-app:latest $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh '''
        trivy image --exit-code 0 --severity HIGH,CRITICAL $ECR_REPO:$IMAGE_TAG || true
        '''
      }
    }

    stage('Push to ECR') {
      steps {
        sh '''
        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin 079662785620.dkr.ecr.eu-north-1.amazonaws.com
        docker push $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
        kubectl apply -f k8s/
        '''
      }
    }
  }

  post {
    success {
      echo "✅ FULL DEVSECOPS PIPELINE SUCCESS"
    }
    failure {
      echo "❌ PIPELINE FAILED"
    }
  }
}
