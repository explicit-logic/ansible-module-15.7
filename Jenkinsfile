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
  }
} 
