# AWS nginx Auto-Scaling Setup

Deploy an auto-scaling nginx server on AWS.

## Table of Contents

1. [VPC & Network Configuration](#1-vpc--network-configuration)
2. [EC2 & nginx Configuration](#2-ec2--nginx-configuration)
3. [Application Load Balancer](#3-application-load-balancer)
4. [Auto Scaling Group using Launch Template](#4-auto-scaling-group-using-launch-template)
5. [Testing](#5-testing)

## 1. VPC & Network Configuration

### A. Create a VPC

1. Open the Amazon VPC console.
2. Navigate to "Your VPCs", then "Create VPC".
3. Name your VPC.
4. Specify a CIDR block, e.g., `10.0.0.0/16`.
5. Click "Create".

### B. Create Subnets

1. Navigate to "Subnets", then "Create subnet".
2. Choose the VPC you created.
3. For public subnet:
   - Name it.
   - Choose an availability zone.
   - Specify a CIDR block, e.g., `10.0.1.0/24`.
4. For private subnet, repeat step 3 with a different CIDR block, e.g., `10.0.2.0/24`.

### C. Internet Gateway & Route Table Configuration

1. Navigate to "Internet Gateways", then "Create internet gateway".
2. Name it and attach to your VPC.
3. Go to "Route Tables" and create a new route table.
4. Associate the public subnet.
5. Edit routes to add `0.0.0.0/0` pointing to the Internet Gateway.

### D. Set up NAT Gateway

1. Navigate to "NAT Gateways" > "Create NAT Gateway".
2. Choose the public subnet and allocate a new Elastic IP.
3. Update or create a new private subnet route table.
4. Edit routes to add `0.0.0.0/0` pointing to the NAT Gateway.

## 2. EC2 & nginx Configuration

### A. Launch Bastion EC2 Instance

1. Open the EC2 dashboard.
2. Click "Launch Instance".
3. Choose the Amazon Linux 2023 AMI.
4. Configure network details with your VPC and public subnet.
5. Add storage, tags, and set up security groups to allow SSH from anywhere (For more security and best practice, choose "My IP").
6. Launch the instance using a key pair (If you didn't choose a key pair, it will make you create one).

### B. Launch Nginx EC2 Instance

1. Repeat steps 1, 2, and 3 from session A
2. Configure network details with your VPC and private subnet
3. Add storage, tags, and set up security group to allow SSH from the Bastion Host security group and HTTP from anywhere (we will change this rule to allow only from the Application Load Balancer security group later one)
4. Launch the instance using the key pair created from the previous session.

### C. Install nginx

1. SSH into the Bastion Host from your terminal
2. From the Bastion Host, SSH into your EC2 instance.
3. Install nginx:

```bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### D. Create an Image

1. In the EC2 console, select your nginx instance.
2. Click "Actions" > "Create Image".
3. Specify a name and description for the image.
4. Click "Create Image".

## 3. Application Load Balancer

1. Go to "Load Balancers" in EC2.
2. Click "Create Load Balancer" > "Application Load Balancer".
3. Follow prompts, use your VPC, and public subnet.
4. Set up listeners and configure security groups to allow port 80 (HTTP).
5. Set target groups to HTTP/80 and register your EC2 instance.
6. Go to the Nginx security group and edit the HTTP rule to only allow from the Application Load Balancer. 

## 4. Auto Scaling Group using Launch Template

### A. Create Launch Template

1. In EC2, navigate to "Launch Templates" > "Create launch template".
2. Use the image of the nginx server you created.
3. Specify instance details.

### B. Create Auto Scaling Group

1. Go to "Auto Scaling Groups" in EC2.
2. Use the launch template you just created.
3. Configure VPC and private subnet.
4. Choose "Attach to existing load balancer" and select the target group of the load balancer
5. Under "Additional settings", check "Enable group metrics collection within Cloudwatch"
6. Choose the desired (1), minimum (1), and maximum capacity (3)
7. Under scalling policies, leave the Scaling policy nae as is, make sure the zmetric type is Average CPU, change the Target value to 50, and instance warmup 300. 
8. Set scaling policies with target tracking to adjust based on "Average CPU Utilization".

## 5. Testing

### A. Testing Access to the Webpage

1. Once your Application Load Balancer is et up and in the 'active' state, navigate to its decription in the EC2 console. Copy the DNS name (URL) provided.
2. Paste the DNS name into the web broswer, you should see the default nginx welcome page

### B. Testing Auto Scaling Group

1. To test the ASG, you'd ideally want to increase the load on your existing EC2 instances to trigger a scale-out event. This can be done using tools like `stress` on Linux:
   - SSH into your Nginx server
   - Install the `stress1 toll
   ```bash
   sudo dnf install -y stress
   ```
   - Run `stress` to simulate CPU load:
   ```bash
   stress --cpu 4 --timeout 300
   ```
2. Go to the CloudWatch console, check the alarms, and you should notice an increase in the CPU utilization.
3. Once the CPU utilization exceeds the target you set (70%), the ASG should launch new instances. You can verify this by going to the EC2 console and checking the number of running instances.
4. Once you've verified that the new instances are launched, stop the `stress` tool (if it has not timed out). Over time as the CPU usage drops, the ASG should terminate the extra instances, returning to the desired count.
5. Once your instances scale, access the ALB URL again to ensure that even with new instances added, you still get a consistant nginx page without interruption. 

---



