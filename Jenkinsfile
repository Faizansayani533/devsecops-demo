pipeline {
  agent { label 'kaniko' }

  environment {
    AWS_REGION   = "eu-north-1"
    ECR_REGISTRY = "079662785620.dkr.ecr.eu-north-1.amazonaws.com"
    IMAGE_NAME   = "devsecops-demo"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    SONARQUBE    = "sonarqube"
  }

  stages {

    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    stage('Maven Build & Test') {
      steps {
        container('maven') {
          sh 'mvn -v'
          sh 'mvn clean verify'
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        container('maven') {
          withSonarQubeEnv("${SONARQUBE}") {
            sh """
              mvn sonar:sonar \
              -Dsonar.projectKey=devsecops-demo \
              -Dsonar.projectName=devsecops-demo
            """
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

	stage('KANIKO-SHELL-TEST') {
  steps {
    container('kaniko') {
      sh 'echo "I am inside kaniko container"'
      sh 'ls -l /'
      sh 'ls -l /kaniko'
      sh 'ls -l /workspace'
    }
  }
}


    stage('Build & Push Image (Kaniko → ECR)') {
      steps {
        container('kaniko') {
          sh '''
            mkdir -p /kaniko/.docker

            cat <<EOF > /kaniko/.docker/config.json
            {
              "credHelpers": {
                "079662785620.dkr.ecr.eu-north-1.amazonaws.com": "ecr-login"
              }
            }
EOF

            /kaniko/executor \
              --context $WORKSPACE \
              --dockerfile $WORKSPACE/Dockerfile \
              --destination $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG \
              --destination $ECR_REGISTRY/$IMAGE_NAME:latest \
              --skip-tls-verify
          '''
        }
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh "trivy image $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG || true"
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          kubectl set image deployment/devsecops-demo devsecops-demo=$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG
          kubectl rollout status deployment/devsecops-demo
        '''
      }
    }
  }

  post {
    success { echo "✅ PIPELINE SUCCESSFUL" }
    failure { echo "❌ PIPELINE FAILED" }
  }
}
