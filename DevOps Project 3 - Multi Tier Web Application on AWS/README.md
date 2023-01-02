<a name="readme-top"></a>

<!-- [![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url] -->

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="">
    <img src="screenshoot/logo.png" alt="Logo" width="80" height="80">
  </a>

  <h3 align="center">DevOps Project 3</h3>

  <p align="center">
    Lift and Shift Application Workload to AWS - AWS Project
  </p>
</div>

## About The Project

<p align="left">
  <img src="screenshoot/aws-project-architecture.png" width="1000" height="550"/>
</p>

A multi tier web application stack setup on a Laptop/Desktop, then using a lift and shift strategy to move the workload to Amazon Web Service (AWS).

### Built With

<!-- - [![Vagrant][vagrant]][vagrant]
- [![VirtualBox][virtualbox]][virtualbox]
- [![Tomcat][tomcat]][tomcat]
- [![RabbitMQ][rabbitmq]][rabbitmq]
- [![Memcache][memcache]][memcache]
- [![Nginx][nginx]][nginx]
- [![MySQL][mysql]][mysql]
- [![Bash][bash_script]][bash_script] -->

- EC2 Instances - VM for tomcat, rabbitmq, memcache, and mysql.
- Elastic Loadbalancer - Nginx LB Replacement.
- Autoscaling - Automation for VM scaling.
- S3/EFS - Shared storage
- Route 53 - Private DNS Service

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- GETTING STARTED -->

## Getting Started

### Prerequisites

<ul style="text-decoration: none;">
  <li>
    <p> AWS Account </p>
  </li>
  <li>
    <p> AWS CLI </p>
  </li>
    <li>
    <p> JDK8 </p>
  </li>
    <li>
    <p> Domain Name </p>
  </li>
    <li>
    <p> Maven </p>
  </li>
</ul>

#### 🧾 Steps

#### 1 - Provision AWS Resources

- Login to your AWS Account
- Configure AWS Certificate Manager for your domain
- Configure the certificate on your domain (I am using Godaddy)
- Create security groups

- Create AWS Key Pair

  ![](screenshoot/create-key-pair.PNG)

#### 2 - Launch EC2 Instances

i. Launch MySQL Instance with these details

- Name: vprofile-db01
- Tag: Key - project, Value - vprofile
- Os Image: Centos 7
- Instance type: t2micro
- Key pair: vprofile-key-pair
- Security Group: vprofile-backend-SG
- Under advanced details add this in user data section

```
#!/bin/bash
DATABASE_PASS='admin123'
sudo yum update -y
sudo yum install epel-release -y
sudo yum install git zip unzip -y
sudo yum install mariadb-server -y


# starting & enabling mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
cd /tmp/
git clone -b vp-rem https://github.com/devopshydclub/vprofile-repo.git
#restore the dump file for the application
sudo mysqladmin -u root password "$DATABASE_PASS"
sudo mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-repo/src/main/resources/db_backup.sql
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

# Restart mariadb-server
sudo systemctl restart mariadb


#starting the firewall and allowing the mariadb to access from port no. 3306
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
sudo systemctl restart mariadb
```

ii. Launch Memcached Instance with these details

- Name: vprofile-mc01
- Tag: Key - project, Value - vprofile
- Os Image: Centos 7
- Instance type: t2micro
- Key pair: vprofile-key-pair
- Security Group: vprofile-backend-SG
- Under advanced details add this in user data section

```
#!/bin/bash
sudo yum install epel-release -y
sudo yum install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sudo memcached -p 11211 -U 11111 -u memcached -d
```

iii. Launch RabbitMQ Instance with these details

- Name: vprofile-rmq01
- Tag: Key - project, Value - vprofile
- Os Image: Centos 7
- Instance type: t2micro
- Key pair: vprofile-key-pair
- Security Group: vprofile-backend-SG
- Under advanced details add this in user data section

```
#!/bin/bash
sudo yum install epel-release -y
sudo yum update -y
sudo yum install wget -y
cd /tmp/
wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
sudo rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
sudo yum -y install erlang socat
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
sudo yum install rabbitmq-server -y
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl status rabbitmq-server
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo systemctl restart rabbitmq-server
```

iv. Launch Tomcat Instance with these details

- Name: vprofile-app01
- Tag: Key - project, Value - vprofile
- Os Image: Ubuntu 18.04
- Instance type: t2micro
- Key pair: vprofile-key-pair
- Security Group: vprofile-app-sg
- Under advanced details add this in user data section

```
#!/bin/bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-8-jdk -y
sudo apt install tomcat8 tomcat8-admin tomcat8-docs tomcat8-common git -y
```

#### 3 - Configure backend IP in Route53

1. Create hosted zone

#### 4 - Build application artifact

```
git clone https://github.com/devopshydclub/vprofile-project.git
```

```
git checkout aws-LiftAndShift
```

```
cd src/main/resources
```

```
vi application.properties
```

#### 5 - Update the host names as per route 53 configurations

#### 6 - Build application artifact

```
cd ../../../ && mv install
```

#### 7 - Create an s3 bucket and Copy the generated artifact to the s3 bucket.

```
aws s3 mb s3://vprofile-artifact-storage-30122022
```

```
aws s3 cp target/vprofile-v2.war s3://vprofile-artifact-storage-30122022
```

#### 8 - Create an IAM for s3 bucket.

#### 9 - Download artifact from s3

i. connect to the tomcat server

```
ssh -i "your-key.pem" ubuntu@your_instance_public_ip
```

ii. Navigate to tomcat server

```
cd /var/lib/tomcat8/webapps
```

iii Stop tomcat server

```
systemctl stop tomcat8
```

iv Remove default tomcat page

```
rm -rf ROOT
```

v. Install aws cli

```
apt install awscli -y
```

vi Copy artifact from s3

```
 aws s3 cp s3://vprofile-artifact-storage-30122022/vprofile-v2.war /tmp/vprofile-v2.war
```

vii Copy artifact to tomcat server

```
 cp /tmp/vprofile-v2.war /var/lib/tomcat8/webapps/ROOT.war
```

viii Start tomcat

```
systemctl start tomcat8
```

#### 10 - Setup LoadBalancer

- Create target group

```
Intances
Target Grp Name: vprofile-elb-TG
protocol-port: HTTP:8080
healtcheck path : /login

Advanced health check settings
Override: 8080
Healthy threshold: 3
available instance: app01 (Include as pending below)
```

- Create LoadBalancer

```
vprofile-prod-elb
Internet Facing
Select all AZs
SecGrp: vprofile-elb-secGrp
Listeners: HTTP, HTTPS
Select the certificate for HTTPS
```

- Create CNAME record on your hosting provider for LoadBalance

```
Name: vprofileapp
Value: your_elb_arn
```

- Lets check our application using our DNS. We can securely connect to our application!

#### 11 - Configure AutoScaling Group for Application Instances

- We will create an AMI from our App Instance.
  ![](screenshoot/vprofile-ami.png)

- Next we will create a Launch template using the AMI created in above step for our ASG

```
Name: vprofile-app-LT
AMI: vprofile-app-image
InstanceType: t2.micro
IAM Profile: vprofile-artifact-storage-role
SecGrp: vprofile-app-SG
KeyPair: vprofile-prod-key
```

- Our Launch template is ready, now we can create our ASG.

```
Name: vprofile-app-ASG
ELB healthcheck
Add ELB
Min:1
Desired:2
Max:4
Target Tracking-CPU Utilization 50
```

- If we terminate any instances we will see ASG will create a new one using LT that we created.
  ![](screenshoot/asg-provisioning-instance.png)

#### 11 - Clean Up

- Delete all resources
<!-- ACKNOWLEDGMENTS -->

## Acknowledgments

Use this space to list resources you find helpful and would like to give credit to. I've included a few of my favorites to kick things off!

- [Choose an Open Source License](https://choosealicense.com)
- [GitHub Emoji Cheat Sheet](https://www.webpagefx.com/tools/emoji-cheat-sheet)
- [Img Shields](https://shields.io)
- [Rumeysa Dogan](https://github.com/rumeysakdogan)
- [Imran](https://github.com/devopshydclub)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & screenshoot -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[contributors-shield]: https://img.shields.io/github/contributors/othneildrew/Best-README-Template.svg?style=for-the-badge
[contributors-url]: https://github.com/othneildrew/Best-README-Template/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/othneildrew/Best-README-Template.svg?style=for-the-badge
[forks-url]: https://github.com/othneildrew/Best-README-Template/network/members
[stars-shield]: https://img.shields.io/github/stars/othneildrew/Best-README-Template.svg?style=for-the-badge
[stars-url]: https://github.com/othneildrew/Best-README-Template/stargazers
[issues-shield]: https://img.shields.io/github/issues/othneildrew/Best-README-Template.svg?style=for-the-badge
[issues-url]: https://github.com/othneildrew/Best-README-Template/issues
[license-shield]: https://img.shields.io/github/license/othneildrew/Best-README-Template.svg?style=for-the-badge
[license-url]: https://github.com/othneildrew/Best-README-Template/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/in/othneildrew
[product-screenshot]: screenshoot/screenshot.png
[vms-are-running-in-virtualbox]: screenshoot/VMs-are-running-in-VirtualBox.png
[host-files-output]: screenshoot/connected-web01-via-ssh.png
[ping-output]: screenshoot/pinging-app01-from-web01.png
[next.js]: https://img.shields.io/badge/next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white
[next-url]: https://nextjs.org/
[react.js]: https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB
[react-url]: https://reactjs.org/
[vue.js]: https://img.shields.io/badge/Vue.js-35495E?style=for-the-badge&logo=vuedotjs&logoColor=4FC08D
[vue-url]: https://vuejs.org/
[angular.io]: https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white
[angular-url]: https://angular.io/
[svelte.dev]: https://img.shields.io/badge/Svelte-4A4A55?style=for-the-badge&logo=svelte&logoColor=FF3E00
[svelte-url]: https://svelte.dev/
[laravel.com]: https://img.shields.io/badge/Laravel-FF2D20?style=for-the-badge&logo=laravel&logoColor=white
[laravel-url]: https://laravel.com
[bootstrap.com]: https://img.shields.io/badge/Bootstrap-563D7C?style=for-the-badge&logo=bootstrap&logoColor=white
[bootstrap-url]: https://getbootstrap.com
[jquery.com]: https://img.shields.io/badge/jQuery-0769AD?style=for-the-badge&logo=jquery&logoColor=white
[jquery-url]: https://jquery.com
[vagrant]: https://img.shields.io/badge/Vagrant-Vagrant-orange
[virtualbox]: https://img.shields.io/badge/Virtualbox-Virtualbox-orange
[nginx]: https://img.shields.io/badge/Nginx-Nginx-orange
[tomcat]: https://img.shields.io/badge/tomcat-tomcat-orange
[rabbitmq]: https://img.shields.io/badge/rabitmq-rabitmq-orange
[memcache]: https://img.shields.io/badge/memcache-memcache-orange
[mysql]: https://img.shields.io/badge/mysql-mysql-orange
[bash_script]: https://img.shields.io/badge/bash-bash-black

```

```
