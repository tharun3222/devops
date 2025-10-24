pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Deploy') {
  steps {
    echo "Deploying from workspace: ${env.WORKSPACE}"
    sh "sudo /usr/local/bin/deploy_site.sh \"${env.WORKSPACE}\""
  }
}
  post {
    success { echo "Deployed!" }
    failure { echo "Deploy failed." }
  }
}
