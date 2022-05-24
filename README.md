# continuous-deployment-of-docker-container-using-ansible-and-jenkins

## Description
This is a DevOps jenkins freestyle project using Git, Jenkins, Ansible and Docker on AWS for deploying a python-flask application in a Docker container.The process should be initiated from a new commit to a specific branch of a GitHub repository. This event kicks off a process that begins building the Docker image. Jenkins supports this event-driven flow using the “GitHub hook trigger for GITScm polling".

## Requirements
1. jenkins Server
2. Docker image build server
3. docker container production/test server

### Jenkins Server
Iam using an ec2 instance installed with amazone linux for this.

### Jenkins Installation and configuration.

~~~
amazon-linux-extras install epel -y
yum install java-1.8.0-openjdk-devel -y
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum install jenkins -y
systemctl start jenkins.service
systemctl enable jenkins.service
~~~

Once the installation is been completed, jenkins web interface will be available on port 8080

~~~
http://13.233.16.92:8080/
~~~

The url will be loaded like below.

![image](https://user-images.githubusercontent.com/100774483/170122856-b3d228bd-f31b-4b46-b038-2e03dc001cb2.png)


We need to unlock jenkins webinterface with a password which can be taken from below location on server.

~~~
/var/lib/jenkins/secrets/initialAdminPassword
~~~

Unlock the jenkins and install the suggested plugins

![image](https://user-images.githubusercontent.com/100774483/170123177-9c2b34eb-61db-44a0-ac4f-d4a2450f34a6.png)


Pluggins will start to install now.

![image](https://user-images.githubusercontent.com/100774483/170123336-bbc655a7-52b2-4bec-812f-014dc091f58b.png)

Once installation is been cmpleted, it will be headed to the user creation.

![image](https://user-images.githubusercontent.com/100774483/170123560-09868ccf-50ea-4190-8308-149dcb13b586.png)

Once the detaisl are entered, click save and continue

Jenkins will  be ready now. 

![image](https://user-images.githubusercontent.com/100774483/170123822-8aaaee2c-300e-41df-a6d7-8731a5e0c877.png)

Click "start using jenkins"

### installing ansible plugin on jenkins now

~~~
1. Go to Manage jenkins 
2. Manage plugins 
3. Availble plugins 
4. Search for ansible and slect the ansible 1.1 
5. Click "install without restart".
~~~

![image](https://user-images.githubusercontent.com/100774483/170124712-2ac732f0-c865-4752-b731-adca23e66538.png)

On the next page, select "Restart jenkins when installation is complete and no job are running"

![image](https://user-images.githubusercontent.com/100774483/170125016-b91dc75b-9f46-4389-bc56-7f1a78d20a73.png)


![image](https://user-images.githubusercontent.com/100774483/170125123-b6f65273-698f-4c45-90b6-b3747db8e701.png)

### installing ansible and git on jenkins server

~~~
amazon-linux-extras install ansible2 -y
yum install git -y
~~~

~~~
which ansible
/bin/ansible
~~~

~~~
ansible --version
ansible 2.9.23
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /bin/ansible
  python version = 2.7.18 (default, Jun 10 2021, 00:11:02) [GCC 7.3.1 20180712 (Red Hat 7.3.1-13)]
  ~~~~
  
  ### Configuring ansible on jenkins
  
  1. Login to jenkins
  2. Select "Manage jenkins"
  3. Select "global tool configuration"
  4. Go to "ansible" section and click "add ansible"
  5. Provide a name on "name" field and the path of ansible installation directory(/bin/) on "path to ansible executable directory"
  6. Click "save"

Now, ansibel is been configured on jenkins.

### Creating Ansible playbook 

~~~
vim /var/deployment/main.yml

---
- name: "Building Docker Image From Grihub Repository"
  hosts: build
  become: true
  vars:
    packages:
      - git
      - pip
      - docker
    project_repo_url: "https://github.com/Jibincl/git-flask-app.git"
    clone_dir: "/var/flask_app/"
    docker_user: "jibincl"
    docker_password: "********"
    image_name: "jibincl/oncompute"
        
  tasks:
                  
    - name: "Build - Installing packages"
      yum:
        name: "{{ packages }}"
        state: present
    
    - name: "Build - Adding Ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true
            
            
    - name: "Build - Installing Python Extension For Docker"
      pip:
        name: docker-py
            
    - name: "Build - Restarting/Enabling Docker Service"
      service:
        name: docker
        state: restarted
        enabled: true
            
    - name: "Build - Clonning Repo {{ project_repo_url }}"
      git:
        repo: "{{project_repo_url}}"
        dest: "{{ clone_dir }}"
      register: clone_status
        
        
    - name: "Build - Loging to Docker-hub Account"
      when: clone_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: present
    
    
    - name: "Build - Creating Docker Image And Push To Docker-hub"
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
      
    
    - name: "Build - Deleting Local Image From Build Server"
      when: clone_status.changed == true
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ clone_status.after }}"
        - latest
    
        
    - name: "Build - Logout to Docker-hub Account"
      when: clone_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: absent
       
    

- name: "Running Image On The Test Server"
  hosts: test
  become: true
  vars:
    image_name: "jibincl/oncompute"
    packages:
      - docker
      - pip
  tasks:
    

    - name: "Test - Installing Packages"
      yum:
        name: "{{ packages }}"
        state: present
    
    - name: "Test - Adding Ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true
            
            
    - name: "Test - Installing Python Extension For Docker"
      pip:
        name: docker-py
            
            
    - name: "Test - Docker service restart/enable"
      service:
        name: docker
        state: started
        enabled: true
    
    
    - name: "Test - Pulling Docker Image"
      docker_image:
        name: "jibincl/oncompute"
        source: pull
        force_source: true
      register: image_status      
    
    - name: "Test- Run Container"
      when: image_status.changed == true     
      docker_container:
        name: flaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:5000"
~~~


### Hosts file

~~~
vim /var/deployment/hosts

[build]
3.133.119.157  ansible_user="ec2-user"  ansible_ssh_private_key_file="/var/deployment/key.pem"


[test]
18.217.42.228 ansible_user="ec2-user"  ansible_ssh_private_key_file="/var/deployment/key.pem"
~~~


### As the playbook is having sensitive data like passwords, it will be better to keep it encrypted.

~~~
ansible-vault encrypt /var/deployment/main.yml 
New Vault password: 
Confirm New Vault password: 
Encryption successful
~~~

### Running ansible playbook through jenkins

1. Login to jenkins
2. Go to New item 
3. provide a job name and select "Free style project" 
4. Click Ok

![image](https://user-images.githubusercontent.com/100774483/170128802-55c876c2-4be6-4045-8cd8-f37d4e46ce73.png)


It will ask a series of question: 
1. Provide description and select source code as GIT. 
2. Provide the git repo url of jenkins and provide the correct branch of your git 
3. Add build step as "Invoke ansible playbook" 
4. Provide the playbook path as "/var/deployment/main.yml" 
5. Add the inventory file under "File or host list" as "/var/deployment/hosts" 
6. Add the ssh key with private key too for the build and test server 
7. Add the ansible vault password with jenkins to access the playbook. 
![image](https://user-images.githubusercontent.com/100774483/170129944-abebb522-da1a-4259-9cb7-39724523a128.png)
8. Select the entered vault password on "vault credentials", go to advanced and disable the host ssh key check too. 
9. Save the settings

### Jenkins manual Build

Once the Job is created, click on "Build now" and check the Console Output and verify everything is fine

![image](https://user-images.githubusercontent.com/100774483/170130757-067cb89a-25d9-41bb-9214-784b56a5e3b8.png)

Now, a container will be crated on test server from the docker file on git repo. 

### Setting up automatic deployment once git repo is modified

1. Go tto git repo
2. Select "webhooks" from "settings"

![image](https://user-images.githubusercontent.com/100774483/170131805-2a60de35-8a90-4ef4-bfad-204e9a13419a.png)

3. Provide the below on webhook field and save settings

~~~
http://13.233.16.92:8080/github-webhook/
~~~


### Reconfiguring project

~~~
1. Go to project and click "configure"
2. Go to "Build triggers"
3. Tick on "GitHub hook trigger for GITScm polling"
4. Click "Save"
~~~

~~~
By configurong this "build trigger", jenkins will run the playbook whenever it gets a webhook from the respective git repo. 
Once the GIT repo is modified, the deployment will be take place automatically now.
~~~

![image](https://user-images.githubusercontent.com/100774483/170133218-c2167fb2-6b22-495d-b518-5158d5d30865.png)


## Conclusion

In this tutorial, we discussed about doing an automatic deployment for a docker container using jenkins, ansible, git  once the repo is modified






