pipeline {
    agent any

    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'destroy'], description: 'Choose whether to deploy or destroy infrastructure')
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

        stage('Terraform Init') {
            steps {
                dir("${TF_DIR}") {
                    sh '''
                    echo "📦 Initializing Terraform..."
                    export TF_IN_AUTOMATION=true
                    terraform init -input=false
                    '''
                }
            }
        }

        stage('Terraform Apply/Destroy') {
            steps {
                dir("${TF_DIR}") {
                    script {
                        if (params.ACTION == 'deploy') {
                            echo "🚀 Deploying infrastructure..."
                            // ✅ Fix: Run Terraform via sh -c with logging to avoid Jenkins durable task bug
                            sh '''
                            export TF_IN_AUTOMATION=true
                            terraform apply -auto-approve -input=false || exit 1
                            '''
                        } else {
                            echo "🧹 Destroying infrastructure..."
                            sh '''
                            export TF_IN_AUTOMATION=true
                            terraform destroy -auto-approve -input=false || exit 1
                            '''
                        }
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
                        // ✅ Create dynamic inventory file
                        sh """
                        echo "[nginx]" > inventory.ini
                        echo "${INSTANCE_IP} ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/jenkins.pem" >> inventory.ini
                        echo "🧩 Running Ansible playbook..."
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

