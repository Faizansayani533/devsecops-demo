pipeline {
  agent { label 'kaniko' }

  environment {
    AWS_REGION   = "eu-north-1"
    ECR_REGISTRY = "079662785620.dkr.ecr.eu-north-1.amazonaws.com"
    IMAGE_NAME   = "devsecops-app"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    SONARQUBE    = "sonarqube"
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/Faizansayani533/devsecops-demo.git'
      }
    }

    stage('Maven Build & Test') {
      steps {
        container('maven') {
          sh 'mvn clean verify'
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        container('maven') {
          withSonarQubeEnv("${SONARQUBE}") {
            sh '''
              mvn sonar:sonar \
              -Dsonar.projectKey=devsecops-demo \
              -Dsonar.projectName=devsecops-demo
            '''
          }
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

    stage('Kaniko Build & Push (ECR)') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context . \
              --dockerfile Dockerfile \
              --destination $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG \
              --destination $ECR_REGISTRY/$IMAGE_NAME:latest
          '''
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        container('kubectl') {
          sh '''
		set -x
        	which kubectl
        	kubectl version --client
        	kubectl get nodes
        	kubectl get deployment -A
        	kubectl set image deployment/devsecops-demo devsecops-demo=$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG
        	kubectl rollout status deployment/devsecops-demo --timeout=120s
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ PIPELINE SUCCESSFUL"
    }
    failure {
      echo "❌ PIPELINE FAILED"
    }
  }
}
