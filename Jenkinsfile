pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:3355.v388858a_47b_33-5
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "4Gi"
        cpu: "2"
    tty: true
"""
    }
  }

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

    stage('Build & Unit Test') {
      steps {
        sh 'mvn clean package'
      }
    }

    stage('SonarQube Analysis (SAST)') {
      environment {
        MAVEN_OPTS = "-Xmx2048m"
      }
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
        timeout(time: 30, unit: 'MINUTES') {
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
        trivy image --exit-code 0 --severity CRITICAL,HIGH $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Push Image to ECR') {
      steps {
        sh '''
        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
        docker push $ECR_REPO:$IMAGE_TAG
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
