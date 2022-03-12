# jenkins-ansible-docker-project

## Description:-

This is a DevOps CI/CD pipeline using Git, Jenkins, Ansible and Docker on AWS for deploying a python-flask application in a Docker container.

The process should be initiated from a new commit to a specific branch of a GitHub repository. This event kicks off a process that begins building the Docker image. Jenkins supports this event-driven flow using the “GitHub hook trigger for GITScm polling".

## Architecture

![draw](https://user-images.githubusercontent.com/97517424/158003817-bb60b4aa-5aac-4bb0-9a12-f9f857c7226a.png)

## Setup
1. CI/CD Server
2. Docker Image Build Server
3. Production/Test Server

Here we use amazon linux on these 3 servers.

## CI/CD Server

**Installing Jenkins:-**
```html
# amazon-linux-extras install epel -y
# yum install java-1.8.0-openjdk-devel -y
# wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
# rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
# yum install jenkins -y
# systemctl start jenkins.service
# systemctl enable jenkins.service
```

**Installing Git and Ansible:-**
```html
# yum install git -y
# amazon-linux-extras install ansible2 -y
```

## Ansible Inventory

```html
[build]
172.31.38.221  ansible_user="ec2-user"

[test]
172.31.42.62 ansible_user="ec2-user"
```

## Ansible Playbook

```html
---
- name: "Building docker image on 'Build' server from Git repo"
  hosts: build
  become: true
  vars:
    packages:
      - docker
      - git
      - python-pip
    git_repo_url: "https://github.com/anushaksv/devops-flask.git"
    clone_dir: "/var/flask_app/"
    docker_user: anushaksv
    docker_pass: **********
    image_name: anushaksv/flask

  tasks:

    - name: "Build - Installing docker, pip & git"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Build - Adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true

    - name: "Build - Installing python2 extension For Docker"
      pip:
        name: docker-py

    - name: "Buid- Restarting/Enabling Docker service"
      service:
        name: docker
        state: restarted
        enabled: true

    - name: "Build - Clonning Repo {{ git_repo_url }}"
      git:
        repo: "{{ git_repo_url }}"
        dest: "{{ clone_dir }}"
      register: clone_status

    - name: "Build - Loging into Docker-hub"
      when: clone_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_pass }}"
        state: present
            
    - name: "Build - Creating docker image and pushing to docker hub"
      when: clone_status.changed == true
      docker_image:
        source: build
        build:
          path: "{{ clone_dir }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ clone_status.after }}"
        - latest  

    - name: "Build - Deleting pulled images from build server"
      when: clone_status.changed == true
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ clone_status.after }}"
        - latest

    - name: "Build -Logout from docker hub"
      when: clone_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_pass }}"
        state: absent

- name: "Pulling builded image and running container on 'Test' server"
  hosts: test
  become: true
  vars:
    docker_image: "anushaksv/flask"
    packages:
      - docker
      - python-pip

  tasks:

    - name: "Test - Installing docker and pip "
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Test - Adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true

    - name: "Test - Installing python2 extension For Docker"
      pip:
        name: docker-py

    - name: "Test - Restarting/Enabling Docker service"
      service:
        name: docker
        state: restarted
        enabled: true
        
    - name: "Test - Pulling Docker image"
      docker_image:
        name: "{{ docker_image }}"
        source: pull
        force_source: yes
      register: image_status      
    
    - name: "Test - Running container"
      when: image_status.changed == true     
      docker_container:
        name: flaskapp
        image: "{{ docker_image }}:latest"
        recreate: yes
        published_ports:
          - "80:5000"

    - name: "Email alert"
      when: image_status.changed == true
      mail:
        host: smtp.gmail.com
        port: 587
        secure: starttls 
        username: ******@gmail.com
        password: ********************       --------------------------------->app password
        to: Anusha <*****@gmail.com>
        subject: Project Notification
        body: New Github code commit found. Image builed and tested.
```

## Configuring Jenkins

1. Install Ansible plugin in Jenkins through "Manage Jenkins" option
2. Add Ansible binary path through Jenkins Global Tool Configuration
3. Create new Job in Jenkins by doing the following steps
- Add the GitHub repository URL
- Add SSH private key and Ansible Vault Credentials
- Select GitHub hook trigger for GITScm polling
- Select Build as Ansible
- Add Playbook path as /var/deployment/main-playbook.yml
- Disable the host SSH key check

## Jenkins manual Build

Once the Job is created, click on "Build now" and check the Console Output and verify everything is fine.

## Automatically trigger each build on the Jenkins server, after each Commit on your Git repository

For this, add the Jenkins Payload URL in GitHub repository's Webhook

![webhook](https://user-images.githubusercontent.com/97517424/158008753-81c8ec13-1c16-4ef3-9980-e8200ca283c6.png)

## Check the console output through Jenkins after a new commit on your Git repository

![jenkins3](https://user-images.githubusercontent.com/97517424/158008815-9ff099e5-57c4-4725-85f9-6531e33c23d5.png)

![jenkins4](https://user-images.githubusercontent.com/97517424/158008827-67e85daf-e9dc-4989-88a2-a24f2ba85283.png)

## Result

A new image is pushed to Docker Hub from the 'Build' host, which is pulled by 'Test' host to create a Docker Container.

![dockerhub](https://user-images.githubusercontent.com/97517424/158009154-72a5820a-c508-42cd-9e3b-17560c627512.png)
