
Search
Write
Sign up

Sign in




How To Create a Load Balancer, Instances, and A Hosted Domain Name In Route 53 Using Terraform
Nwokolo Emmanuel
Nwokolo Emmanuel

·
Follow

17 min read
·
Jan 25, 2023
81


3



GOALS:

How to create and configure a vpc, subnet, route table, etc. with terraform
How to create an Ec2 instance and a load balancer with terraform.
How to create a route 53 A record and hosted domain name with terraform
How to configure your Ec2 instances with Ansible
Hey! welcome back.

in this blog, I am going to be showing you how to create a load balancer with 3 Ec2 instances behind it.

also how to create a vpc and its dependencies.

and lastly how to create a route 53 with a hosted domain name all using terraform.

this blog will teach you the fundamentals of how to “use terraform to create a vpc, subnets, instances, and load balancer” by yourself

but if you want a well-detailed video explanation of a zoom class.

then you should CLICK HERE to reach me. see you soon!

NOTE:

before you start this blog there are some things you must do so you don’t end up getting an error when running your terraform script.

prerequisite:

install terraform
create your IAM user & access keys
store your access key for use.
Don’t worry all these prerequisites has be taught in another blog.

CLICK HERE to get it.

when you are done you can come back here to continue.

but if you already have these things ready then let’s go!!

1. CREATING A VPC WITH ITS DEPENDENCIES ON TERRAFORM
in this section of the blog, we will be creating a vpc and all of its dependencies. all in terraform.

you should create a folder or directory and call it terraform.

in that folder create a file and name it main.tf.


the above part that is circled helps us to create everything we will be doing on aws in a particular region.

in this case, I picked London. you might want to change the region or change your location on your AWS console to London.

in the main.tf file you input this script inside.

provider "aws" {
  region = "eu-west-2"
}
# Create VPC
resource "aws_vpc" "Altschool_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = {
    Name = "Altschool_vpc"
  }
}
this script helps us to only create our vpc.

Internet Gateway

the image below is where we create our internet gateway


# Create Internet Gateway

resource "aws_internet_gateway" "Altschool_internet_gateway" {
  vpc_id = aws_vpc.Altschool_vpc.id
  tags = {
    Name = "Altschool_internet_gateway"
  }
}
you can name it whatever you want to name it. but make sure that the vpc id is being placed in this block.

Route Tables


in the picture above we are creating our public route table for our vpc.

# Create public Route Table
resource "aws_route_table" "Altschool-route-table-public" {
  vpc_id = aws_vpc.Altschool_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.Altschool_internet_gateway.id
  }
  tags = {
    Name = "Altschool-route-table-public"
  }
}
copy the script above and also add it to the main.tf file you created.

Subnet Route Table


this script is going to be used by terraform to create our public route table for the two subnets that we are going to be creating.

# Associate public subnet 1 with public route table
resource "aws_route_table_association" "Altschool-public-subnet1-association" {
  subnet_id      = aws_subnet.Altschool-public-subnet1.id
  route_table_id = aws_route_table.Altschool-route-table-public.id
}
# Associate public subnet 2 with public route table
resource "aws_route_table_association" "Altschool-public-subnet2-association" {
  subnet_id      = aws_subnet.Altschool-public-subnet2.id
  route_table_id = aws_route_table.Altschool-route-table-public.id
}
Public Subnet

okay in this part we are creating our public subnets. somethings might look strange to you if you are not really used to terraform.


for instance, in our subnets, we are adding different availability zone for the two different subnets.

one is in EU-west-2a and the other is in EU-west-2b. they should not be in the same availability zones.

because our load balancer will be using these two subnets just like the way we created it in the:

“How to Configure A Service In A Private Server Using Ansible & Creating A Load Balancer In AWS”

just that this time we are using terraform.

