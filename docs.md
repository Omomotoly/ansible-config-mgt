# Ansible Configuration Management (Automating Projects 7–10

# Architectural Diagram

![screenshot](images/archi-diagram.png)
## Automation Steps

1\. Prepare the Jenkins instance as an Ansible control/jump host
2\. Wire it to GitHub, and
3\. Configure Jenkins to archive your Ansible repo on every change to main. 


## Step 1: Install & Configure Ansible on an EC2 Jenkins (Jenkins-Ansible) Jump Server
### 1. Rename the EC2 instance from Jekins to Jenkins-Ansible 

![screenshot](images/1.png)


### 2. Create GitHub repo ansible-config-mgt

![screenshot](images/2.png) 

Then clone the repo on the Jenkins-Ansible instance:

```
git clone https://github.com/Omomotoly/ansible-config-mgt.git
cd ansible-config-mgt
```
### 3. Install Ansible on Jenkins-Ansible (control node)
```
sudo apt update
sudo apt install -y ansible git
ansible --version
```
![screenshot](images/3.png) 
![screenshot](images/4.png) 
![screenshot](images/5.png)

###  4. Configure Jenkins build job to archive ansible-config-mgt repo

* **Create a Jenkins Freestyle project**

   * Log into your Jenkins dashboard → New Item.

   * Enter project name: ansible.

   * Inside project config:

* **Connect GitHub repo created earlier (ansible-config-mgt) SCM setup**

  * Under Source Code Management, choose Git.

  * Repository URL:

    ```
    https://github.com/<your-username>/ansible-config-mgt
    ```
  * Credentials:

    HTTPS: add a GitHub personal access token in Jenkins Credentials store.

  * Branch specifier:
    ```
    */main
    ```

![screenshot](images/6.png)
![screenshot](images/7.png)

* **Configure GitHub Webhook**

    **On the GitHub repo:**

    i. Go to Settings → Webhooks → Add Webhook.

   ii. Payload URL:

    ```
    http://<jenkins-ansible-server-public-IP>:8080/github-webhook/
    ```
  iii. Content type: application/json

   iv. Choose Just the push event.

    v. Save.

![screenshot](images/9.png)
![screenshot](images/10.png)

On Jenkins:

* In project config → Build Triggers → select:

GitHub hook trigger for GITScm polling

  This ensures Jenkins will build whenever there’s a push to main.

* **Configure Post-Build Archiving**

    Now, we want Jenkins to save the repo snapshot after every build:

   i. Scroll to Post-build Actions → select Archive the artifacts.

   ii. Files to archive:

    `**/*`

![screenshot](images/8.png)

Note: `**/*` means all files recursively in the workspace.

Artifacts will be stored here per build:

```
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```
* **Test the Setup**

1. Make a change in your repo:

```
echo "Testing CI $(date)" >> README.md
git add README.md
git commit -m "ci: test webhook build"
git push origin main
```
![screenshot](images/11.png)

2. GitHub webhook will fire → Jenkins job runs.

3. Check build log in Jenkins dashboard. 
   
![screenshot](images/12.png)

4. Verify artifacts:

```
ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```
You should see the full repo snapshot.

![screenshot](images/13.png)

At this point: every Git push to main in ansible-config-mgt → automatically triggers Jenkins → repo snapshot is archived as a build artifact.
## Jenkins Automation Scope

The Jenkins integration demonstrated in this project currently automates the **Continuous Integration (CI) workflow** for the Ansible repository.

The implemented workflow is:

```text
GitHub Push
    ↓
GitHub Webhook
    ↓
Jenkins Build
    ↓
Repository Checkout
    ↓
Artifact Archiving
```

A push to the `main` branch automatically triggers the Jenkins job. Jenkins checks out the latest version of the repository and archives the repository contents as build artifacts.

**Important:** At this stage, Jenkins is **not configured to automatically execute the Ansible playbook against the target infrastructure**. The Ansible playbook is executed separately from the Jenkins build using the Ansible control node.

Therefore, the automation demonstrated here covers **GitHub-triggered Jenkins builds and artifact archiving**, while the Ansible configuration management and infrastructure deployment are demonstrated separately in the Ansible execution steps.

This distinction ensures that the documented functionality matches the implementation and evidence provided in this project.

## Step 2 — Prepare Your Development Environment Using Visual Studio Code

### Install Visual Studio Code

On your local machine (Windows/Mac/Linux):

* Download VS Code from the official website: https://code.visualstudio.com

* Install with default options.

### Configure GitHub Integration in VS Code

To collaborate effectively, connect VS Code to your GitHub repo.

* Authenticate GitHub in VS Code

    * Open VS Code → View → Command Palette → search for GitHub: Sign in.

    * Sign in with your GitHub account (Personal Access Token or browser-based OAuth).

* Clone Repo Directly in VS Code (Optional)

    * Open Source Control tab → “Clone Repository” → paste repo URL.

    * Choose a local folder, VS Code auto-opens the repo.

![screenshot](images/14.png)


