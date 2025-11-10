pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['deploy', 'destroy'],
            description: 'Choose whether to deploy or destroy infrastructure'
        )
    }

    environment {
        TF_DIR = 'terraform'
        ANSIBLE_DIR = 'ansible'
        AWS_REGION = 'us-east-1'   // change if needed
    }

    stages {

        stage('Checkout Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/Liteesh/nginx-project1.git'
            }
        }

    stage('Terraform Init & Apply/Destroy') {
        steps {
            dir("${TF_DIR}") {
                script {
                    echo "🚀 Running Terraform using EC2 IAM Role..."
                    sh '''
                    export AWS_DEFAULT_REGION=${AWS_REGION}
                    terraform init -input=false

                    if [ "${ACTION}" = "deploy" ]; then
                        terraform apply -auto-approve -input=false
                    else
                        terraform destroy -auto-approve -input=false
                    fi
                    '''
                }
            }
        }
    }

        stage('Get EC2 Public IP') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                dir("${TF_DIR}") {
                    script {
                        env.INSTANCE_IP = sh(script: "terraform output -raw public_ip", returnStdout: true).trim()
                        echo "✅ EC2 Public IP: ${env.INSTANCE_IP}"
                        echo "🌍 Access your app: http://${env.INSTANCE_IP}"
                    }
                }
            }
        }

        stage('Configure Nginx using Ansible') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                dir("${ANSIBLE_DIR}") {
                    script {
                        echo "🧩 Creating dynamic inventory and running Ansible..."
                        sh """
                        echo "[nginx]" > inventory.ini
                        echo "${INSTANCE_IP} ansible_user=ec2-user ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/jenkins.pem" >> inventory.ini

                        echo "⏳ Waiting for instance to be reachable..."
                        sleep 30

                        # Disable host key checking for automation
                        export ANSIBLE_HOST_KEY_CHECKING=False

                        echo "🚀 Executing Ansible playbook..."
                        ansible-playbook -i inventory.ini playbook.yml
                        """
                    }
                }
            }
        }

        stage('Health Check') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                script {
                    echo "🌐 Checking application health..."
                    sh """
                    sleep 15
                    curl -I http://${INSTANCE_IP} || echo '⚠️ Health check failed, verify Nginx manually.'
                    """
                }
            }
        }
    }

    post {
        always {
            echo "✅ Pipeline completed. Action: ${params.ACTION}"
        }
    }
}
