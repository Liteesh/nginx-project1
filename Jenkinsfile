pipeline {
    agent any

    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'destroy'], description: 'Choose action')
    }

    environment {
        TF_DIR = 'terraform'
        ANSIBLE_DIR = 'ansible'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Liteesh/nginx-devops-project.git'
            }
        }

        stage('Terraform Init') {
            steps {
                dir("${TF_DIR}") {
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform Apply or Destroy') {
            steps {
                dir("${TF_DIR}") {
                    script {
                        if (params.ACTION == 'deploy') {
                            sh 'terraform apply -auto-approve'
                        } else {
                            sh 'terraform destroy -auto-approve'
                        }
                    }
                }
            }
        }

        stage('Get Public IP') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                dir("${TF_DIR}") {
                    script {
                        env.INSTANCE_IP = sh(script: "terraform output -raw public_ip", returnStdout: true).trim()
                        echo "Instance Public IP: ${env.INSTANCE_IP}"
                    }
                }
            }
        }

        stage('Configure Nginx with Ansible') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                dir("${ANSIBLE_DIR}") {
                    sh '''
                    echo "[nginx]" > inventory.ini
                    echo "${INSTANCE_IP} ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/id_rsa" >> inventory.ini
                    ansible-playbook -i inventory.ini playbook.yml
                    '''
                }
            }
        }

        stage('Health Check') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                sh 'curl -I http://${INSTANCE_IP}'
            }
        }
    }
}
###this is my code 