This ensures your local IDE is always in sync with GitHub.

### Clone Repo on Jenkins-Ansible Instance

Since Jenkins will actually run Ansible playbooks, we also need the repo available on the EC2 server. SSH into your Jenkins-Ansible instance and run:

```
git clone <ansible-config-mgt repo link>
```
Expected output:

Cloning into 'ansible-config-mgt'...
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
Unpacking objects: 100% (5/5), done.

Now verify:
```
cd ansible-config-mgt
ls
```
You should see your project files (e.g., README.md).

At this stage:

* You have VS Code installed locally with GitHub integration.
* Repo is cloned both on your local machine (for coding) and Jenkins-Ansible EC2 (for execution).

## Step 3 – Begin Ansible Development

1\. Create a New Branch for Feature Development

Working on the main branch is risky, as it may cause broken automation pipelines if unstable code is merged. Instead, we follow Git best practices: create feature branches for each task or module.

On your local machine or Jenkins-Ansible EC2 instance, run:

Navigate to your project directory
```
cd ansible-config-mgt
```
Create and switch to a new branch
```
git checkout -b dev-setup
```
```
git branch
```
![screenshot](images/15.png)

2\. Organize Your Project Structure

Run the following to create directories:

```
mkdir playbooks inventory
```

Your structure now looks like this:

```
ansible-config-mgt/
├── inventory/
├── playbooks/
├── README.md
```
3\. Create Your First Playbook: common.yaml

Inside the playbooks directory, create the file:
```
nano playbooks/common.yaml
```
4\. Create Environment-Specific Inventory Files

Inside the inventory directory, create four files:
```
cd inventory
```
```
touch dev.ini staging.ini uat.ini prod.ini
```
![screenshot](images/16.png)

Result:

We now have a Git-controlled project with a scalable structure.

Environments are isolated via inventory files.

common.yaml serves as your baseline for consistent infrastructure configuration.

## Step 4 – Setting Up an Ansible Inventory

1\. Enable SSH Agent and Add Your Key

Start the SSH agent and load your private key (the same one used when launching your EC2 servers):

### Start ssh-agent
```
eval `ssh-agent -s`
```
### Add your private key (replace with your path)
```
ssh-add ~/my-aws-key.pem
```
Confirm the key has been added:
```
ssh-add -l
```
![screenshot](images/17.png)

Expected output: fingerprint and name of your key (e.g., 2048 SHA256:xyz my-aws-key.pem).

2\. SSH into Jenkins-Ansible with Agent Forwarding

Agent forwarding (-A) allows your Jenkins-Ansible host to connect to other servers without re-entering the key:

```
ssh -A ubuntu@<jenkins-ansible-public-ip>
```
![screenshot](images/18.png)

1\. Update Inventory File

Inside your project repo:

```
cd ansible-config-mgt/inventory
nano dev.ini
```
Paste the following (replace <private-ip> values with your actual EC2 internal IPs):

```
[nfs]
<nfs server private ip address> ansible_ssh_user=ec2-user

[webservers]
<web server1 private ip address> ansible_ssh_user=ec2-user
<web server2 private ip address> ansible_ssh_user=ec2-user

[db]
<database private ip address> ansible_ssh_user=ec2-user

[lb]
<load balancer private ip address> ansible_ssh_user=ubuntu
```
![screenshot](images/19.png)

[nfs], [webservers], [db], [lb] are groups you can call in playbooks.
ansible_ssh_user tells Ansible which user account to use for SSH login.

At this point: Your dev environment is structured, and Ansible knows how to connect to each server group.

## Step 5 – Create a Common Playbook

1. Create the Playbook File

Inside the playbooks directory:

```
cd ../playbooks
nano common.yaml
```

Paste the following:

