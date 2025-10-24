pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Deploy') {
      steps {
        echo "Deploying from workspace: ${env.WORKSPACE}"
        // call the server deploy script and pass the workspace path (handles spaces)
        sh "sudo /usr/local/bin/deploy_site.sh \"${env.WORKSPACE}\""
      }
    }
  } // end stages

  post {
    success {
      echo "Deployment finished: SUCCESS"
    }
    failure {
      echo "Deployment finished: FAILURE"
    }
  }
} // end pipeline
