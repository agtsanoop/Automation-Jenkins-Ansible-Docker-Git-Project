# Automation-Jenkins-Ansible-Docker-Git-Project
This is a DevOps jenkins freestyle project using Git, Jenkins, Ansible and Docker on AWS for deploying a python-flask application in a Docker container.The process should be initiated from a new commit to a specific branch of a GitHub repository. This event kicks off a process that begins building the Docker image. Jenkins supports this event-driven flow using the “GitHub hook trigger for GITScm polling".


# Architecture

![Arc](https://user-images.githubusercontent.com/87975855/172131199-e8147aa0-e1ac-4b30-a32e-f03bd7644ba4.png)

1. jenkins Server
2. Docker Image Build Server
3. Production/Test Server

## Installing / Configuring Jenkins, Git and Ansible

```
amazon-linux-extras install epel -y
yum install java-1.8.0-openjdk-devel -y
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum install jenkins git ansible -y
systemctl start jenkins.service
systemctl enable jenkins.service
```

## After the instalation, we can test the jenkins using the below port:

```
http://IP8080/   //provide your jenkins server Public IP
```
## Ansible Inventory file
```
[buildserver]
172.31.13.7 ansible_user="ec2-user"

[testserver]
172.31.4.79 ansible_user="ec2-user"
```
# Variable files

## packagevariable.yml
```
---
packages:
  - git
  - pip
  - docker
```
## variables.yml
```
---

git_repo_url: "git@github.com:agtsanoop/flask_app.git"
clone_directory: "/var/flaskapp/"
docker_login_user: ""   // Use you docker Hub user name
docker_login_pass: "password"         // user your docker password
docker_image: "sanoopagt/flaskapplication"
```

# Ansible Playbook
 
``` 
---

- name: "Building docker image on a build serveri using git repo"
  hosts: buildserver
  become: true
  vars_files:
    - packagevariable.yml
    - variables.yml

  tasks:

    - name: "Installing docker, Git and pip - BuildServer"
      yum:
        name: "{{packages}}"
        state: present

    - name: "Adding ec2-user to docker group - BuildServer"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: yes

    - name: "Installing docker-py python pacakge for docker using pip -BuildServer"
      pip:
        name: docker-py

    - name: "Starting/Enabling Docker service - BuildServer"
      service:
        name: "docker"
        state: restarted
        enabled: true

    - name: "Cloning image using git to {{clone_directory}} - BuildServer"
      git:
        repo: "{{git_repo_url}}"
        dest: "{{clone_directory}}"
      register: cloning_status

    - debug:
        msg: "{{cloning_status}}"


    - name: "Login to dockerHub - BuildServer"
      when: cloning_status.changed == true
      docker_login:
        username: "{{docker_login_user}}"
        password: "{{docker_login_pass}}"
        state: present

    - name: "Creating Docker image and pushing to docker hub - BuildServer"
      when: cloning_status.changed == true
      docker_image:
        source: build
        build:
          path: "{{clone_directory}}"
          pull: yes
        name: "{{docker_image}}"
        tag: "{{item}}"
        push: yes
        force_tag: yes
        force_source: yes
      with_items:
        - "{{cloning_status.after}}"
        - latest

    - name: "Removing the image from local sytem"
      when: cloning_status.changed == true
      docker_image:
        state: absent
        name: "{{docker_image}}"
        tag: "{{item}}"
      with_items:
        - "{{cloning_status.after}}"
        - latest

    - name: "Logout to dockerHub - BuildServer"
      when: cloning_status.changed == true
      docker_login:
        username: "{{docker_login_user}}"
        password: "{{docker_login_pass}}"
        state: absent



- name: "Running container on Test Instance"
  become: true
  hosts: testserver
  vars:
    packages :
      - docker
      - pip
    docker_image: "sanoopagt/flaskapplication"
  tasks:

    - name: "Installing docker and  pip module in test server"
      yum:
        name: "{{packages}}"
        state: present

    - name: "Adding ec2-user to docker group - test Server"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: yes

    - name: "Installing docker-py python pacakge for docker using pip -test Server"
      pip:
        name: docker-py

    - name: "Starting/Enabling Docker service - test Server"
      service:
        name: "docker"
        state: restarted
        enabled: true


    - name: "pulling image to test server"
      docker_image:
        name: "{{docker_image}}"
        source: pull
        force_source: yes
      register: dockerimage_status

    - debug:
        msg: "{{dockerimage_status}}"

    - name: "running container in test server"
      when: dockerimage_status.changed == true
      docker_container:
        name: flaskapplicaton
        image: "{{docker_image}}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:8080"
```

# Docker logins is encrypted using ansible vault

```
ansible-vault encrypt /var/deployment/variables.yml
New Vault password:
Confirm New Vault password:
Encryption successful

# ls -l /var/deployment/
-rw-r--r-- 1 root root    96 Mar 10 11:31 hosts
-rw------- 1 root root 16490 Mar 10 11:35 main.yml
# chmod 644 /var/deployment/main.yml  // to correct permission
```

# Configuring Jenkins

1. Login to Jenkins. http://Public_IP:8080/ password : /var/lib/jenkins/secrets/initialAdminPassword ( Initial password can be obtained from this location)

![jnk1](https://user-images.githubusercontent.com/87975855/172175906-c4653c3c-0e50-48f1-9d56-769a8074a31a.png)

2. Customise Plugins >> Install suggested Plugins

3. Create Jenkin admin user

4. Install Ansible plugin

   Manage Jenkins >> Plugin Manager >> Ansible >> Install without restart

    ![jnk2](https://user-images.githubusercontent.com/87975855/172176229-0795de60-0d41-4159-b197-62b093f2c7bd.png)
   
   Once the plugin installation is done you will need to restart jenkins.
   
5. Go to Manage Jenkins >> Global Tool Configuration >> Under Ansible [ Name : Ansible Path to ansible executables directory: /bin/]

6. To create a job

   New item >> Enter Item Name >> Freestyle Project >> OK   
   
   ![jkn3](https://user-images.githubusercontent.com/87975855/172176635-6012bff5-3324-4573-b279-0a127c356d3e.png)

   
  Add a decription "Docker Image Build and test project" >> Source Code Management select Git and add repo url >> Under Build, choose "Invoke ansible Playbook", then add Playbook path "/var/deployment/main.yml".

Add Inventory >> File or host list >> File path or comma separated host list "/var/deployment/hosts"

Under advanced section, make sure Disable the host SSH key check is disabled.

7. Under Vault Credentials >> add Jenkins >> refer screnshot to add vault logins >> Then under Vault credential choose "Vault password"
          
   ![jkn4](https://user-images.githubusercontent.com/87975855/172177194-8b61b763-93c5-4885-9dd8-454c3c2ced40.png)

   
   
