pipeline {
  agent any
  tools {
      maven 'maven-3.9'
  }
  parameters {
    string(name: 'ANSIBLE_SERVER', defaultValue: '139.59.150.13')
  }
  stages {
      stage("copy files to ansible server") {
          steps {
              script {
                echo "copying all neccessary files to ansible control node"
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-server-key', keyFileVariable: 'PK')]) {
                  sh 'scp -o StrictHostKeyChecking=no -i $PK ansible/* root@$ANSIBLE_SERVER:/root'

                  withCredentials([sshUserPrivateKey(credentialsId: 'ec2-server-key', keyFileVariable: 'keyFile')]) {
                    sh 'scp -o StrictHostKeyChecking=no -i $PK $keyFile root@$ANSIBLE_SERVER:/root/ssh-key.pem'
                  }
                }
              }
          }
      }
      stage("execute ansible playbook") {
        steps {
          script {
            echo "calling ansible playbook to configure ec2 instances"

            def remote = [:]
            remote.name = "ansible-server"
            remote.host = "$ANSIBLE_SERVER"
            remote.allowAnyHosts = true
            withCredentials([sshUserPrivateKey(credentialsId: 'ansible-server-key', keyFileVariable: 'keyFile', usernameVariable: 'user')]) {
              remote.user = user
              remote.identityFile = keyFile
              sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"
            }
          }
        }
      }
  }
} 
