# Ansible Configuration Management (Automate Project 7 to 10)
## Introduction

In Projects 7 to 10, I performed several manual tasks to provision virtual servers, install and configure software, and deploy web applications. Each server had to be configured individually from installing packages and editing configuration files to setting up shared storage, databases, and load balancers.

As infrastructure grows, performing these repetitive tasks manually becomes inefficient, time-consuming, and prone to errors. This project introduces Ansible Configuration Management, which automates routine administrative tasks using a declarative language, YAML, to define the desired state of infrastructure rather than executing commands manually on each server.
Ansible Client as a Jump Server (Bastion Host)

A Jump Server, also known as a Bastion Host, is an intermediary server that provides secure access to resources within an internal network. In a properly secured architecture, web servers reside in a private subnet and are not directly accessible from the internet — not even through direct SSH connections. Access is granted only through the Jump Server, which improves security by reducing the attack surface.

In this architecture, the Virtual Private Cloud (VPC) is divided into two subnets:

    Public Subnet: Contains resources with public IP addresses that are reachable from the internet.
    Private Subnet: Contains internal resources that are accessible only through private IP addresses and are not directly exposed to the internet.

Objectives

    Install and configure an Ansible client to function as a Jump Server (Bastion Host).
    Create and execute simple Ansible playbooks to automate server configuration tasks.

![screenshot](images/1.png)

Note: in this implementation, the DB Server actually runs Ubuntu (not RHEL 8 as shown), and was grouped with the Load Balancer under the apt-based play.
# Step 1 – Install and Configure Ansible on EC2 Instance.

1\. Rename your Jenkins EC2 instance.

![screenshot](images/2.png)

2\. Connect to the Jenkins-Ansible instance

![screenshot](images/3.png)

3\. Install Ansible

```
sudo apt update
sudo apt install ansible
```
4\. Then check the version
```
ansible --version
```
![screenshot](images/4.png)

5\. Configure Jenkins build job to archive your repository content every time you change it.

* Create a new Freestyle project ansible in Jenkins and point it to your 'ansible-config-mgt' repository.

![screenshot](images/5.png) 
![screenshot](images/6.png)
![screenshot](images/7.png)

* Configure a webhook in GitHub and set the webhook to trigger ansible build.

![screenshot](images/8.png)

* Test the setup by making some change in README.md file in main branch and make sure that builds starts automatically and Jenkins saves the files (build artifacts) in following folder

![screenshot](images/9.png)
![screenshot](images/10.png)

* Verified the archived artifacts from the successful build:

```
ls /var/lib/jenkins/jobs/ansible/builds/1/archive/
cat /var/lib/jenkins/jobs/ansible/builds/1/archive/README.md
```
![screenshot](images/11.png)

This confirms Jenkins successfully archived the updated README.md from the main branch after the webhook-triggered build.

# Step 2 - Prepare your development environment using Visual Studio Code
VS Code was already installed and configured with Remote-SSH access to the Jenkins-Ansible instance from prior setup, so this step required no additional configuration.

Clone down the ansible-config-mgt repo to the Jenkins-Ansible instance.

```
git clone https://github.com/Omomotoly/ansible-config-mgt.git
```
![screenshot](images/12.png)
# Step 3 - Begin Ansible Development.

1\. Create a feature branch In your ansible-config-mgt repo, 

Check out the newly created branch from main for development:
```
git checkout -b feature/prj-11-ansible-config
```

2\. Create a directory and name it playbooks – It will be used to store all your playbook files.

3\. Create a directory and name it inventory – It will be used to keep your hosts organised.

4\. Within the playbooks folder, create your first playbook, and name it common.yml.

5\. Within the inventory folder, create an inventory file () for each environment (Development, Staging, Testing and Production) dev, staging, uat, and prod respectively. These inventory files use .ini languages style to configure Ansible hosts.

![screenshot](images/13.png)

# Step 4 — Set Up an Ansible Inventory

An Ansible inventory defines the hosts and groups of hosts on which commands, modules, and tasks in a playbook operate. Since our intention is to execute Linux commands on remote hosts, it's important to have a way to organize our hosts in an inventory.

Loaded the private key into ssh-agent on the local machine and connected to the Jenkins-Ansible server with agent forwarding.

