# devops18

pipeline code:
===============
pipeline {
    agent any

    stages {
        stage('code') {
            steps {
                git branch: 'main', url: 'https://github.com/aravind-ku/devops18.git'
            }
        }
        stage('init') {
            steps {
                script {
                    sh 'echo -e "yes\n" | terraform init -reconfigure'
                }
            }
        }
        stage('plan') {
            steps {
                sh 'terraform plan'
            }
        }
        stage('apply') {
            steps {
                sh 'terraform apply --auto-approve'
            }
        }
        stage('Deploy') {
            steps {
                sh '''
                    export ANSIBLE_CONFIG=/etc/ansible/ansible.cfg
                    export ANSIBLE_HOST_KEY_CHECKING=False
                    ansible-playbook -i /opt/ansible/inventory/aws_ec2.yml ansible/deployment.yml
                    '''
            }
        }
    }
    post {
    always {
        script {
            def color = currentBuild.currentResult == "SUCCESS" ? "good" : "danger"

            slackSend(
                channel: "#devops",
                color: color,
                message: """
                        ==================================
                        📋 *JENKINS BUILD REPORT*
                        ==================================
                        
                        🖥 Job        : ${env.JOB_NAME}
                        🔢 Build No   : ${env.BUILD_NUMBER}
                        📌 Status     : *${currentBuild.currentResult}*
                        🌿 Branch     : ${env.BRANCH_NAME ?: 'main'}
                        
                        🔗 Build Logs
                        ${env.BUILD_URL}
                        
                        ==================================
                        """
                )
            }
        }
    }
}


cd /etc/ansible/
ll
vim ansible.cfg
----------------------------------------------
[defaults]
inventory = /opt/ansible/inventory/aws_ec2.yml
host_key_checking = False
remote_user = ec2-user
private_key_file = /etc/ansible/aws.pem

[inventory]
enable_plugins = aws_ec2

------------------------------------------------------------ 
vim aws.pem  #ADD the pem file code here in pem file
chown jenkins:jenkins /etc/ansible/aws.pem
chmod 400 /etc/ansible/aws.pem

next:
=======
cd /opt/
ll
mkdir ansible
cd ansible/
ll
mkdir inventory
cd inventory/
vim aws_ec2.yml
----------------------------
---
plugin: amazon.aws.aws_ec2

regions:
  - us-east-1

filters:
  "tag:aws:autoscaling:groupName": "web-server-asg"

compose:
  ansible_host: public_dns_name

groups:
  webservers: true

---------------------------

ansible --version
ansible all -i /opt/ansible/inventory/aws_ec2.yml -m ping

