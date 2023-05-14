## Deploy application to kubernetes cluster

![Flow](https://github.com/bhuvanchandmaddi/ci-cd-project/blob/main/.images/k8clusterflow.PNG?raw=true)

### Create a Eks bootstrap machine
* Create a Ec2 instance
* Install awscli and configure it aws secret and access keys
* Install eksctl to create EKS cluster and create EKS cluster
* Install the kubectl to talk to the kubernetes cluster
* Also create a user, Let's say k8user
```
# create a user
useradd k8user

#set the password
passwd k8user

# Enable passwordbased authenication
nano /etc/ssh/sshd_config and set as shoen below
Set PasswordBasedAuthentication yes

# Execue visudo
visudo
```

### Add this machine as slave to ansible controller

* Connect to Ansible controller and copy the public key of ansible user to newly created EKS bootsrap machine's k8user using ssh-copy-id

```
ssh-copy-id k8user@<ip-of-the- Eks bootstrap machine>
```

* Add the entry in ansible inventory.
```
[docker] 
<docker-machine-ip> ansible_user=dockeradm ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
[ansible]
<ansible-machine-ip> ansible_user=ansibleadm ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
[Kubernetes]
<k8-machine-ip> ansible_user=k8user ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
```

* Test the connectivity using ping: ansible kubernetes -m ping

## Write the kubernetes manifest for application

* Deployment manifest
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: valaxy-deployment
spec:
  selector:
    matchLabels:
      app: valaxy-devops-project
  replicas: 2 # tells deployment to run 2 pods matching the template
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

  template:
    metadata:
      labels:
        app: valaxy-devops-project
    spec:
      containers:
      - name: valaxy-devops-project
        image: bmaddi/register:1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```
* Service manifest
```
apiVersion: v1
kind: Service
metadata:
  name: valaxy-service
  labels:
    app: valaxy-devops-project
spec:
  selector:
    app: valaxy-devops-project
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 31200
```

* Create them in /opt/k8 folder
* Test them by executing kubectl apply

### Ansible playbook to run deployment

* create this place in ansible machine and place it in /opt/ansible
```
- name: Execute ls command
  hosts: k8
  user: root
  
  gather_facts: false
  tasks:
  - name: Apply the deployment
    command: kubectl --kubeconfig=/root/.kube/config apply -f /opt/k8/Deployment.yaml

  - name: Apply the service
    command: kubectl --kubeconfig=/root/.kube/config apply -f /opt/k8/Service.yaml

  - name: update deployment with new pods if image updated in docker hub
      command: kubectl rollout restart deployment.apps/valaxy-regapp

```
* Test this playbook by running it manually, it should create Deployment and Service manifest in k8 cluster.


### Configure CD pipeline in Jenkins

* Create a new jenkins pipeline to do this CD job.
* It should run a simple task i.e connec to ansible server and execute the playbook
* Configure system, with Eks bootstrap machine details in jenkins.
* Configure the pipeline to create only action i.e using publish over plugin's exec bloc run the playbook.

```
ansible-playbook /opt/ansible/deployk8manifests.yaml -b --become-user=root
```

### Create a CI pipeline in jenkins

* Copy the earlier pipeline(where we deployed to docker conatiner).
* All the steps remains the same. only one change. It used to execute 2 playbooks
    1. imageBuilding.yaml - For building image and push it to docker-hub
    2. runningContainer - For running a conatiner

* Just remove the runningContainer.yaml playbook execution form the exec block.

### Update the CI pipeline

* Update the CI pipeline to execute the  CD pipeline once CI executes fine

* Edit CI pipeline -> Post-build Actions -> Projects to build -> Select the CD project

* Execute the complete pipeline. By running a build after making any sample change in maven application.

