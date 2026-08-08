test change for project 12
Project 11: Ansible Configuration Management & Bastion Host Setup
Executive Summary
In Projects 7 through 10, setting up virtual servers, configuring databases, and deploying web applications involved repetitive manual SSH connections and commands. Project 11 automates these operations using Ansible Configuration Management through a declarative YAML code architecture. A central Jenkins-Ansible Control Node acts as a Jump Server (Bastion Host) to securely configure remote RHEL 8 and Ubuntu targets across private subnets.


<img width="856" height="662" alt="image" src="https://github.com/user-attachments/assets/47c2157d-d660-4bb0-baac-3400eae69fdc" />


🛠 Step-by-Step Implementation Guide
Step 1 — Install and Configure Ansible on Control Node
 1-Updated EC2 instance tag to Jenkins-Ansible (ip-172-31-32-54).

 2-Installed Ansible core on the control node:

   sudo apt update
   sudo apt install ansible -y
   ansible --version
 3- Verification: Confirmed Ansible Core 2.16.3 with Python 3.12.3 execution environment.

 Step 2 — Local Development Setup (VS Code & SSH Agent Forwarding)
   1-SSH Key Addition: Added the private key (Project11 Jenkins-Ansible.pem) to the local Windows PowerShell ssh-agent:
      PowerShell
    
    Start-Service ssh-agent
    Set-Service -Name ssh-agent -StartupType Automatic
    ssh-add "C:\Path\To\Project11 Jenkins-Ansible.pem"
    ssh-add -l

    1- GitHub Authentication: Authenticated VS Code with GitHub to enable seamless push/pull functionality directly from the local workspace.

Step 3 — Workspace & Directory Structure Setup
 Inside the repository Project11-ansible-config-mgt, created the standard directory structure:

  -playbooks/: Stores orchestration playbooks (common.yml).

  -inventory/: Stores environment configurations (dev, staging, uat, prod).

  Project11-ansible-config-mgt/
├── inventory/
│   ├── dev
│   ├── prod
│   ├── staging
│   └── uat
├── playbooks/
│   └── common.yml
└── README.md

Step 4 — Configure Ansible Inventory (inventory/dev)
Mapped target server IPs to their respective host groups and configured SSH remote users (ec2-user for RHEL, ubuntu for Ubuntu):

Ini, TOML
[nfs]
54.147.28.40 ansible_ssh_user=ec2-user

[webservers]
100.26.29.45 ansible_ssh_user=ec2-user
32.199.167.150 ansible_ssh_user=ec2-user

[db]
3.48.184.62 ansible_ssh_user=ec2-user

[lb]
34.234.95.26 ansible_ssh_user=ubuntu

Step 5 — Jenkins CI/CD Automation & GitHub Webhooks Integration
-Created Freestyle project ansible pr11 in Jenkins linked to https://github.com/amarsaleem333/Project11-ansible-config-mgt on branch */main.

-GitHub Webhook: Configured Payload URL (http://3.89.199.93:8080/github-webhook/) with content type application/json.

-Payload Verification: Verified delivery payload status returning HTTP 200 (green checkmark).

-Jenkins Build Trigger & Post-Build Archiving: Enabled GitHub hook trigger for GITScm polling and added Archive the artifacts with pattern **.

-Automated CI Test: Pushed changes to GitHub, triggering Jenkins Build #3, which fetched changes and archived build artifacts (Finished: SUCCESS).

Step 6 — Common Playbook Creation (playbooks/common.yml)
Wrote multi-OS tasks utilizing yum for RHEL hosts and apt for Ubuntu host:

YAML

---
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

Step 7 — Playbook Execution & Validation
Ran the playbook from the control node against internal VPC target IPs (172.31.23.26, 172.31.20.222, 172.31.23.88, 172.31.17.253):

Bash
cd Project11-ansible-config-mgt
ansible-playbook -i inventory/dev.yml playbooks/common.yml

Step 8 — AWS Resource Management
All 7 EC2 instances (NFS, Web1, Web2, DB, Apache LB, Nginx LB, Jenkins-Ansible) were safely stopped to prevent unneeded AWS charges.

<img width="1024" height="838" alt="image" src="https://github.com/user-attachments/assets/760e52ba-5c84-4062-b2fe-0676b3db2053" />

Project 11 Issues & Workarounds

Issue 1: SSH Permission Denied Across Internal Targets

Obstacle: The Ansible control node couldn't authenticate to private target instances (172.31.23.26, 172.31.20.222, 172.31.23.88, 172.31.17.253) without copying private .pem keys onto remote servers.

Workaround: Configured local Windows ssh-agent forwarding on the workstation using ssh-add "Project11 Jenkins-Ansible.pem". This allowed the control node to act as a secure Jump Server (Bastion Host) using keyless forwarding.

Issue 2: Heterogeneous OS Package Manager Conflicts (RHEL vs. Ubuntu)

Obstacle: Target nodes ran different Linux distributions (RHEL 8 for NFS/Web/DB and Ubuntu 24.04 for the Load Balancer), causing single package manager commands (yum or apt) to fail on opposite OS targets.

Workaround: Structured playbooks/common.yml into distinct target host plays—using yum for the webservers, nfs, and db groups, and apt (with update_cache: yes) for the lb group.

Issue 3: AWS Security Group Connection Timeouts

Obstacle: Playbook execution hung indefinitely during target fact gathering due to blocked internal traffic on port 22 within the VPC.

Workaround: Updated inbound VPC Security Group rules on target EC2 instances to explicitly permit SSH (Port 22) traffic originating from the Control Node's private IP (172.31.32.54/32).

Issue 4: Jenkins Artifact Archiving Failures

Obstacle: Jenkins builds passed SCM updates but failed to record repository state changes for verification reports.

Workaround: Configured the Post-build Action Archive the artifacts with wildcard path ** to recursively record all inventory files, playbooks, and directory structures per build trigger.





      
  

    