# Create Public Subnet-1
resource "aws_subnet" "Altschool-public-subnet1" {
  vpc_id                  = aws_vpc.Altschool_vpc.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "eu-west-2a"
  tags = {
    Name = "Altschool-public-subnet1"
  }
}
# Create Public Subnet-2
resource "aws_subnet" "Altschool-public-subnet2" {
  vpc_id                  = aws_vpc.Altschool_vpc.id
  cidr_block              = "10.0.2.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "eu-west-2b"
  tags = {
    Name = "Altschool-public-subnet2"
  }
}
Network Acl

in this last part of the vpc section, we will be creating a Network Acl.

it’s basically responsible for allowing or denying specific inbound or outbound traffic at the subnet level.


and as you can see here we added our vpc id and our two subnets id.

resource "aws_network_acl" "Altschool-network_acl" {
  vpc_id     = aws_vpc.Altschool_vpc.id
  subnet_ids = [aws_subnet.Altschool-public-subnet1.id, aws_subnet.Altschool-public-subnet2.id]
  ingress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
  egress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
}
and now this is all we will need to create a vpc using terraform. we now move on to the subnets section.

NOTE: everything in the vpc section must be in the main.tf file.

only if you really understand how to use variables then you can create variables for some items.

this blog is for a total beginner to understand.

2. CREATING SECURITY GROUP
in this section of the blog we are going to create security groups for two things.

1. our load balancer.

2. our Ec2 instances.

let’s start!

Load Balancer Security group

in the load balancer security group we stated the load balancer name and description below.

you can change the name of your load balancer if you want to


the ingress in terraform basically means inbound rules. it’s very similar to when we were creating it in the other blog on aws console.

and the egress means outbound rule. very simple to understand. yea?

# Create a security group for the load balancer
resource "aws_security_group" "Altschool-load_balancer_sg" {
  name        = "Altschool-load-balancer-sg"
  description = "Security group for the load balancer"
  vpc_id      = aws_vpc.Altschool_vpc.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    
  }
  egress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
copy the above code and also put it in the main.tf file.

Ec2 Instance Security Group

now we will need to create our Ec2 instance security group to be able to SSH into the instances.


in the security group, you can change the name and description if you want to.

now look at the inbound rules for HTTP and HTTPS. we are adding our load balancer security group to it.

we also created a rule for our SSH connection.

# Create Security Group to allow port 22, 80 and 443
resource "aws_security_group" "Altschool-security-grp-rule" {
  name        = "allow_ssh_http_https"
  description = "Allow SSH, HTTP and HTTPS inbound traffic for private instances"
  vpc_id      = aws_vpc.Altschool_vpc.id
 ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    security_groups = [aws_security_group.Altschool-load_balancer_sg.id]
  }
 ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    security_groups = [aws_security_group.Altschool-load_balancer_sg.id]
  }
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
   
  }
  tags = {
    Name = "Altschool-security-grp-rule"
  }
}
all this should also be in the main.tf file if you are new to all this and don’t know how to use variables

3. CREATING INSTANCES
the third section is where we create our instances and this is going to be fun just like the way it was thought of in the MASTERMIND.


this is how we are going to create our three instances. if you look. the ami (amazon machine image) that we are using is ubuntu.

the instance type is t2.micro. to stay in the range of our free tier. and also if you look at the security groups we added the one we create for the instance.

now notice that this subnet for instances 1 and 3 are the same but instance 2 is in another subnet.

also if you plan on changing the subnet

just make sure the availability zone and the subnet availability zone you are using for one instance are the same.

