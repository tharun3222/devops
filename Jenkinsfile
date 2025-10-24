pipeline {
  agent any

  environment {
    // Replace these with your actual values in Jenkins (or leave blank and use credentials)
    EC2_USER = "ec2-user"         // ec2-user for Amazon Linux; ubuntu for Ubuntu
    EC2_IP   = "PUBLIC_IPV4"      // replace with your instance Public IPv4 or set as a Jenkins parameter
    SSH_CRED_ID = "ec2-ssh"       // Jenkins Credentials ID for SSH private key
    REMOTE_WEBROOT = "/usr/share/nginx/html" // destination; pipeline will detect and adapt if needed
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build (optional)') {
      steps {
        // add build steps if you have assets to compile, minify, etc.
        echo "No build step defined — just deploying static files."
      }
    }

    stage('Deploy to EC2') {
      steps {
        script {
          // Use sshagent plugin to access SSH key stored in Jenkins credentials
          sshagent (credentials: [env.SSH_CRED_ID]) {
            // copy files to remote home dir first (to avoid permission issues)
            sh """
              echo "---- scp files to remote:${EC2_USER}@${EC2_IP}:~/site_tmp ----"
              scp -o StrictHostKeyChecking=no -r * ${EC2_USER}@${EC2_IP}:~/site_tmp/
            """

            // remote commands: install webserver if missing, move files into webroot, create small hostname endpoint
            // This block detects OS and installs nginx or apache, then places files into webroot and sets permissions.
            sh """
              ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} << 'SSH_EOF'
                set -e
                echo "Detected remote user: \$(whoami)"
                # detect distro
                if [ -f /etc/os-release ]; then
                  . /etc/os-release
                  OS_NAME=\$(echo \$ID | tr '[:upper:]' '[:lower:]')
                else
                  OS_NAME="unknown"
                fi
                echo "OS_NAME=\$OS_NAME"

                # prefer nginx on Amazon Linux/Centos; on Ubuntu use apt->nginx
                if command -v nginx >/dev/null 2>&1; then
                  WEBSERVER="nginx"
                elif command -v apache2 >/dev/null 2>&1 || command -v httpd >/dev/null 2>&1; then
                  WEBSERVER="apache"
                else
                  # install nginx by default
                  if [ \"\$OS_NAME\" = \"ubuntu\" ]; then
                    sudo apt-get update -y
                    sudo apt-get install -y nginx
                    WEBSERVER="nginx"
                    sudo systemctl enable --now nginx
                  elif [ \"\$OS_NAME\" = \"amzn\" ] || [ \"\$OS_NAME\" = \"amazon\" ] || [ \"\$OS_NAME\" = \"centos\" ] || [ \"\$OS_NAME\" = \"rhel\" ]; then
                    if command -v yum >/dev/null 2>&1; then
                      sudo yum update -y
                      # amazon linux extras might be available; installing nginx via yum
                      sudo yum install -y nginx || sudo amazon-linux-extras install -y nginx1 || true
                      sudo systemctl enable --now nginx
                      WEBSERVER=\"nginx\"
                    fi
                  else
                    # fallback attempt for other distros
                    sudo apt-get update -y || sudo yum update -y || true
                    sudo apt-get install -y nginx || sudo yum install -y nginx || true
                    sudo systemctl enable --now nginx || true
                    WEBSERVER=\"nginx\"
                  fi
                fi

                echo "Using webserver: \$WEBSERVER"

                if [ \"\$WEBSERVER\" = \"nginx\" ]; then
                  # ensure webroot path
                  REMOTE_ROOT=/usr/share/nginx/html
                else
                  REMOTE_ROOT=/var/www/html
                fi

                echo "Preparing remote webroot: \$REMOTE_ROOT"

                # backup current site (optional)
                sudo mkdir -p /var/backups/jenkins-deploy || true
                sudo cp -r \$REMOTE_ROOT /var/backups/jenkins-deploy/site-\$(date +%Y%m%d-%H%M%S) 2>/dev/null || true

                # copy uploaded files into webroot (move from home dir)
                sudo rm -rf \$REMOTE_ROOT/* || true
                sudo cp -r ~/site_tmp/* \$REMOTE_ROOT/ || true
                # make sure index is readable
                sudo chown -R root:root \$REMOTE_ROOT
                sudo chmod -R 755 \$REMOTE_ROOT

                # create a small hostname endpoint to satisfy index.html JS
                hostname > /tmp/jenkins-hostname.txt
                sudo mv /tmp/jenkins-hostname.txt \$REMOTE_ROOT/__hostname__

                # restart webserver to pick up changes
                if [ \"\$WEBSERVER\" = \"nginx\" ]; then
                  sudo systemctl restart nginx || sudo service nginx restart || true
                else
                  sudo systemctl restart apache2 || sudo service apache2 restart || true
                fi

                # cleanup uploaded staging folder
                rm -rf ~/site_tmp || true

                echo "DEPLOY_DONE"
SSH_EOF
            """
          } // sshagent
        } // script
      } // steps
    } // stage

    stage('Report') {
      steps {
        echo "Your site should now be available at: http://${EC2_IP}/"
        // Try a simple curl to show HTTP status (this runs on the Jenkins agent)
        sh "echo 'HTTP status from ${EC2_IP}:' ; curl -I --max-time 10 http://${EC2_IP} || true"
      }
    }
  } // stages

  post {
    failure {
      echo "Deployment failed — check the console output for errors."
    }
  }
}
