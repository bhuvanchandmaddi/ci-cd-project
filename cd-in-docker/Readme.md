## Step by step instructions to create CI/CD on container

Here the continious integration part remains same i.e creating maven build but the continious delivery changes i.e we deploy i to the docker machine by conainerizing the application.

### prerequisites

* Jenkins machine(We alrady have it from last class)
* Docker machine(spinup ec2 instance and install docker)

### Run the tomcat image(manually)

* ssh into docker machine
* Create a docker file with tomcat image and run it for testing

```
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps 
```
**Note**: The reason we copied contents form webapps.dist to webapps is that in container images all the webapps conents will be present in webapps.dist, so you ned to do that, else you may not be able to access the tomcat honepage after running the container

```
-> Build the image
docker build -t sampleapp:latest .

-> Run the container
docker run --name=sampleapp -d -p 8081:8080 sampleapp:latest
```

### containerize our hello world application(manuallay)

* From the jenkins master, we will get the application war archive, now we need to update the Dockerfile to copy the war(just copy the war form jenkins master for testing) into the conatainer. So the updated Dockerfile looks like this

```
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps 
ADD ./*.war webapps
```

* Build and run the image, so you could access the application.


### Integrating Jenkins with docker host
* After the war is built by the jenkins job, so one has to copy the war form jenkins machine to ansible machine.
* We use **Publish Over SSH** plugin, to copy the artifact. 
* After installing plugin, we need to configure the machine details in manage jenkins -> Configure system. You need to specify the hostname(or ip), user(dockeradm as shown below) and password etc
* Then in the build you need to specify the details like from where to where as shown below.
![publishoverssh]()
* We need to do above actions using non-root user.
```
# create a user
useradd dockeradm

#set the password
passwd dockeradm

# add the user to docker's group
usermod -a -G docker:dockeradm

# Enable passwordbased authenication
nano /etc/ssh/sshd_config and set as shoen below
Set PasswordBasedAuthentication yes

# Finally copy the Dockerfile to a location(Eg: /opt/docker) and change the ownership of that directy to newly created user.(This is because the Ansible machine copies artifacts to this machine by connecting to it using dockeradm user)

chown -R dockeradm:dockeradm /opt/docker
```

* Finally test the pipline, war file should be copied from jenkins machine to docker machine.


### Continious delivery of our application

* Once the artifact is copied, we need to build the docker image using above Dockerfile and then run it.

* Those commands will go into exec section of the plugin un pipeline configuration

```
cd /opt/docker;
docker build -t register:latest .;
docker stop registerapp;
docker rm registerapp;
docker run --name registerapp -d -p 8088:8080 register:latest
```
* Finally test the pipline by pushing changes to github and trigerring the pipeline. you should be able to access the application at below address

```
http://<public ip of docker machine>:8088
```

### problem with above approach
* It is not a good idea to hard core the steps in the jenkins configuration.
* We use ansible for image build part and pushing it to artifactory and docker machine to run the container as shown below
![ansible]()

### Ansible setup

* Create a Ec2 insance and install ansible
* Now create a user called ansibeadm(using root is not recommended)
```
# Create the user
useradd ansibleadm

# set the password
passwd ansibleadm

# enter below line in visudo, so thet every time we don't need to enter nonroot user password
visudo
ansible ALL=(ALL)       NOPASSWD: ALL

# switch to ansibleadm user and create ssh key pair
# we will copy this pub key into the machines to which we want to connect
su ansibleadm
ssh-keygen

# Enable passwordbased authenication
nano /etc/ssh/sshd_config and set as shoen below
Set PasswordBasedAuthentication yes

# we want to control docker-machine using this ansbile
# so copy the public key into the dockeradm user's authorized_key section

ssh-copy-id dockeradm@<private-ip-of-dockerhost>

# test ssh passwordless login
ssh -i /home/ansible/.ssh/id_rsa dockeradm@<private-ip-of-dockerhost>

```

* update the ansible inventory i.e(/etc/ansible/hosts) with docker host details and perform ping test
```
cat /ect/ansible/hsts
[docker] 
<private-ip> ansible_user=dockeradm ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
```
* anisble ping test: **ansible docker -m ping**

### Writing-playbooks
* Create ansible-playbooks for building the image,pushing it to the dockerhub an dexecuting them
* We will create 2 playbook, 1 for building the image and pushing it to dockerhub and other for running the containers
* Install docker on ansible machine as well
* we will execute first playbook in ansible-machine itself, which will build the image and push it to artifactory
```
- name: Execute ls command
  hosts: ansible
  gather_facts: false
  tasks:
  - name: Build the docker image
    command: docker build -t bmaddi/register:1 .
    args:
      chdir: /opt/ansible
  - name: push the image
    command: docker push bmaddi/register:1
```
* Before executing the playbook, we need to update the ansible inventory to add itself(i.e ansible controller) as it's slave.
* For that you need to copy the public key under ansibleadm user's ssh directory in authorized_keys file. This can be done easily using ssh-copy-id
```
ssh-copy-id ansibleadm@<ip-of-ansible-machine-itself>
```
* Finally update the inventory as shown below
```
[docker] 
<docker-machine-ip> ansible_user=dockeradm ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
[ansible]
<ansible-machine-ip> ansible_user=ansibleadm ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
```
* Another playbook to run the image on docker host.
```
- name: Execute ls command
  hosts: docker
  gather_facts: false
  tasks:
  - name: Stop the cotainer
    command: docker stop registerapp
    ignore_errors: true 

  - name: remove the  container
    command: docker rm registerapp
    ignore_errors: true

  - name: Remove the image
    command: docker rmi bmaddi/register:1
    ignore_errors: true

  - name: Run the container
    command: docker run --name=registerapp -d -p 8085:8080  bmaddi/register:1
```
* This playbook we want to run on the docker machine, which pulls the image and runs the conatiner.


### Finally update the pipeline

* Finally update the pipeline configuration to execute the 2 playbooks by adding below lines in exec block of publish over ssh.
```
cd /opt/ansible;
ansible-playbook imageBuilding.yaml;
sleep 10;
ansible-playbook runningContainer.yaml;
```