Ansible uses TCP port 22 by default, which means it needs to reach the target servers via ssh — from Jenkins-Ansible. For this, you'll implement the concept of ssh-agent, so you don't need to import your private key into ssh-agent manually.

* On your local machine (not the Jenkins-Ansible server):

```
eval `ssh-agent -s`
ssh-add server-key.pem
```

* Confirmed the key was successfully forwarded to the Jenkins-Ansible server:

```
ssh-add -l
```
![screenshot](images/ssh-agent.png)

* Updated inventory/dev.ini with the private IP addresses of the servers from Projects 7–10:
```
[nfs]
172.31.23.26 ansible_ssh_user=ec2-user

[webservers]
172.31.29.216 ansible_ssh_user=ec2-user
172.31.18.110 ansible_ssh_user=ec2-user

[db]
172.31.29.78 ansible_ssh_user=ec2-user

[lb]
172.31.46.134 ansible_ssh_user=ubuntu
```

![screenshot](images/14.png)

# Step 5 — Create a Common Playbook

Wrote instructions for Ansible to perform on all servers listed in inventory/dev.yml. The common.yml playbook holds configuration for repeatable, re-usable, multi-machine tasks common to all systems in the infrastructure.

1\. Updated playbooks/common.yml with the following code:

```
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  become: yes
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest

- name: update LB server
  hosts: lb
  become: yes
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```

![screenshot](images/15.png)

This playbook is split into two plays: the first installs/updates wireshark on the RHEL 8 servers (webservers, nfs, db) using yum; the second does the same on the load balancer (lb) using apt, after refreshing the package cache. Both plays use become: yes to run as the root user.

2\. Extend the Playbook (Optional Tasks)
You can add more common configuration tasks like creating directories, changing timezones, or running scripts. Example:

![screenshot](images/16.png)

This approach highlights a key Ansible principle: separating inventory from playbooks. The same common.yaml can later run against staging or prod inventories without code duplication — ensuring scalability and environment parity
# Step 6 - Update GIT with the latest code

At this point, all directories and files existed locally on the Jenkins-Ansible server, and needed to be pushed to GitHub.

In a real-world team setting, it's important to collaborate using Git properly. Many organizations enforce a rule that no code is deployed before it's been reviewed by an extra pair of eyes — known as the "Four Eyes Principle." Since a separate feature branch had already been created, the next step was to raise a Pull Request (PR), have the branch peer-reviewed, and merge it into the main branch.

Commit your code into GitHub:

    Use git commands to add, commit and push your branch to GitHub.

![screenshot](images/*.png)26 ![screenshot](images/*.png)27 ![screenshot](images/*.png)28

    Create a Pull Request (PR)

![screenshot](images/*.png)30 ![screenshot](images/*.png)31 ![screenshot](images/*.png)32

    Checkout from the feature branch into the main, and pull down the latest changes.

![screenshot](images/*.png)33

    After merging the pull request into main, Jenkins automatically triggered a new build (build #3) via the GitHub webhook, confirming the CI pipeline correctly picks up changes from main.

![screenshot](images/*.png)34

Confirm the archive actually contains your new files, not just README.md

ls /var/lib/jenkins/jobs/ansible/builds/3/archive/

![screenshot](images/*.png)35

This confirms Jenkins correctly picked up and archived the full updated repository content from main, including the new inventory/ and playbooks/ directories.
Step 7 — Run the First Ansible Test

Run the playbook:

ansible-playbook -i inventory/dev.yml playbooks/common.yml

![screenshot](images/*.png)D

Troubleshooting note: The first run failed on the DB server with Could not detect which major revision of yum is in use, since the DB Server actually runs Ubuntu (not RHEL, as originally assumed). Fixed by restructuring common.yml to group db with lb under the apt play instead of the yum play.

![screenshot](images/*.png)E

Re-ran the playbook — completed successfully with failed=0 across all 5 hosts.

![screenshot](images/*.png)36 ![screenshot](images/*.png)37

Verified Wireshark was installed on every host:

ansible all -i inventory/dev.yml -a "which wireshark"

![screenshot](images/*.png)38

Output confirmed /usr/bin/wireshark on all 5 servers.
