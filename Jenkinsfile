pipeline {
  agent any

  environment {
    EC2_USER = "ec2-user"         // change to ubuntu if using Ubuntu
    EC2_IP   = "PUBLIC_IPV4"      // or set as Jenkins parameter/env var
    SSH_CRED_ID = "ec2-ssh"       // Jenkins credential ID (SSH Username with private key)
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Deploy to EC2') {
      steps {
        // Bind the SSH private key to a temporary file path (SSH_KEY)
        withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CRED_ID, keyFileVariable: 'SSH_KEY')]) {
          // Use shell to scp files up, then run remote commands using ssh
          sh """
            echo "Using SSH key file: \$SSH_KEY"
            # copy repository contents to a staging folder on remote
            scp -i \$SSH_KEY -o StrictHostKeyChecking=no -r * ${env.EC2_USER}@${env.EC2_IP}:~/site_tmp/
            
            # Run remote commands (install webserver if needed, move files to webroot)
            ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${env.EC2_USER}@${env.EC2_IP} <<'SSH_EOF'
              set -e
              if [ -f /etc/os-release ]; then . /etc/os-release; fi
              if command -v nginx >/dev/null 2>&1; then WEBSERVER=nginx; \
              elif command -v apache2 >/dev/null 2>&1 || command -v httpd >/dev/null 2>&1; then WEBSERVER=apache; \
              else
                if [ "\$ID" = "ubuntu" ]; then sudo apt-get update -y; sudo apt-get install -y nginx; WEBSERVER=nginx; sudo systemctl enable --now nginx; \
                else sudo yum install -y nginx || sudo amazon-linux-extras install -y nginx1 || true; sudo systemctl enable --now nginx; WEBSERVER=nginx; fi
              fi

              if [ "\$WEBSERVER" = "nginx" ]; then REMOTE_ROOT=/usr/share/nginx/html; else REMOTE_ROOT=/var/www/html; fi
              sudo mkdir -p /var/backups/jenkins-deploy || true
              sudo cp -r \$REMOTE_ROOT /var/backups/jenkins-deploy/site-\$(date +%Y%m%d-%H%M%S) 2>/dev/null || true
              sudo rm -rf \$REMOTE_ROOT/* || true
              sudo cp -r ~/site_tmp/* \$REMOTE_ROOT/ || true
              sudo chown -R root:root \$REMOTE_ROOT
              sudo chmod -R 755 \$REMOTE_ROOT
              hostname > /tmp/jenkins-hostname.txt
              sudo mv /tmp/jenkins-hostname.txt \$REMOTE_ROOT/__hostname__
              if [ "\$WEBSERVER" = "nginx" ]; then sudo systemctl restart nginx || true; else sudo systemctl restart apache2 || true; fi
              rm -rf ~/site_tmp || true
              echo "DEPLOY_DONE"
SSH_EOF
          """
        } // withCredentials
      }
    }

    stage('Report') {
      steps {
        echo "Your site should be available at: http://${EC2_IP}/"
        sh "echo 'HTTP status from ${EC2_IP}:' ; curl -I --max-time 10 http://${EC2_IP} || true"
      }
    }
  }

  post {
    failure { echo "Deployment failed — check console output." }
  }
}
