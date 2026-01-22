pipeline {
  agent { label 'kaniko' }

  environment {
    AWS_REGION   = "eu-north-1"
    ECR_REGISTRY = "079662785620.dkr.ecr.eu-north-1.amazonaws.com"
    IMAGE_NAME   = "devsecops-demo"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    SONARQUBE    = "sonarqube"
  }

  tools {
    maven "Maven3"
  }

  stages {

    stage('Checkout Code') {
      steps {
        git url: 'https://github.com/Faizansayani533/devsecops-demo.git', branch: 'main'
      }
    }

    stage('Maven Build & Test') {
      steps {
        sh "mvn clean verify"
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv("${SONARQUBE}") {
          sh """
          mvn sonar:sonar \
          -Dsonar.projectKey=devsecops-demo \
          -Dsonar.projectName=devsecops-demo \
          -Dsonar.host.url=$SONAR_HOST_URL \
          -Dsonar.login=$SONAR_AUTH_TOKEN
          """
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

    /* ========== OPTIONAL (NVD ERROR AATA HAI ISLIYE OFF) ==========
    stage('OWASP Dependency Check') {
      steps {
        dependencyCheck additionalArguments: '--scan .', odcInstallation: 'Default'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }
    ================================================================ */

    stage('Build & Push Image (Kaniko → ECR)') {
      steps {
        container('kaniko') {
          sh '''
            echo "Configuring ECR auth..."
            mkdir -p /kaniko/.docker

            cat <<EOF > /kaniko/.docker/config.json
            {
              "credHelpers": {
                "079662785620.dkr.ecr.eu-north-1.amazonaws.com": "ecr-login"
              }
            }
EOF

            echo "Building & pushing image..."

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
        sh """
        trivy image $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG || true
        """
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh """
        kubectl set image deployment/devsecops-demo devsecops-demo=$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG -n default
        kubectl rollout status deployment/devsecops-demo -n default
        """
      }
    }
  }

  post {
    success {
      echo "✅ PIPELINE SUCCESSFUL – APP DEPLOYED"
    }
    failure {
      echo "❌ PIPELINE FAILED – CHECK LOGS"
    }
  }
}
