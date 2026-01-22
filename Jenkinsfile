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

    // =======================
    // OWASP DEPENDENCY CHECK
    // =======================
    stage('OWASP Dependency Check') {
      steps {
        container('dependency-check') {
          withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
            sh '''
              rm -rf /tmp/dc-report
              mkdir -p /tmp/dc-report

              /usr/share/dependency-check/bin/dependency-check.sh \
                --project "devsecops-demo" \
                --scan target \
                --scan pom.xml \
                --format HTML \
                --out /tmp/dc-report \
                --disableAssembly \
                --nvdApiKey $NVD_API_KEY \
                --failOnCVSS 9
            '''
          }
        }
      }
    }

    // =======================
    // BUILD & PUSH IMAGE
    // =======================
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

    // =======================
    // TRIVY IMAGE SCAN
    // =======================
stage('Trivy Image Scan') {
  steps {
    container('trivy') {
      sh '''
        # Scan and generate JSON (fail on CRITICAL)
        trivy image \
          --severity CRITICAL \
          --exit-code 1 \
          --no-progress \
          --format json \
          --output trivy-report.json \
          $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG

        # Convert JSON to HTML report
        trivy convert \
          --format template \
          --template "@builtin/html.tpl" \
          --output trivy-report.html \
          trivy-report.json
      '''
        }
      }
    }

    // =======================
    // INSTALL KUBECTL
    // =======================
    stage('Install kubectl') {
      steps {
        sh '''
          if [ ! -f /home/jenkins/bin/kubectl ]; then
            curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
            chmod +x kubectl
            mkdir -p /home/jenkins/bin
            mv kubectl /home/jenkins/bin/
          fi

          /home/jenkins/bin/kubectl version --client
        '''
      }
    }

    // =======================
    // DEPLOY TO EKS
    // =======================
    stage('Deploy to EKS') {
      steps {
        sh '''
          echo "Updating deployment..."

          /home/jenkins/bin/kubectl set image deployment/devsecops-demo \
            devsecops-demo=$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG \
            -n default

          /home/jenkins/bin/kubectl rollout status deployment/devsecops-demo \
            -n default --timeout=180s
        '''
      }
    }

    // =======================
    // PUBLISH REPORTS
    // =======================
    stage('Publish Security Reports') {
      steps {

        publishHTML([
          allowMissing: true,
          alwaysLinkToLastBuild: true,
          keepAll: true,
          reportDir: '/tmp/dc-report',
          reportFiles: 'dependency-check-report.html',
          reportName: 'OWASP Dependency Check Report'
        ])

        publishHTML([
          allowMissing: true,
          alwaysLinkToLastBuild: true,
          keepAll: true,
          reportDir: '.',
          reportFiles: 'trivy-report.html',
          reportName: 'Trivy Image Scan Report'
        ])
      }
    }
  }
}
