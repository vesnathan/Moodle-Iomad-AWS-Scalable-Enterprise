This was effective using Moodle 4.1 in March of 2023
  
Launch EC2 instance
- t2.micro
- Name: moodle-server-1
- select or create keypair in ppk format
- create security group: HTTP, create inbound http rule for all
- create security group: HTTPS, create inbound https rule for all
- create security group: SSH, create inbound ssh rule from my ip
- Add above security groups
- Launch Stack

Open Putty
- host name: ec2-user@xxx.xxx.xxx.xxx (public ip of ec2 instance)
- click ssh-auth-credentials-private key file-browse and select keypair created above
- click open

install PHP 
$ sudo yum install amazon-linux-extras  
$ sudo amazon-linux-extras enable php8.1   
$ sudo yum clean metadata  
$ sudo yum install php-cli php-pdo php-fpm php-json php-mysqlnd  
  
install Apache, Start it, and add as service  
$ sudo yum install httpd  
$ sudo systemctl start httpd  
$ sudo systemctl enable httpd.service  
  
test PHP  
$ cd /var/www/html  
$ sudo nano info.php  
<?php  
phpinfo();  
?>  
- cntrl-X, then y, then enter  
  
Reboot server  
$ sudo systemctl reboot  
  
Check PHP is working  
- go to ec2 public ip in browser /info.php  
  
- Change Permissions on folders (use stricter permissions for production server)  
$ sudo chmod -R 777 /var/www/html  
 
  
Get Moodle or Iomad files to /var/www/html  
$ cd /var/www/html
$ sudo wget https://github.com/iomad/iomad/archive/refs/heads/IOMAD_401_STABLE.zip
$ sudo unzip IOMAD_401_STABLE.zip
$ sudo cp -r iomad-IOMAD_401_STABLE/* /var/www/html/
$ sudo rm -R iomad-IOMAD_401_STABLE

Reopen Putty  
- install dependencies  
$ sudo yum install git  php-gd php-pear php-mbstring php-soap php-intl php-zts php-xml php-devel php-sodium php-opcache amazon-efs-utils
 
  
Create EFS storage  
- name: moodle-efs  
- VPC: default  
- Standard  
- create  
- note the file system id   
    fs-XXXXXXXXXXX  
  
Open AWS VPC  
- create a new Security Group named Moodle EFS Target with no rules  
- create a new Security Group named Moodle EFS Mount with inbound NFS rule with source Moodle NFS Target  
- Add the Moodle EFS Target group to the EC2 instance  
- For each Mount Target in EFS remove Deafult security group and add Moodle EFS Mount  
  
 
  
create moodle data directory  
$ cd /var  
$ sudo mkdir moodledata  
$ sudo chmod -R 777 /var/moodledata 
  
- edit /etc/fstab to mount the efs drive to the moodledata directory on boot  
$ sudo nano /etc/fstab  
fs-XXXXXXXXXXX:/ /var/moodledata efs _netdev,noresvport,tls  
- cntrl-X, then y, then enter  
  
Create Database  
- Go to AWS RDS and Create Database  
- Standard Create  
- Aurora SQL  
- Production  
- username: admin  
- password: XXXXXXXXXXX 
- Instance: db.r5.large  
- Memory optimized classes 
- Create an Aurora Replica  
- Connect to an EC2 compute resource - moodle-server  
  
Install epel  
$ sudo amazon-linux-extras install epel  
$ sudo yum update  
  
Install Redis Server  
$ sudo yum install redis  
$ sudo systemctl start redis  
$ sudo systemctl enable redis  
$ sudo sysctl vm.overcommit_memory=1  
$ sudo nano /etc/sysctl.conf  
vm.overcommit_memory = 1  
  
Install gcc  
$ sudo yum group install "Development Tools"  
  
Install PHP Redis Plugin  
$ sudo pecl install redis  
- answer no, no, no  
  
Add extension=redis.so and xml.so to phpini  
$ sudo nano /etc/php.ini  
extension=redis.so  
extension=xml.so  
  
Also change max_input_vars to 5000  
- cntrl-W  
- enter "max_input_vars"  
- uncomment line and change 1000 to 5000  
- cntrl-X, then y, then enter  
  
Create Elasticache Redis Cluster  
- Cluster mode: disabled  
- Name: moodle-redis  
- Location: cloud  
- Multi-AZ: enable  
- Engine version: 7  
- TLS Prefereed  
- Port: 6379  
- Parameter groups: default.redis7  
- Node Type: cache.t3.medium  
- Number of replicas: 2  
- Network type: ipv4  
- Subnet groups: Existing (Moodle)  
- Availability Zone placements: No Preference  
- Next  
- Encryption in transit: Enable  
- Encryption in transit: disabled   
  
Install Moodle  
- visit ec2 public ip address in browser  
- Data directory = /var/moodledata  
- Next  
- Type = Aurora MySQL  
- Database Host = End point of RDS Writer Instance  
- Database User: admin  
- Database password:  
- Database Port: 3306  
  
Add instance in Moodle->SiteAdmin->Plugins->Caching->Config->Redis Add Instance  
- Store Name: Redis Store  
- Store Primary endpoint  
- bottom of page->Stores used when no mapping is present->edit mappings  
- Application: Redis Store  
- Session: Redis Store  
  
Create EC2 image  
- got to ec2 dashboard and right click instance name->create image  
  
Launch second server  
- Launch instance from AMI  
- Name: moodle-server-2  
- Security Groups same as other instance  
- Test new server is working  
- I had to redo permissions on the server  
  
Create target group for load balancer  
- Click Target Group  
- Choose a target type: Instances  
- Target group name: moodle-target-group  
- create target group  
  
Create Load Balancer  
- click load balancers->create load balancer->Application Load Balancer->create  
- Name: moodle-load-balancer  
- Default Action: moodle-target-group  
- Create  
  
Change DNS to point to load balancer 
  
Create Launch Template  
- Name: moodle-launch-template  
- My AMI's->Owned By Me->Moodle EC2 Image  
- Instance t2.small  
- Create Launch Template  
  
Create Auto-Scaling-Group  
- Name: Moodle Auto-Scaling Group  
- Launch template: moodle-launch-template  
- Next  
- Availability Zones: all
- Next
- Attach to an existing load balancer
- Choose from your load balancer target groups
- moodle-target-group
- Next
- Maximum Capacity: 2
- Next
- Next
- Next

Delete moodle-server-2