# creating instance 1
resource "aws_instance" "Altschool1" {
  ami             = "ami-01b8d743224353ffe"
  instance_type   = "t2.micro"
  key_name        = "root-server2-london"
  security_groups = [aws_security_group.Altschool-security-grp-rule.id]
  subnet_id       = aws_subnet.Altschool-public-subnet1.id
  availability_zone = "eu-west-2a"
  tags = {
    Name   = "Altschool-1"
    source = "terraform"
  }
}
# creating instance 2
 resource "aws_instance" "Altschool2" {
  ami             = "ami-01b8d743224353ffe"
  instance_type   = "t2.micro"
  key_name        = "root-server2-london"
  security_groups = [aws_security_group.Altschool-security-grp-rule.id]
  subnet_id       = aws_subnet.Altschool-public-subnet2.id
  availability_zone = "eu-west-2b"
  tags = {
    Name   = "Altschool-2"
    source = "terraform"
  }
}
# creating instance 3
resource "aws_instance" "Altschool3" {
  ami             = "ami-01b8d743224353ffe"
  instance_type   = "t2.micro"
  key_name        = "root-server2-london"
  security_groups = [aws_security_group.Altschool-security-grp-rule.id]
  subnet_id       = aws_subnet.Altschool-public-subnet1.id
  availability_zone = "eu-west-2a"
  tags = {
    Name   = "Altschool-3"
    source = "terraform"
  }
}
you can also use the count module when creating an instance. like it was taught in the MASTERMIND.

Storing Our Ip Addresses

in this section of creating Ec2 instances. we will need to store the IP addresses in a file and terraform can help us with that.

now you might be wondering why we need to store the IP address using terraform.

well, let’s just say I am lazy and I don’t want to always go to my AWS console before I can see my IP addresses.


now there are two very important things here. first, that is the file name.

when creating a file using the local_file module. the file name must be the exact path you want terraform to create the file. like mine it’s:

/vagrant/terraform_assignment/host-inventory
and terraform will create this file for you in that path.

and second the content. you can see we called it in the file content.

# Create a file to store the IP addresses of the instances
resource "local_file" "Ip_address" {
  filename = "/vagrant/terraform_assignment/host-inventory"
  content  = <<EOT
${aws_instance.Altschool1.public_ip}
${aws_instance.Altschool2.public_ip}
${aws_instance.Altschool3.public_ip}
  EOT
}
4. CREATING AN APPLICATION LOAD BALANCER
we are creating an application load balancer in this section to be able to switch between our instances.


in our load balancer. we made the type application and in our subnets, we added our public subnet 1 and subnet 2

we also added our three instances to the load balancer. if you also noticed our security group is that of the load balancer.

# Create an Application Load Balancer
resource "aws_lb" "Altschool-load-balancer" {
  name               = "Altschool-load-balancer"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.Altschool-load_balancer_sg.id]
  subnets            = [aws_subnet.Altschool-public-subnet1.id, aws_subnet.Altschool-public-subnet2.id]
  #enable_cross_zone_load_balancing = true
  enable_deletion_protection = false
  depends_on                 = [aws_instance.Altschool1, aws_instance.Altschool2, aws_instance.Altschool3]
}
Creating Target Groups

our target group is just plain and simple there is nothing too much here.


we just added our vpc id and changed the target type to the instance with a protocol HTTP. the health check can be left as the default like above.

# Create the target group
resource "aws_lb_target_group" "Altschool-target-group" {
  name     = "Altschool-target-group"
  target_type = "instance"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.Altschool_vpc.id
  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 3
    unhealthy_threshold = 3
  }
}
Listener & Listener Rule

we are going to be adding a listener rule to our load balancer.

A listener is a process that checks for connection requests, using the protocol and port that you configure.

The rules that you define for a listener determine how the load balancer routes request to the targets in one or more target groups.


in the above picture, we set our target group arn to our load balancer target group arn that we created before.

we also made the default action and action type to forward.

# Create the listener
resource "aws_lb_listener" "Altschool-listener" {
  load_balancer_arn = aws_lb.Altschool-load-balancer.arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.Altschool-target-group.arn
  }
}
# Create the listener rule
resource "aws_lb_listener_rule" "Altschool-listener-rule" {
  listener_arn = aws_lb_listener.Altschool-listener.arn
  priority     = 1
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.Altschool-target-group.arn
  }
  condition {
    path_pattern {
      values = ["/"]
    }
  }
}
Attaching The Target Groups To The Load Balancer

in this part, we are going to be attaching the target group to the load balancer.