```
- name: update web, and nfs servers
hosts: webservers, nfs
become: yes
tasks:
- name: ensure wireshark is at the latest version
    yum:
    name: wireshark
    state: latest

- name: update LB and db server
hosts: lb, db
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
![screenshot](images/20.png)

2\. Extend the Playbook

You can add more common configuration tasks like creating directories, changing timezones, or running scripts. Example:

- name: Create a directory and file
    file:
    path: /opt/app/config.txt
    state: touch

- name: Change timezone to UTC
    command: timedatectl set-timezone UTC

- name: Run a shell script
    shell: /home/ubuntu/scripts/deploy.sh

![screenshot](images/21.png)

Result:

Wireshark is installed/updated on all servers.

RHEL servers use yum, Ubuntu servers uses apt.

Infrastructure is now consistent and repeatable, forming the baseline for future automation.

## Step 6 – Update Git with the Latest Code

1. Stage and Commit Code

From your project directory:

Check changes
```
git status  
```
Stage specific files (or use . to add all)
```
git add playbooks/common.yaml inventory/dev.ini  
```
Commit changes
```
git commit -m "Added common playbook and dev inventory"
```
![screenshot](images/22.png)

2. Push to Feature Branch

If you’re working on a feature branch (recommended):

```
git push origin dev-setup
```
![screenshot](images/23.png)

3. Create a Pull Request (PR)

Go to GitHub → open your repo → compare & create pull request from feature-ansible-setup into main.

Add context in the PR description (e.g., “Initial playbook and inventory setup”).

![screenshot](images/24.png) 
![screenshot](images/25.png)

4. Peer Review

As in real-world teams, code reviews are crucial. Another developer (or you wearing the reviewer’s hat) reviews the PR for:

Code quality (proper YAML formatting).

Correct inventory host definitions.

Clear commit messages.

If all looks good, approve and merge into main.

![screenshot](images/26.png) 
![screenshot](images/27.png)

5. Sync Local Repo with Main

After merging:

### Switch back to main
git checkout main  

### Pull latest merged changes
git pull origin main

![screenshot](images/28.png)

6. Jenkins CI Automation

Once merged, Jenkins (via the configured webhook) triggers a build automatically.

Jenkins checks out the updated repo.

![screenshot](images/29.png) 
![screenshot](images/30.png)

Build artifacts are stored in:

```
ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```
![screenshot](images/31.png)

## Step 7 – Run the First Ansible Test

Now it’s time to validate automation end-to-end by running the common.yaml playbook against dev servers.

### 1. Connect with VSCode Remote SSH

Using the Remote SSH plugin in VSCode, connect to your jenkins-ansible EC2 instance. This allows you to edit code locally in VSCode while executing commands directly on the server.

Edit config file
From your local machine

```
cd ~/.ssh
nano config
```
Add the following code: 
```
Host jenkins-ansible
    HostName <YOUR_EC2_PUBLIC_IP>
    User ubuntu
    IdentityFile ~/.ssh/your-key.pem
```
![screenshot](images/32.png)
![screenshot](images/33.png)

### 2. Execute the Playbook

Navigate into your repo and run the playbook:

```
cd ~/ansible-config-mgt
ansible-playbook -i inventory/dev.ini playbooks/common.yaml
```
* **Problem**: Ansible execution hangs
Running ansible-playbook hung indefinitely after displaying ok: [IP_ADDRESS] during host key verification.
* **Cause**: Ansible operates in non-interactive parallel threads. Because host key checking was active and the remote host keys were not yet accepted in known_hosts, Ansible paused while awaiting background interactive yes/no prompts.
* **Solution**: To resolve the non-interactive prompt block, I pre-populated the ~/.ssh/known_hosts file on the control node using ssh-keyscan to fetch and record public host keys for all managed inventory nodes before running playbooks.
```
sudo -u jenkins bash -c 'ssh-keyscan -H <TARGET_IPs> >> ~/.ssh/known_hosts'
```
Then run the ansible playbook command again

* **Issue**: File Creation Task Failure (/opt/app/config.txt)

* **Error**: Running the playbook failed at the "Create a directory and file" task with fatal: [IP]: FAILED! => {"msg": "Error, could not touch /opt/app/config.txt"}.
* **Cause**: The file module with state: touch attempts to create a file inside a directory path (/opt/app), but the parent directory /opt/app did not exist on the target remote hosts.
* **Resolution**: Updated the playbook to explicitly ensure the target directory exists (state: directory) before creating the file:

Edit the playbook file
```
- name: Ensure /opt/app directory exists
  file:
    path: /opt/app
    state: directory
    mode: '0755'

- name: Create config file
  ansible.builtin.file:
    path: /opt/app/config.txt
    state: touch
    mode: '0644'
```
Then run the ansible playbook command again

* **Issue**: Shell Script Execution Failure (deploy.sh)

* **Error**: Running the "Run a shell script" task failed across target servers with rc: 127 and No such file or directory.
* **Cause**: The shell module attempts to execute /home/ubuntu/scripts/deploy.sh locally on each remote server, but neither the directory nor the script existed on the target hosts.
* **Resolution**: Handled by creating the target directory and deploying a shell script with execute permissions (0755) prior to execution:

```
- name: Ensure deployment script exists
      ansible.builtin.copy:
        dest: /home/ubuntu/scripts/deploy.sh
        content: |
          #!/bin/bash
          echo "Deploy script executed successfully!"
        mode: '0755'
      notify: Run deployment script
```
Added this block at the bottom of the playbook at the handlers section:
```
handlers:
  - name: Run deployment script
    ansible.builtin.command:
      cmd: /home/ubuntu/scripts/deploy.sh
```
![screenshot](images/36.png)

![screenshot](images/37.png)


### 3. Verify Installation

On each target server (web, db, nfs, lb), confirm Wireshark was installed:

Check if Wireshark exists in PATH
```
which wireshark  
```
![screenshot](images/38.png)

At this point:

Jenkins has archived the updated automation code.

Ansible has successfully executed an automated configuration across multiple servers.

This workflow reflects a GitOps model—all infrastructure changes go through Git, are reviewed, and applied automatically through CI/CD pipelines. It ensures traceability, repeatability, and scalability.