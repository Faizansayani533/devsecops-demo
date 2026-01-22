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
        checkout scm
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

    stage('Install kubectl') {
      steps {
        sh '''
          if ! command -v kubectl >/dev/null 2>&1; then
            echo "Installing kubectl..."
            curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
            chmod +x kubectl
            mkdir -p $HOME/bin
            mv kubectl $HOME/bin/
            export PATH=$HOME/bin:$PATH
          fi

          kubectl version --client
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          echo "Updating deployment..."
          kubectl set image deployment/devsecops-demo \
            devsecops-demo=$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG \
            -n default

          echo "Waiting for rollout..."
          kubectl rollout status deployment/devsecops-demo -n default --timeout=120s
        '''
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
