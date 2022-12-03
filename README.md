# ACS-Configs
AWS Cloud Solution For 2 Company Websites Using A Reverse Proxy Technology

The sole aim of this project to build the infrastructure in AWS using terraform.

As part of the company's desire for improved security and performance, a decision has been made to use a reverse proxy technology from NGINX to achieve this.

Cost, Security, and Scalability are the major requirements for this project. Hence, implementing the architecture designed below, ensure that infrastructure for both websites, WordPress and Tooling, is resilient to Web Server's failures, can accommodate to increased traffic and, at the same time, has reasonable cost.

### Starting Off Your AWS Cloud Project
There are few requirements that must be met before you begin:

1. Properly configure your AWS account and Organization Unit Watch How To Do This [Here](https://www.youtube.com/watch?v=9PQYCc_20-Q)

- Create an AWS Master account. (Also known as Root Account)
- Within the Root account, create a sub-account and name it DevOps. (You will need another email address to complete this)
- Within the Root account, create an AWS Organization Unit (OU). Name it Dev. (We will launch Dev resources in there)
- Move the DevOps account into the Dev OU.
- Login to the newly created AWS account using the new email address.

2. Create a free domain name for your fictitious company at Freenom domain registrar [here](https://www.freenom.com/).

3. Create a hosted zone in AWS, and map it to your free domain from Freenom. [Watch how to do that here](https://youtu.be/IjcHp94Hq8A)
- create a
**NOTE:** As you proceed with configuration, ensure that all resources are appropriately tagged, for example:

Project: <Give your project a name>
Environment: <dev>
Automated: <No> (If you create a resource using an automation tool, it would be <Yes>)


## Set Up a Virtual Private Network (VPC)
Always make reference to the architectural diagram and ensure that your configuration is aligned with it.

1. Create a VPC call it ACS-VPC with a block of 10.0.0.0/16
- click on actions and edit dns hostNames.
- enable and save changes.
- create and attach internet gateways and attach to the vpc call it ACS-igw. Click on attach to vpc
2. Create subnets as shown in the architecture.
- set even numbers for public subnet and odd numbers for private subnets
Note: Create the subnets in groups of public and private subnets. 2 public subnets and 4 private subnets
- select a vpc and call it ACS-public-subnet1 in us-east-1a and an 1pv4 block of 10.0.0.0/24
- select a vpc and call it ACS-public-subnet2 in us-east-1b and an 1pv4 block of 10.0.2.0/24
- select a vpc and call it ACS-private-subnet1 in us-east-1a and an 1pv4 block of 10.0.1.0/24
- select a vpc and call it ACS-private-subnet2 in us-east-1b and an 1pv4 block of 10.0.3.0/24
- select a vpc and call it ACS-private-subnet3 in us-east-1a and an 1pv4 block of 10.0.5.0/24
- select a vpc and call it ACS-private-subnet4 in us-east-1b and an 1pv4 block of 10.0.7.0/24
**Note:** The load balancer needs at least two availability zones

3. Create a route table and associate it with public subnets
- connect the vpc to the route table and call it ACS-public-rtb
4. Create a route table and associate it with private subnets
- connect the vpc to the route table and call it ACS-private-rtb
5. Create an [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
6. Edit a route in public route table, and associate it with the Internet Gateway. (This is what allows a public subnet to be accessible from the Internet)
- select a table go to actions and select edit routes. destination should be anywhere and target should be the igw created above. This is only for the public route
7. Create 3 [Elastic IPs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- Allocate an elastic ip. give it a tag called name: ACS-NAT
- associate it to a resource in order not to get charged
8. Create a [Nat Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) and assign one of the Elastic IPs (*The other 2 will be used by Bastion hosts*)
- click on NAT-Gateway
- name it ACS-natgateway
- add it to a public subnet
- select an elastic ip
- save
- connect the private rtb to the nat gateway
9. Create a Security Group for:

- Nginx Servers: Access to Nginx should only be allowed from a [Application Load balancer (ALB)](https://aws.amazon.com/elasticloadbalancing/application-load-balancer/).
- Bastion Servers: Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. Hence, you can use your workstation public IP address. To get this information, simply go to your terminal and type `curl www.canhazip.com`
- Application Load Balancer: ALB will be available from the Internet
- Webservers: Access to Webservers should only be allowed from the Nginx servers (internal load balancers). 
- Data Layer: Access to the Data layer, which is comprised of Amazon [Relational Database Service (RDS)](https://aws.amazon.com/rds/) and Amazon [Elastic File System (EFS)](https://aws.amazon.com/efs/) must be carefully designed – only webservers should be able to connect to RDS, while Nginx and Webservers will have access to EFS Mountpoint.

<!-- do not forget to write the configuration of the resources -->

### TLS Certificates From Amazon Certificate Manager (ACM)
You will need TLS certificates to handle secured connectivity to your Application Load Balancers (ALB).

- Navigate to AWS ACM
- Request a public wildcard certificate for the domain name you registered in Freenom e.g *.<domain>
- Use DNS to validate the domain name
- Tag the resource with a name eg ACS-cert

### Setup EFS
Amazon Elastic File System (Amazon EFS) provides a simple, scalable, fully managed elastic Network File System (NFS) for use with AWS Cloud services and on-premises resources. In this project, we will utilize EFS service and mount filesystems on both Nginx and Webservers to store data.

1. Create an EFS filesystem called ACS-filesystem. Click on customize
- give it a tag
2. Create an EFS mount target per AZ in the VPC, associate it with both subnets dedicated for data layer. i.e. private subnet1 and private subnet2
3. Associate the Security groups created earlier for data layer.
4. Create an EFS access point. (Give it a name and leave all other settings as default). (This is where we specify the servers to mount with)
- create two access point for each servers one for the wordpress and one for the tooling so the files do not overwrite each other
- name: wordpress; root directory: /wordpress; userID: 0; groupID: 0; all permissions should be 0 as well; POSIX permissions to apply to the root directory path should be 0755
- tag is with a name of wordpress-ap and tooling-ap


### Setup RDS
**Pre-requisite:** Create a KMS key from Key Management Service (KMS) to be used to encrypt the database instance.

To ensure that your databases are highly available and also have failover support in case one availability zone fails, we will configure a multi-AZ set up of RDS MySQL database instance. In our case, since we are only using 2 AZs, we can only failover to one, but the same concept applies to 3 Availability Zones. We will not consider possible failure of the whole Region, but for this AWS also has a solution – this is a more advanced concept that will be discussed in following projects.

To configure RDS, follow steps below:

1. Create a subnet group and add 2 private subnets (data Layer)
- name it ACS-rds-subnet. Choose a vpc. select the two availability zones. 
- create a database. from the dashboard choose databases. select mysql engine with the latest version. select the free tier
2. Create an RDS Instance for mysql 8.*.*
3. To satisfy our architectural diagram, you will need to select either Dev/Test or Production Sample Template. But to minimize AWS cost, you can select the Do not create a standby instance option under Availability & durability sample template (The production template will enable Multi-AZ deployment)
4. Configure other settings accordingly (For test purposes, most of the default settings are good to go). In the real world, you will need to size the database appropriately. You will need to get some information about the usage. If it is a highly transactional database that grows at 10GB weekly, you must bear that in mind while configuring the initial storage allocation, storage autoscaling, and maximum storage threshold.
5. Configure VPC and security (ensure the database is not available from the Internet)
6. Configure backups and retention
7. Encrypt the database using the KMS key created earlier
8. Enable CloudWatch monitoring and export Error and Slow Query logs (for production, also include Audit)
- initial database name can be test. leave the other settings as default
**Note This service is an expensive one. Ensure to review the monthly cost before creating. (DO NOT LEAVE ANY SERVICE RUNNING FOR LONG)**

### Proceed With Compute Resources
You will need to set up and configure compute resources inside your VPC. The resources related to compute are:

- [EC2 Instances](https://www.amazonaws.cn/en/ec2/instance-types/)
- [Launch Templates](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html)
- [Target Groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)
- [Autoscaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)
- [TLS Certificates](https://en.wikipedia.org/wiki/Transport_Layer_Security)
- [Application Load Balancers (ALB)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)

## Set Up Compute Resources for Nginx
**Provision EC2 Instances for Nginx**
1. Create an EC2 Instance based on RH-8 Amazon Machine Image (AMI) in any 2 Availability Zones (AZ) in any AWS Region (it is recommended to use the Region that is closest to your customers). Use EC2 instance of T2 family (e.g. t2.micro or similar)

2. Ensure that it has the following software installed:

- python
- ntp
- net-tools
- vim
- wget
- telnet
- epel-release
- htop

3. Create an AMI out of the EC2 instance

**Prepare Launch Template For Nginx (One Per Subnet)**
- Make use of the AMI to set up a launch template
- Ensure the Instances are launched into a public subnet
- Assign appropriate security group
 -Configure Userdata to update yum package repository and install nginx
**Configure Target Groups**
- Select Instances as the target type
- Ensure the protocol HTTPS on secure TLS port 443
- Ensure that the health check path is /healthstatus
- Register Nginx Instances as targets
- Ensure that health check passes for the target group
**Configure Autoscaling For Nginx**
- Select the right launch template
- Select the VPC
- Select both public subnets
- Enable Application Load Balancer for the AutoScalingGroup (ASG)
- Select the target group you created before
- Ensure that you have health checks for both EC2 and ALB
- The desired capacity is 2
- Minimum capacity is 2
- Maximum capacity is 4
- Set scale out if CPU utilization reaches 90%
- Ensure there is an SNS topic to send scaling notifications

### Set Up Compute Resources for Bastion
**Provision the EC2 Instances for Bastion**
1. Create an EC2 Instance based on RH-8 Amazon Machine Image (AMI) per each Availability Zone in the same Region and same AZ where you created Nginx server
2. Ensure that it has the following software installed

- python
- ntp
- net-tools
- vim
- wget
- telnet
- epel-release
- htop

3. Associate an Elastic IP with each of the Bastion EC2 Instances
4. Create an AMI out of the EC2 instance
**Prepare Launch Template For Bastion (One per subnet)**
- Make use of the AMI to set up a launch template
- Ensure the Instances are launched into a public subnet
- Assign appropriate security group
- Configure Userdata to update yum package repository and install Ansible and git
**Configure Target Groups**
- Select Instances as the target type
- Ensure the protocol is TCP on port 22
- Register Bastion Instances as targets
- Ensure that health check passes for the target group
**Configure Autoscaling For Bastion**
- Select the right launch template
- Select the VPC
- Select both public subnets
- Enable Application Load Balancer for the AutoScalingGroup (ASG)
- Select the target group you created before
- Ensure that you have health checks for both EC2 and ALB
- The desired capacity is 2
- Minimum capacity is 2
- Maximum capacity is 4
- Set scale out if CPU utilization reaches 90%
- Ensure there is an SNS topic to send scaling notifications

### Set Up Compute Resources for Webservers
**Provision the EC2 Instances for Webservers**
Now, you will need to create 2 separate launch templates for both the WordPress and Tooling websites

1. Create an EC2 Instance RH-8 each for WordPress and Tooling websites per Availability Zone (in the same Region).

2. Ensure that it has the following software installed

- python
- ntp
- net-tools
- vim
- wget
- telnet
- epel-release
- htop
- php

3. Create an AMI out of the EC2 instance
**Prepare Launch Template For Webservers (One per subnet)**
- Make use of the AMI to set up a launch template
- Ensure the Instances are launched into a public subnet
- Assign appropriate security group
- Configure Userdata to update yum package repository and install wordpress (Only required on the WordPress launch template)

## CONFIGURE APPLICATION LOAD BALANCER (ALB)
**Application Load Balancer To Route Traffic To NGINX**

Nginx EC2 Instances will have configurations that accepts incoming traffic only from Load Balancers. No request should go directly to Nginx servers. With this kind of setup, we will benefit from intelligent routing of requests from the ALB to Nginx servers across the 2 Availability Zones. We will also be able to offload SSL/TLS certificates on the ALB instead of Nginx. Therefore, Nginx will be able to perform faster since it will not require extra compute resources to validate certificates for every request.

1. Create an Internet facing ALB
2. Ensure that it listens on HTTPS protocol (TCP port 443)
3. Ensure the ALB is created within the appropriate VPC | AZ | Subnets
4. Choose the Certificate from ACM
5. Select Security Group
6. Select Nginx Instances as the target group
Application Load Balancer To Route Traffic To Web Servers
Since the webservers are configured for auto-scaling, there is going to be a problem if servers get dynamically scalled out or in. Nginx will not know about the new IP addresses, or the ones that get removed. Hence, Nginx will not know where to direct the traffic.

To solve this problem, we must use a load balancer. But this time, it will be an internal load balancer. Not Internet facing since the webservers are within a private subnet, and we do not want direct access to them.

1. Create an Internal ALB
2. Ensure that it listens on HTTPS protocol (TCP port 443)
3. Ensure the ALB is created within the appropriate VPC | AZ | Subnets
4. Choose the Certificate from ACM
5. Select Security Group
6. Select webserver Instances as the target group
7. Ensure that health check passes for the target group

NOTE: This process must be repeated for both WordPress and Tooling websites.

### Configuring DNS with Route53
Earlier in this project you registered a free domain with Freenom and configured a hosted zone in Route53. But that is not all that needs to be done as far as DNS configuration is concerned.

You need to ensure that the main domain for the WordPress website can be reached, and the subdomain for Tooling website can also be reached using a browser.

Create other records such as CNAME, alias and A records.

**NOTE:** You can use either CNAME or alias records to achieve the same thing. But alias record has better functionality because it is a faster to resolve DNS record, and can coexist with other records on that name. Read here to get to know more about the differences.

- Create an alias record for the root domain and direct its traffic to the ALB DNS name.
- Create an alias record for tooling.<yourdomain>.com and direct its traffic to the ALB DNS name.

Congratulations!


You have just created a secured, scalable and cost-effective infrastructure to host 2 enterprise websites using various Cloud services from AWS. At this point, your infrastructure is ready to host real websites’ load. Since it is a pretty expensive infrastructure to keep it up and running, we are going to start using Infrastructure as Code tool Terraform to easily provision and destroy this set up.