we are creating 3 attachments for our three instances. but this is not really practical.

there are better ways to do it just like I taught in the MASTERMIND.

# Attach the target group to the load balancer
resource "aws_lb_target_group_attachment" "Altschool-target-group-attachment1" {
  target_group_arn = aws_lb_target_group.Altschool-target-group.arn
  target_id        = aws_instance.Altschool1.id
  port             = 80
}
 
resource "aws_lb_target_group_attachment" "Altschool-target-group-attachment2" {
  target_group_arn = aws_lb_target_group.Altschool-target-group.arn
  target_id        = aws_instance.Altschool2.id
  port             = 80
}
resource "aws_lb_target_group_attachment" "Altschool-target-group-attachment3" {
  target_group_arn = aws_lb_target_group.Altschool-target-group.arn
  target_id        = aws_instance.Altschool3.id
  port             = 80 
  
  }
here we are just adding the load balancer target group arn to the target group arn.

and we are also adding the instances id.

all on port 80.

5. CREATING THE ROUTE 53
in this section, we are going to be writing a terraform script that helps us set up a hosted zone on route 53.

we are doing this because just using our load balancer DNS is not really practical.

now create a new file in that same directory. name it route53.tf


in the file, there are three sections as you can see. first, we are creating a variable where we will store our domain name in the default.

for me, it is nwokolo.live and for you, it should be different. in the type, we set it as a string, and the description is domain name.

second, we are creating a hosted zone where we called our variable in the name section.

and lastly, we are creating an A record with the name starting with terraform-test. and the variable of the domain name listed above follows it.

variable "domain_name" {
  default    = "nwokolo.live"
  type        = string
  description = "Domain name"
}
# get hosted zone details
resource "aws_route53_zone" "hosted_zone" {
  name = var.domain_name
  tags = {
    Environment = "dev"
  }
}
# create a record set in route 53
# terraform aws route 53 record
resource "aws_route53_record" "site_domain" {
  zone_id = aws_route53_zone.hosted_zone.zone_id
  name    = "terraform-test.${var.domain_name}"
  type    = "A"
  alias {
    name                   = aws_lb.Altschool-load-balancer.dns_name
    zone_id                = aws_lb.Altschool-load-balancer.zone_id
    evaluate_target_health = true
  }
}
we also added our zone id from the zone we created above it.

in the alias section, we are adding the name of our load balancer through its DNS name.

and also the zone id.

Outputs

create a last file and name it outputs.tf. in this file we are going to be printing out the output of our load balancer.


where we can easily collect the DNS name to check if everything is working fine after running ansible.

NOTE: when you are done with the terraform script first thing to write is

terraform init
to initialize terraform. simple. and it’s a must.

second thing:

terraform plan
this helps you see the plan of the script.

the last thing:

terraform apply
this is to execute everything in the terraform script.


the run was successful!!


6. THE ANSIBLE PLAYBOOK
in this section we are going to be configuring ansible to do three things:

1 install apache

2 print our IP address

3 change our instances timezone to Lagos/Africa

you can change the time zone to whatever country you are in but as for me, that’s mine.


the ansible file above will help us do everything that is to be done even updating and upgrading the servers.

---
- hosts: all
  become: true
  tasks:
  - name: update and upgrade the servers
    apt:
      update_cache: yes
      upgrade: yes
  - name: install apache2
    tags: apache, apache2, ubuntu
    apt:
      name:
        - apache2
      state: latest 
  - name: set timezone to Africa/Lagos
    tags: time
    timezone: name=Africa/Lagos
  - name: print hostname on server
    tags: printf
    shell: echo "<h1>This is my server name $(hostname -f)</h1>" > /var/www/html/index.html
  - name: restart apache2
    tags: restart
    service:
      name: apache2
      state: restarted
Ansible Config File

in the ansible config file, you will set up where your private key is.


like the one above. the private key file is in the path I specified above.

and the inventory file was the name of the file I created with terraform where all my IP addresses are stored by terraform.

