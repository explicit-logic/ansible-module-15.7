# Module 15 - Configuration Management with Ansible

This repository contains a demo project created as part of my **DevOps studies** in the [TechWorld with Nana – DevOps Bootcamp](https://www.techworld-with-nana.com/devops-bootcamp).

**Demo Project:** Ansible Integration in Jenkins

**Technologies used:** Ansible, Jenkins, DigitalOcean, AWS, Boto3, Docker, Java, Maven, Linux, Git

**Project Description:**

- Create and configure a dedicated server for Jenkins
- Create and configure a dedicated server for Ansible Control Node
- Write Ansible Playbook, which configures 2 EC2 Instances
- Add ssh key file credentials in Jenkins for Ansible Control Node server and Ansible Managed Node servers
- Configure Jenkins to execute the Ansible Playbook on remote Ansible Control Node server as part of the CI/CD pipeline
- So the Jenkinsfile configuration will do the following:
  - a.Connect to the remote Ansible Control Node server
  - b.Copy Ansible playbook and configuration files to the remote Ansible Control Node server
  - c.Copy the ssh keys for the Ansible Managed Node servers to the Ansible Control Node server
  - d.Install Ansible, Python3 and Boto3 on the Ansible Control Node server
  - e.With everything installed and copied to the remote Ansible Control Node server, execute the playbook remotely on that Control Node that will configure the 2 EC2 Managed Nodes

---

Overview
![](./images/overview.png)


### Create and configure a dedicated server for Jenkins

Install Jenkins on DigitalOcean
https://github.com/explicit-logic/jenkins-module-8.1

https://github.com/explicit-logic/jenkins-module-8.2


### Create and configure a dedicated server for Ansible Control Node

- Create a droplet on DigitalOcean

CPU Option: Regular
vCPU: 2
RAM: 2 GB
Disk: 60 GB

![](./images/ansible-droplet.png)

Name it `ansible-server`

Connect to a droplet and install ansible

```sh
ssh root@<PUBLIC-IP>
apt update
apt install ansible-core
```

Install required modules: `boto3`, `botocore`

```sh
apt install python3-boto3
```

Configre aws credentials

```sh
mkdir .aws
vim credentials
```

Copy and paste default credentials from your machine:
```sh
cat ~/.aws/credentials
```

```conf
[default]
aws_access_key_id =
aws_secret_access_key =
```

Exit server

### Create 2 EC2 Instances to be managed by Ansible

- Launch 2 amazon EC2 instances

Create a new key pair: `ansible-jenkins`

![](./images/ec2-instances.png)

### Execute Ansible Playbook from Jenkins Pipeline to configure 2 EC2 instances

**JenkinsFile: Copy files from Jenkins to Ansible Server**

- Add global credentials at Jenkins

Kind: `SSH Username with private key`

ID: `ansible-server-key`

Username: `root`

Make a private key that was used to create `ansible-server` compatible with Jenkins

```sh
ssh-keygen -p -f ~/.ssh/id_rsa -m pem -P "" -N ""
```

See the converted key:
```sh
cat ~/.ssh/id_rsa
```

The key value must have `RSA` in the first line:
```
-----BEGIN RSA PRIVATE KEY-----
```

Copy the whole value to jenkins

- Add global credentials to Jenkins from EC2 instance

Kind: `SSH Username with private key`

ID: `ec2-server-key`

Username: `ec2-user`

Copy the whole value to jenkins from downloaded EC2 key
```sh
cat ~/Downloads/ansible-jenkins.pem
```

![](./images/ec2-server-key.png)

Add a new stage to Jenkinsfile, set ANSIBLE_SERVER param which is public ip of the ansible server
```groovy
      stage("copy files to ansible server") {
          steps {
              script {
                echo "copying all neccessary files to ansible control node"
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-server-key', keyFileVariable: 'PK')]) {
                  sh 'scp -o StrictHostKeyChecking=no -i $PK ansible/* root@$ANSIBLE_SERVER:/root'

                  withCredentials([sshUserPrivateKey(credentialsId: 'ec2-server-key', keyFileVariable: 'keyFile')]) {
                    sh "scp -o StrictHostKeyChecking=no -i $PK $keyFile root@$ANSIBLE_SERVER:/root/ssh-key.pem"
                  }
                }
              }
          }
      }
```

- Create a new Pipeline at Jenkins

Name: `ansible-pipeline`

Type: `Pipeline`

![](./images/ansible-pipeline.png)

Set up

- Pipeline script from SCM

Repository URL: https://github.com/explicit-logic/ansible-module-15.7

Credentials: `github` (username and password)

Branch Specifier: `main`

![](./images/pipeline-scm.png)

Run the pipeline, press `Build with Parameters`

![](./images/pipeline-copy-stage.png)

Connect to the ansible server and check if the files transfered

```sh
ssh root@<ansible-server-ip>
ls
```

![](./images/ansible-server-transfered-configs.png)

### Jenkinsfile: Execute Ansible Playbook from Jenkins

Install Jenkins plugin: `SSH Pipeline Steps`

Add the following stage to Jenkinsfile:

```groovy
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
              sshCommand remote: remote, command: "ls -l"
            }
          }
        }
      }
```

Execute pipeline again

![](./images/exec-on-ansible-server.png)

Finally, replace `ls` with playbook run command:

```groovy
sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"
```

![](./images/exec-playbook-from-jenkins.png)

### Add optional step of preparing ansible server

Add to Jenkinsfile
```groovy
sshScript remote: remote, script: "prepare-ansible-server.sh"
```

Execute the pipeline

![](./images/demo.gif)

Check docker on the EC2 instance:

```sh
mv ~/Downloads/ansible-jenkins.pem ~/.ssh
chmod 400 ~/.ssh/ansible-jenkins.pem
ssh -i ~/.ssh/ansible-jenkins.pem ec2-user@<ec2-ip-address>
docker version
docker compose version
```

![](./images/check-ec2-docker.png)

### Clean up
