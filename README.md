End to End Project
===================

Create 5 instances
1. Jenkins
2. Controller
3. QA server1
4. QA Server2
5. EKS or Kops Server

Do ssh Connection between QA servers and Controller
====================================================

connect to QA1 via Git bash
----------------------------
whoami
sudo passwd ubuntu
give password
sudo vim /etc/ssh/sshd_config
here change password authentication NO --> YES
come out of file {esc :wq}
sudo service ssh restart
exit

Connect to QA2 via Another git bash
------------------------------------
whoami
sudo passwd ubuntu
give password
sudo vim /etc/ssh/sshd_config
here change password authentication NO --> YES
come out of file {esc :wq}
sudo service ssh restart
exit

connect Controller via Git Bash
-------------------------------
whoami
ssh-keygen
ssh-copy-id ubuntu@private ip of QA1
give ur password of QA1
ssh-copy-id ubuntu@private ip of QA2
give ur password of QA2

install ansible in controller 
------------------------------
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo add-apt-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible

ansible --version
sudo vim /etc/ansible/hosts

Paste Private ips of QA1 and QA2

ansible all -a "date"

it will execute in 2 servers of QA1 and QA2

in controller
--------------

whoami
sudo passwd ubuntu
give password
sudo vim /etc/ssh/sshd_config
here change password authentication NO --> YES
come out of file {esc :wq}
sudo service ssh restart
exit
====================================================================================================

connect to jenkins via Git bash
--------------------------------

whoami
sudo su - jenkins
ssh-keygen
ssh-copy-id ubuntu@private ip of controller
give ur password of controller

sudo apt-get update
sudo apt-get install -y openjdk-11-jdk
sudo apt-get install -y git maven

{for jenkins permanent installation:}

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins

jenkins --version

copy public ip and paste it in browser:8080

sudo vim give that path for password

Jenkins will open do necessary plugins and requirements

==========================================================================================================

install EKS setup on EKS server
-------------------------------


curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin 
kubectl version --short --client

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

Create an IAM Role and attache it to EC2 instance

to create clusters

eksctl create cluster --name <Cluster-name> \
   --region ap-south-1 \
--node-type t2.medium \

eksctl delete cluster <cluster-name> --region <region-name>
-------------------------------------------------------------------------------------------------------
DO ssh Connection between EKS and Jenkins
------------------------------------------
whoami
sudo passwd ubuntu
give password
sudo vim /etc/ssh/sshd_config
here change password authentication NO --> YES
come out of file {esc :wq}
sudo service ssh restart
exit

connect to Jenkins

whoami
sudo su - jenkins
ssh-keygen
ssh-copy-id ubuntu@private ip of controller
give ur password of controller

=========================================================================================================


go to Jenkins DashBoard

clink on Newitem

give name for the project (End-to-End-project)

select Pipeline Project

ok
---------------------------------------------------------------------------------------

click on project
click on configure
click on pipeline section
here you can create Scripted or Declarative Pipeline

