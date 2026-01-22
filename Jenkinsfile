post {
  always {

    publishHTML([
      allowMissing: false,
      alwaysLinkToLastBuild: true,
      keepAll: true,
      reportDir: 'dc-report',
      reportFiles: 'dependency-check-report.html',
      reportName: 'OWASP Dependency Check Report'
    ])

    publishHTML([
      allowMissing: false,
      alwaysLinkToLastBuild: true,
      keepAll: true,
      reportDir: '.',
      reportFiles: 'trivy-report.html',
      reportName: 'Trivy Image Scan Report'
    ])
  }

  success {
    echo "✅ PIPELINE SUCCESSFUL — No CRITICAL vulnerabilities"
  }

  failure {
    echo "❌ PIPELINE FAILED — CRITICAL vulnerabilities found"
  }
}
