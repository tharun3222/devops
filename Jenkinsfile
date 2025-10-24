pipeline {
    agent any

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                # ===== Basic Setup =====
                EC2_USER=ubuntu            # change to ec2-user if Amazon Linux
                EC2_IP=YOUR_PUBLIC_IPv4    # put your EC2 public IPv4 here
                KEY_PATH=/var/lib/jenkins/your-key.pem  # path to your EC2 private key on Jenkins

                echo "Connecting to EC2: $EC2_USER@$EC2_IP"

                # ===== Copy HTML file to EC2 =====
                scp -o StrictHostKeyChecking=no -i $KEY_PATH index.html $EC2_USER@$EC2_IP:/tmp/

                # ===== Run remote commands =====
                ssh -o StrictHostKeyChecking=no -i $KEY_PATH $EC2_USER@$EC2_IP <<'EOF'
                  # install nginx if not present
                  if ! command -v nginx >/dev/null 2>&1; then
                      sudo apt update -y && sudo apt install -y nginx
                      sudo systemctl enable nginx
                      sudo systemctl start nginx
                  fi

                  # copy html file to nginx web folder
                  sudo cp /tmp/index.html /var/www/html/index.html
                  sudo systemctl restart nginx
                  echo "Deployment done!"
                EOF
                '''
            }
        }

        stage('Access Website') {
            steps {
                echo 'Open your webpage at: http://YOUR_PUBLIC_IPv4/'
            }
        }
    }
}
