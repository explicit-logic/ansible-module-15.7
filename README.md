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

## Overview

![](./images/overview.png)

The pipeline runs on **Jenkins**, copies the Ansible files to a remote **Ansible Control Node**, and from there runs a playbook that configures **2 EC2 Managed Nodes**. Ansible is agentless, so only the control node needs Ansible installed — the managed nodes are reached over plain SSH.

### 1. Create and configure a dedicated server for Jenkins

Install Jenkins on DigitalOcean following the earlier modules:

- https://github.com/explicit-logic/jenkins-module-8.1
- https://github.com/explicit-logic/jenkins-module-8.2

### 2. Create and configure a dedicated server for the Ansible Control Node

- Create a droplet on DigitalOcean

| Setting    | Value     |
| ---------- | --------- |
| CPU Option | Regular   |
| vCPU       | 2         |
| RAM        | 2 GB      |
| Disk       | 60 GB     |

![](./images/ansible-droplet.png)

Name it `ansible-server`.

Connect to the droplet and install Ansible. `ansible-core` ships the `ansible-playbook` binary used later in the pipeline:

```sh
ssh root@<PUBLIC-IP>
apt update
apt install ansible-core
```

> 📚 [Ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

Install the AWS SDK packages `boto3` / `botocore`. Ansible's `aws_ec2` **dynamic inventory** plugin uses them to query AWS for the managed nodes at runtime, so you never hardcode the EC2 IP addresses:

```sh
apt install python3-boto3
```

> 📚 [`amazon.aws.aws_ec2` inventory plugin](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)

Configure AWS credentials so the plugin can authenticate against your account:

```sh
mkdir .aws
vim .aws/credentials
```

Copy in your local default credentials (view them with `cat ~/.aws/credentials`):

```conf
[default]
aws_access_key_id = <YOUR_ACCESS_KEY_ID>
aws_secret_access_key = <YOUR_SECRET_ACCESS_KEY>
```

> 📚 [AWS CLI — configuration and credential file settings](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

Exit the server.

### 3. Create 2 EC2 instances to be managed by Ansible

Launch 2 Amazon Linux EC2 instances. Create a new key pair named `ansible-jenkins` and download the `.pem` file — Ansible will use it to SSH into the managed nodes as user `ec2-user`.

![](./images/ec2-instances.png)

### 4. Write the Ansible Playbook that configures the EC2 instances

Everything the control node needs lives in the [`ansible/`](./ansible) directory.

[`ansible.cfg`](./ansible/ansible.cfg) — points Ansible at the dynamic inventory and sets the SSH user and key used to reach the managed nodes:

```conf
[defaults]
host_key_checking = False
inventory = inventory_aws_ec2.yaml

enable_plugins = aws_ec2

remote_user = ec2-user
private_key_file = ~/ssh-key.pem
```

> 📚 [Ansible configuration settings](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)

[`inventory_aws_ec2.yaml`](./ansible/inventory_aws_ec2.yaml) — the dynamic inventory. Set `regions` to the region your instances run in:

```yaml
---
plugin: aws_ec2
regions:
  - eu-central-1
keyed_groups:
  - key: tags
    prefix: tag
  - key: instance_type
    prefix: instance_type
```

[`my-playbook.yaml`](./ansible/my-playbook.yaml) — installs Docker and the Docker Compose plugin on every managed node:

```yaml
---
- name: Install Docker
  hosts: all
  become: yes
  tasks:
    - name: Install Docker
      yum:
        name: docker
        update_cache: yes
        state: present
    - name: Start docker daemon
      systemd:
        name: docker
        state: started

- name: Install Docker-compose
  hosts: all
  tasks:
    - name: Create docker-compose directory
      file:
        path: ~/.docker/cli-plugins
        state: directory
    - name: Get architecture of remote machine
      shell: uname -m
      register: remote_arch
    - name: Install docker-compose
      get_url:
        url: "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-{{ remote_arch.stdout }}"
        dest: ~/.docker/cli-plugins/docker-compose
        mode: +x
```

> 📚 Modules used: [`ansible.builtin.yum`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html) · [`ansible.builtin.systemd_service`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_service_module.html) · [`ansible.builtin.file`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html) · [`ansible.builtin.get_url`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html)

### 5. Add SSH key credentials in Jenkins

The pipeline needs two SSH keys: one to reach the **control node** (DigitalOcean root key) and one to reach the **managed nodes** (the EC2 key pair).

**a. Control Node key** — add a global credential in Jenkins

- Kind: `SSH Username with private key`
- ID: `ansible-server-key`
- Username: `root`

Jenkins requires the classic **PEM** (`RSA`) key format. Convert the key you used to create `ansible-server` in place:

```sh
ssh-keygen -p -f ~/.ssh/id_rsa -m pem -P "" -N ""
```

Verify the first line now reads `-----BEGIN RSA PRIVATE KEY-----`:

```sh
cat ~/.ssh/id_rsa
```

Copy the whole value into Jenkins.

> 📚 [Jenkins — using SSH credentials](https://www.jenkins.io/doc/book/using/using-credentials/) · [SSH Credentials plugin](https://plugins.jenkins.io/ssh-credentials/)

**b. Managed Node key** — add a second global credential

- Kind: `SSH Username with private key`
- ID: `ec2-server-key`
- Username: `ec2-user`

Copy in the downloaded EC2 key:

```sh
cat ~/Downloads/ansible-jenkins.pem
```

![](./images/ec2-server-key.png)

### 6. Jenkinsfile — copy files to the Ansible Control Node

This stage uses an `ANSIBLE_SERVER` build parameter (the public IP of the control node) and the two credentials above. It `scp`s the contents of `ansible/` to the control node and drops the EC2 key there as `ssh-key.pem` (the path `ansible.cfg` expects).

```groovy
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
```

> ⚠️ Note the **single quotes** around the `sh` steps. They let the shell — not Groovy — expand `$PK` / `$keyFile`, which keeps the secret out of the interpolated Groovy string (Jenkins warns about this otherwise).
>
> 📚 [`withCredentials` / `sshUserPrivateKey`](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/) · [Handling credentials in a Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials)

### 7. Create the Jenkins pipeline

- Create a new item in Jenkins
- Name: `ansible-pipeline`
- Type: `Pipeline`

![](./images/ansible-pipeline.png)

Configure **Pipeline script from SCM**:

- Repository URL: https://github.com/explicit-logic/ansible-module-15.7
- Credentials: `github` (username and password)
- Branch Specifier: `main`

![](./images/pipeline-scm.png)

Run it with **Build with Parameters**, setting `ANSIBLE_SERVER` to the control node's public IP.

![](./images/pipeline-copy-stage.png)

Connect to the Ansible server and confirm the files transferred:

```sh
ssh root@<ansible-server-ip>
ls
```

![](./images/ansible-server-transfered-configs.png)

### 8. Jenkinsfile — execute the Ansible Playbook on the Control Node

Install the Jenkins plugin **`SSH Pipeline Steps`** — it provides the `sshCommand` and `sshScript` steps.

> 📚 [SSH Pipeline Steps plugin](https://plugins.jenkins.io/ssh-steps/) · [step reference](https://www.jenkins.io/doc/pipeline/steps/ssh-steps/)

Add the stage below. Start with `ls -l` to confirm the SSH connection to the control node works:

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

![](./images/exec-on-ansible-server.png)

Once the connection works, replace `ls -l` with the playbook run:

```groovy
sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"
```

![](./images/exec-playbook-from-jenkins.png)

### 9. (Optional) Prepare the Control Node automatically

Instead of installing Ansible/Boto3 on the control node by hand (step 2), let the pipeline do it. [`prepare-ansible-server.sh`](./prepare-ansible-server.sh) installs the prerequisites:

```sh
#!/usr/bin/env bash

apt update
apt install ansible -y
apt install python3-boto3
```

Run it from the pipeline with `sshScript`, just before the playbook command:

```groovy
sshScript remote: remote, script: "prepare-ansible-server.sh"
sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"
```

Execute the pipeline.

![](./images/demo.gif)

Verify Docker landed on an EC2 managed node:

```sh
mv ~/Downloads/ansible-jenkins.pem ~/.ssh
chmod 400 ~/.ssh/ansible-jenkins.pem
ssh -i ~/.ssh/ansible-jenkins.pem ec2-user@<ec2-ip-address>
docker version
docker compose version
```

![](./images/check-ec2-docker.png)

### Clean up

To avoid ongoing charges once the demo is done:

- Terminate the 2 EC2 instances
- Delete the `ansible-jenkins` EC2 key pair
- Destroy the `ansible-server` DigitalOcean droplet (and the Jenkins droplet if no longer needed)
- Remove the `ansible-server-key` and `ec2-server-key` credentials from Jenkins