we will be needing a remote user because while running your playbook ansible may throw you an error.

that you should use the remote ubuntu user and not root.

[defaults]
inventory = host-inventory
remote_user = ubuntu
private_key_file = /vagrant/terraform_assignment/root-server2-london.pem

success too!!

ansible-playbook -i host-inventory site.yml
the above command is to run the playbook

Smart Tips

you can also automatically run your ansible playbook in your terraform file using this command:

provisioner "local-exec" {
command = "ansible-playbook -i host-inventory site.yml"
}
just copy this file and put it at the end of the terraform script we wrote above.

NOTE: make sure your .pem file is in the same directory where you are running the ansible-playbook

and ssh into each server before running the ansible playbook. if not it will hang.

7. PUTTING YOUR ROUTE 53 NAME SERVERS IN THE DOMAIN PROVIDER.
in this section which is going to be the last section.

we are going to be putting our name servers that were created in our route 53 hosted zone in the domain provider.

in my case I used name.com. you might have used something else but the method is going to be the same.


now on my route 53 dashboard, you can see that terraform already created a hosted zone for my domain name. so now we click on the hosted zone.


now that you are in your hosted zone you click on your hosted zone name.


we are now in. above you will see three record names. we will be ticking on the first one. the one with the name servers and type NS.


and on it, we will be copying the 4 name servers that route 53 created for us.

we are going to be pasting it into our domain provider.

if you also look at the image above you will see TTL and it says 172800 seconds.

which means that it will take up to two days before your domain name can start working.

but that is not usually the case. In my experience, it should take you about 12 hours before it starts working fine.

also note that route 53 will charge you $0.50 every month for a hosted zone.

Putting Our Name Servers On Our Domain Provider.

we will now head over to name.com like I said before you can use any domain provider of your choice the process is the same


now that we are here I will be clicking on the domain name so that I can access and manage it.


and now that I am in. we can see name servers. we are going to be clicking on manage name servers.

NOTE: if you are using another domain provider you can just navigate through your domain name to find the name servers.


as you can see there is something here. Sometimes by default, the domain provider will have created name servers for you.

but we don’t want to use that we want to use the one that was created by route 53. so we delete the ones that are here.


and now we go back to route 53 and copy the name servers one by one and paste them here.


the mistake most people make when copying these name servers is they forget to remove the dot behind the name servers.

so an error is thrown saying invalid hostname.


now that we are done adding the 4 name servers we need to save the changes.


we are good to go.

from here we just wait for our route 53 to come up.

12 hours later…..

it’s up!!




the A record we set to our domain name is already working. with our three instances’ IP addresses showing on the screen.

and I know you might want to try out the main domain name. so let me bring it to you.


as you can see it’s not working. you are wondering why. well, it’s because we only created a record for the name — terraform-test.nwokolo.live

you can also create a record for it using the terraform script in the route 53 section.

Reminder
if you want a well-detailed video explanation of a zoom class in the Mastermind.

CHECK HERE

Resources
Creating An IAM User With An Access Key And Secret Access Key For Terraform Access
How to Configure a Service Manually In A Private Server & Creating A Load Balancer In AWS
How to Configure a Service In A Private Server Using BASH script & Creating A Load Balancer In AWS
How to Configure A Service In A Private Server Using Ansible & Creating A Load Balancer In AWS
NOTE: if you have any questions or want to add to this blog. you can message me through E-mail.

I reply faster to people that are subscribed to my newsletter!!

Conclusion
If you loved this blog post give it a like, comment, and don’t forget to click on the follow button.

And if you would love to get an update on the two interesting blogs I will be posting this week then you should sign up for my newsletter right here!!

Terraform
AWS
Cloud Computing
Infrastructure As Code
Cloud
81


3


Nwokolo Emmanuel
Written by Nwokolo Emmanuel
244 Followers
I am a Cloud Engineer, I love sharing easy solutions to problems that I found difficult. Interested in Open Source | twitter: twitter.com/CloudTopG

Follow