pipeline
{
    agent any
    stages
    {
        stage("Continious Download")
        {
            steps
            {
                git ' https://github.com/intelliqittrainings/maven.git'
            }
        }
        stage("Continious Build")
        {
            steps
            {
                sh 'mvn package'
            }
        }

execute for build purpose

if it is Successfully completed
========================================================================================================
connect to jenkins via git bash and install docker

curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

docker --version

create a tomee container here

docker run --name t1 -d -P tomee
docker container ls
docker exec -it t1 bash
ls
cd webapps
ls
pwd
copy this path (for dockerfile purpose)
exit

docker rm -f t1
docker system prune -af

==========================================================================================================

Go to console output and copy the path of Building.war

we are coping the webapp file from /var/lib/jenkins/workspace/endtoend/webapp/target/webapp.war this location to /var/lib/jenkins/workspace/endtoend

create a dockerfile

cat > dockerfile << EOF

FROM tomee
MAINTAINER intelliqit
COPY <paste the building.war path>  <paste the tomee path>
EOF

ls

go to pipeline syntax and select "Shell Script"

paste the above dockerfile and convert it into groovy code

stage("dockerfile design")
        {
            steps
            {
               sh 'cp /var/lib/jenkins/workspace/endtoend/webapp/target/webapp.war .' 
               
               sh '''cat > dockerfile << EOF
                     FROM tomee
                     MAINTAINER ravindra
                     COPY  webapp.war  /usr/local/tomee/webapps/testapp.war'''
            }
        }

[this path /usr/local/tomee/webapps/ is taken from docker run --name t1 -d -P tomee}

Execute for Build, if it is successfull then push the image into docker registy

stage("Docker image Creation")
        {
            steps
            {
                sh 'sudo docker build -t intelliqit/javaapp .'
            }
        }

EXECUTE FOR BUILD JOB
=========================================================================================================

After the image is successfully create we need to push that image into Docker registry

 stage("push Docker image")
        {
            steps
            {
                sh 'sudo docker push intelliqit/javaapp'
            }
        }

EXECUTE THE BUILD JOB

If it is successfully completed the connet CONTROLLER via gitbash
==========================================================================================================

in git bash

ansible all -a "ls -la"
here to check how many local machines are running

vim playbook1.yml

---
- name: install docker and required s/w's
  hosts: all
  tasks: 
    - name: install python pip
      apt: 
        name: python3-pip
        state: present
        update_cache: yes
    - name: download docker, install docker and also install docker-py
      shell: "{{item}}"
      with_items: 
         - curl -fsSL https://get.docker.com -o get-docker.sh
         - sh get-docker.sh
         - pip3 install docker-py
...

execute this playbook
ansible-playbook playbook1.yml -b

vim deploy_app.yml

---
- name: deploy javaapp as docker container in all QA Servers
  hosts: all
  tasks: 
    - name: create a docker container for javaapp
      docker_container: 
          name: javaapp
          image: intelliqit/javaapp
          ports: 
            - 9090:8080
...

go to jenkins pipeline job

stage ('deploy docker image into QAservers using Ansible')
{
steps
 {
   sh 'ssh ubuntu@<private ip of controller> ansible-palybook deploy_app.yml -b' 
 }
}

EXECUTE THE BUILD JOB
=============================================================================================

If it is successfully executed then 

take public ip of QA server and paste it in browser:9090

here tomcat interface is coming

if u give <public ip>:9090/testapp

it shows username and password interface as a container
-------------------------------------------------------------------------------------------

go to Jenkins Pipeline 

here we download and run the selenium scripts

before that go to console output and copy the path  of "Running on jenkins" which is on top

stage ('download and run selenium scripts')
{
  steps
   {
      git 'https://github.com/intelliqittrainings/FunctionalTesting.git'
      sh 'java -jar <paste the path of Running on jenkins>/testing.jar'
   }
}

EXECUTE THE BUILD JOB
===============================================================================================================================

if it is successfully generated, once again goto controller gitbash and create a "delete.yml" file for docker conatainers delete

vim delete.yml

---
- name: delete docker containers
  hosts: all
  tasks: 
    - name: delete docker containers
      docker_container: 
          name: javaapp
          state: absent
...

Go to Jenkins Pipeline

stage ('delete docker containers fom QA Server')
{
  steps
   {
    sh 'ssh ubuntu@<private ip of controller> ansible-playbook delete.yml -b'
   }
}

EXECUTE THE BUILD JOB

it is successfully created then
=================================================================================================

Connect EKS cluster with Gitbash

kubectl get nodes

vim javaapp-deployment.yml

---
apiVersion: appa/v1
kind: Deployment
metadata: 
  name: javaapp-deployment
  labels: 
    type: javaapp
spec: 
  replicas: 2
  selector: 
    matchLabels: 
        type: javaapp
  templates: 
     metadata: 
        name: javaapp-pod
        labels: 
           type: javaapp
     spec: 
       containers: 
         - name: javaapp
           image: intelliqit/javaapp
---
apiVersion: v1
kind: Service
metadata: 
  name: javaapp-service
  labels: 
    type: javaapp
spec: 
   type: LoadBalancer
   selector: 
     type: javaapp
   ports: 
     - targetPort: 8080
       port: 8080
       nodeport: 30008
...

Goto pipeline Project

stage ('deploy into kubernetes cluster')
{
  steps
  {
    sh 'ssh ubuntu@<private ip of EKS cluster> kubectl apply -f javaapp-deployment.yml'
  }
}      

EXECUTE THE BUILD JOB
------------------------------------------------------------------------------------------------------------------
if it successfully completed we can check this to goto EKS gitbash

kubectl get all

here we found the deployment job in running state.

=====================================================================================================================

this is the total entire project details and flow to run CI-CD process.
devops project
