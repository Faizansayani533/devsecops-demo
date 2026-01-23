pipeline {
  agent { label 'kaniko' }

  environment {
    AWS_REGION   = "eu-north-1"
    ECR_REGISTRY = "079662785620.dkr.ecr.eu-north-1.amazonaws.com"
    IMAGE_NAME   = "devsecops-app"
    IMAGE_TAG    = "${BUILD_NUMBER}"
    SONARQUBE    = "sonarqube"
    GITOPS_REPO  = "https://github.com/Faizansayani533/devsecops-gitops.git"
  }

  stages {

    // =========================
    // CHECKOUT SOURCE
    // =========================
    stage('Checkout Source Code') {
      steps {
        checkout scm
      }
    }

    // =========================
    // BUILD & TEST
    // =========================
    stage('Maven Build & Test') {
      steps {
        container('maven') {
          sh 'mvn clean verify'
        }
      }
    }

    // =========================
    // SONARQUBE (SAST)
    // =========================
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

    // =========================
    // OWASP DEPENDENCY CHECK
    // =========================
    stage('OWASP Dependency Check') {
      steps {
        container('dependency-check') {
          sh '''
            echo "🔍 OWASP Dependency Check (OFFLINE DB MODE)"

            mkdir -p dc-report

            /usr/share/dependency-check/bin/dependency-check.sh \
              --project "devsecops-demo" \
              --scan target \
              --scan pom.xml \
              --format HTML \
              --out dc-report \
              --disableAssembly \
              --data odc-data \
              --failOnCVSS 9
          '''
        }
      }
    }

    // =========================
    // BUILD & PUSH IMAGE
    // =========================
    stage('Build & Push Image (Kaniko)') {
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

    // =========================
    // TRIVY IMAGE SCAN
    // =========================
    stage('Trivy Image Scan (CRITICAL Gate)') {
      steps {
        container('trivy') {
          sh '''
            echo "🔍 Trivy CRITICAL image scan..."

            trivy image \
              --scanners vuln \
              --severity CRITICAL \
              --exit-code 1 \
              --no-progress \
              $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG
          '''
        }
      }
    }

    // =========================
    // UPDATE GITOPS REPO
    // =========================
    stage('Update GitOps Repo') {
      steps {
        withCredentials([string(credentialsId: 'gitops-token', variable: 'GIT_TOKEN')]) {
          sh '''
            echo "🚀 Updating GitOps repo..."

            rm -rf gitops
            git clone https://$GIT_TOKEN@github.com/Faizansayani533/devsecops-gitops.git gitops

            cd gitops/petclinic

            sed -i "s|image: .*|image: $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG|g" deployment.yaml

            git config user.email "jenkins@devsecops.com"
            git config user.name "jenkins"

            git add deployment.yaml
            git commit -m "Update image to $IMAGE_TAG"
            git push origin main
          '''
        }
      }
    }
  }

  // =========================
  // REPORT PUBLISH
  // =========================
  post {
    always {
      publishHTML([
        allowMissing: true,
        alwaysLinkToLastBuild: true,
        keepAll: true,
        reportDir: 'dc-report',
        reportFiles: 'dependency-check-report.html',
        reportName: 'OWASP Dependency Check Report'
      ])
    }

    success {
      echo "✅ CI PASSED — ArgoCD will auto-deploy"
    }

    failure {
      echo "❌ PIPELINE FAILED — Security or quality gate blocked"
    }
  }
}