More from Nwokolo Emmanuel
How To Install and Set Up Laravel, Nginx, and MySQL With Docker Compose on Ubuntu 20.04
Nwokolo Emmanuel
Nwokolo Emmanuel

How To Install and Set Up Laravel, Nginx, and MySQL With Docker Compose on Ubuntu 20.04
Learn the 7 easy steps to install and set up laravel with Docker-compose on ubuntu 20.04
12 min read
·
Feb 10, 2023
223

2

How To Install Prometheus and Grafana On Your Cluster Using Terraform and Helm.
Nwokolo Emmanuel
Nwokolo Emmanuel

How To Install Prometheus and Grafana On Your Cluster Using Terraform and Helm.
Learn the simple steps to installing prometheus and grafana on your cluster using terraform!
7 min read
·
Apr 25, 2023
60

2

Discover The 3 Steps To Creating An IAM User With Access & Secret Access keys For Terraform Scripts
Nwokolo Emmanuel
Nwokolo Emmanuel

Discover The 3 Steps To Creating An IAM User With Access & Secret Access keys For Terraform Scripts
In this blog you will discover proven methods that help you set up an environment that is comfortable for terraform to run command to AWS
7 min read
·
Jan 25, 2023
71

7 Easy Steps to Configure A Private Nginx Server Using Ansible & Creating A Load Balancer In AWS
Nwokolo Emmanuel
Nwokolo Emmanuel

7 Easy Steps to Configure A Private Nginx Server Using Ansible & Creating A Load Balancer In AWS
The only blog you will ever need to understand the configuration of an Nginx server using ansible, bastion host and a load balancer!
16 min read
·
Jan 5, 2023
125

3

See all from Nwokolo Emmanuel
Recommended from Medium
Three flavors of Terraform iteration
Marc Dougherty
Marc Dougherty

Three flavors of Terraform iteration
I was writing a Terraform module to create a Google Cloud Load Balancer with an arbitrary set of GKE services as backends. To achieve this…
4 min read
·
May 22, 2024
3

GitOps on AWS EKS: Building a CI/CD Pipeline with Jenkins & ArgoCD
Divyam Sharma
Divyam Sharma

GitOps on AWS EKS: Building a CI/CD Pipeline with Jenkins & ArgoCD
Deployment of Microservices application with EFK stack on Amazon EKS, leveraging the power of Jenkins, ArgoCD, and Helm.
8 min read
·
Mar 22, 2024
10

Lists



Natural Language Processing
1477 stories
·
989 saves
Auto Scaling with CloudWatch Scaling Alarms using Terraform
Mattiamazzari
Mattiamazzari

Auto Scaling with CloudWatch Scaling Alarms using Terraform
Auto Scaling is an effective way to improve availability and reliability of applications. You can increase the number of EC2 instances when…
12 min read
·
Apr 14, 2024
7

Getting Started with Atlantis on GKE for Terraform Automation with GCP Workload Identity
Wade Xu
Wade Xu

Getting Started with Atlantis on GKE for Terraform Automation with GCP Workload Identity
In this comprehensive guide, we’ll take you on a journey to set up Atlantis on Google Kubernetes Engine (GKE) with a focus on Terraform…
6 min read
·
Jan 2, 2024
2

1

Creating a Terraform Module for S3 Remote Backend with DynamoDB State Locking
Deniz Yilmaz
Deniz Yilmaz

Creating a Terraform Module for S3 Remote Backend with DynamoDB State Locking
Configuring Amazon S3 for State File Storage and DynamoDB for Reliable State Locking
6 min read
·
Jan 29, 2024
7

Terraform S3 Backend Best Practices (revised)
Jason Bornhoft
Jason Bornhoft

Terraform S3 Backend Best Practices (revised)
How to set up a secure Terraform backend using AWS S3 + DynamoDB
6 min read
·
Nov 30, 2023
8

See more recommendations
Help

Status

About

Careers

Press

Blog

Privacy

Terms

Text to speech

Teams
