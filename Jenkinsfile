pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Deploy') {
      steps {
        // run safe deploy script via sudo (defined on the server)
        sh 'sudo /usr/local/bin/deploy_site.sh'
      }
    }
  }
  post {
    success { echo "Deployed!" }
    failure { echo "Deploy failed." }
  }
}
