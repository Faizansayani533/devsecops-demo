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

    stage('Checkout Source') {
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

    // ---------------------------
    // DOWNLOAD ODC DATABASE (ONCE PER POD)
    // ---------------------------
stage('Prepare Dependency-Check DB') {
  steps {
    container('dependency-check') {
      withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
        sh '''
          echo "📥 Preparing Dependency-Check DB..."

          if [ ! -d "/odc-data/nvdcve" ]; then
            echo "First time DB download..."
            /usr/share/dependency-check/bin/dependency-check.sh \
              --updateonly \
              --data /odc-data \
              --nvdApiKey $NVD_API_KEY
          else
            echo "Using existing offline DB"
          fi
        '''
      }
    }
  }
}

    // ---------------------------
    // OWASP SCAN (OFFLINE MODE)
    // ---------------------------
stage('OWASP Dependency Check') {
  steps {
    container('dependency-check') {
      sh '''
        echo "🔍 OWASP Dependency Check (OFFLINE DB MODE)"

        rm -rf dc-report
        mkdir dc-report

        /usr/share/dependency-check/bin/dependency-check.sh \
          --project "devsecops-demo" \
          --scan target \
          --scan pom.xml \
          --format HTML \
          --out dc-report \
          --disableAssembly \
          --data /odc-data \
          --noupdate \
          --failOnCVSS 9
      '''
    }
  }
}

    // ---------------------------
    // BUILD & PUSH IMAGE
    // ---------------------------
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

    // ---------------------------
    // TRIVY IMAGE SCAN
    // ---------------------------
    stage('Trivy Image Scan (CRITICAL Gate)') {
      steps {
        container('trivy') {
          sh '''
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

    // ---------------------------
    // UPDATE GITOPS REPO
    // ---------------------------
    stage('Update GitOps Repo') {
      steps {
        withCredentials([string(credentialsId: 'gitops-token', variable: 'GIT_TOKEN')]) {
          sh '''
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
      echo "✅ CI PASSED — Argo CD will deploy automatically"
    }

    failure {
      echo "❌ PIPELINE FAILED — Security gate or quality gate blocked"
    }
  }
}
