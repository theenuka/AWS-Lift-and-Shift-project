# AWS Lift and Shift: Multi-Tier Web Application Deployment

## Project Overview
This project documents a lift-and-shift migration of the VProfile multi-tier web application to AWS. The application stack is moved to EC2 with minimal code changes while adding managed AWS services for availability, security, and scalability.

VProfile is a Java-based social networking platform with four core services:
- Tomcat 10 application server (WAR deployment)
- MySQL database (persistent data)
- Memcached (database query cache)
- RabbitMQ (asynchronous messaging)

Each service runs on its own EC2 t3.micro instance. AWS managed services are layered on top: Application Load Balancer, Auto Scaling Group, S3 artifact storage, Route 53 private DNS, and ACM for TLS. The public entry point is https://vprofileapp.theenuka.xyz.

## Architecture and Technology Stack
The architecture follows a three-tier design:
- Presentation tier: Application Load Balancer (public HTTPS)
- Application tier: Tomcat EC2 instance (project-app01)
- Data tier: MySQL (project-db01), Memcached (project-mc01), RabbitMQ (project-rmq01)

All internal service communication uses Route 53 private DNS (vprofile.in) to avoid hard-coded IPs.

| Component | Details |
| --- | --- |
| Application server | Apache Tomcat 10 on EC2 t3.micro - project-app01, us-east-2c |
| Database | MySQL on EC2 t3.micro - project-db01, port 3306, us-east-2c |
| Cache | Memcached on EC2 t3.micro - project-mc01, port 11211, us-east-2c |
| Message broker | RabbitMQ on EC2 t3.micro - project-rmq01, port 5672, us-east-2c |
| Load balancer | Application Load Balancer - vprofile-theenuka-elb (internet-facing, IPv4), 3 AZs |
| Target group | vprofile-theenuka-tg - HTTP, port 8080 |
| Auto Scaling Group | vprofile-theenuka-app-asg - min 1, desired 1, max 4, 3 AZs |
| Artifact store | S3 bucket - vprofile-theenu-artifacts - vprofile-v2.war |
| Private DNS | Route 53 private hosted zone - vprofile.in - 4 A records |
| SSL certificate | ACM wildcard certificate - *.theenuka.xyz |
| Public domain | GoDaddy - theenuka.xyz - CNAME vprofileapp -> ELB DNS |
| SSH key pair | project-prod-key (RSA) |
| VPC | vpc-default |
| AWS region | US East (Ohio) - us-east-2 |

![Figure 1 - Full project architecture diagram](docs/images/image.png)

## Flow of Execution (12 Steps)
Each step below includes the related screenshot from docs/images.

### Step 1: Log in to AWS and set the region
Confirmed the console region as US East (Ohio), us-east-2.

![Figure 2 - AWS Console home showing us-east-2](docs/images/image%20copy.png)

### Step 2: Create a key pair
Created project-prod-key (RSA) for SSH access across all EC2 instances.

![Figure 3 - EC2 Key Pairs: project-prod-key](docs/images/image%20copy%202.png)

### Step 3: Create security groups
Defined layered security groups for ELB, app tier, and backend tier with strict ingress rules.

![Figure 4 - Security groups list](docs/images/image%20copy%203.png)
![Figure 5 - ELB SG inbound rules](docs/images/image%20copy%204.png)
![Figure 6 - App SG inbound rules](docs/images/image%20copy%205.png)
![Figure 7 - Backend SG inbound rules](docs/images/image%20copy%206.png)
![Figure 8 - Default VPC SG](docs/images/image%20copy%207.png)

### Step 4: Launch EC2 instances with user data
Provisioned four t3.micro instances with user data scripts for automated setup.

![Figure 9 - EC2 instances running](docs/images/image%20copy%208.png)

### Step 5: Configure Route 53 private DNS
Created private hosted zone vprofile.in with A records for app, db, cache, and broker.

![Figure 10 - Route 53 private hosted zone](docs/images/image%20copy%209.png)

### Step 6: Build the application from source
Ran mvn install to compile and package the application as vprofile-v2.war.

![Figure 11 - Maven build phases](docs/images/image%20copy%2010.png)
![Figure 12 - Maven build success](docs/images/image%20copy%2011.png)

### Step 7: Upload the artifact to S3
Uploaded vprofile-v2.war to s3://vprofile-theenu-artifacts.

![Figure 13 - CLI upload and verification](docs/images/image%20copy%2012.png)
![Figure 14 - S3 bucket with artifact](docs/images/image%20copy%2013.png)

### Step 8: Deploy the artifact to Tomcat
Replaced ROOT app on project-app01 and restarted Tomcat 10.

![Figure 15 - Tomcat deployment steps](docs/images/image%20copy%2014.png)

### Step 9: Configure ALB with HTTPS
Issued ACM wildcard certificate, created ALB and target group, and set HTTP/HTTPS listeners.

![Figure 16 - ACM certificate](docs/images/image%20copy%2015.png)
![Figure 17 - Load balancer list](docs/images/image%20copy%2016.png)
![Figure 18 - ELB details and listeners](docs/images/image%20copy%2017.png)
![Figure 19 - Target group health](docs/images/image%20copy%2018.png)

### Step 10: Point domain to the load balancer
Added GoDaddy CNAME vprofileapp -> ELB endpoint and validated DNS records.

![Figure 20 - GoDaddy DNS records](docs/images/image%20copy%2019.png)
![Figure 21 - CNAME records](docs/images/image%20copy%2020.png)

### Step 11: Verify the full deployment
Verified HTTPS, certificate, login, RabbitMQ status, MySQL user list, and Memcached caching.

![Figure 22 - Login page over HTTPS](docs/images/image%20copy%2021.png)
![Figure 23 - Browser security panel](docs/images/image%20copy%2022.png)
![Figure 24 - Certificate viewer](docs/images/image%20copy%2023.png)
![Figure 25 - Application dashboard](docs/images/image%20copy%2024.png)
![Figure 26 - RabbitMQ status page](docs/images/image%20copy%2025.png)
![Figure 27 - Users list page](docs/images/image%20copy%2026.png)
![Figure 28 - First request: DB -> cache](docs/images/image%20copy%2027.png)
![Figure 29 - Second request: cache hit](docs/images/image%20copy%2028.png)

### Step 12: Build an Auto Scaling Group
Created AMI, launch template, and ASG for the application tier.

![Figure 30 - AMI created](docs/images/image%20copy%2029.png)
![Figure 31 - Launch template](docs/images/image%20copy%2030.png)
![Figure 32 - Auto Scaling Group](docs/images/image%20copy%2031.png)
![Figure 33 - ASG details](docs/images/image%20copy%2032.png)

## Technical Skills Demonstrated
- AWS infrastructure provisioning across EC2, S3, Route 53, ALB, ACM, and ASG
- Linux server automation with user data and service management with systemctl
- Layered security group design for tiered access control
- Java build pipeline with Maven and WAR artifact management
- DNS architecture for private service discovery and public domain routing
- HTTPS and certificate management with ACM
- End-to-end validation of app, cache, database, and message broker integrations

## Conclusion
This project delivers a complete lift-and-shift migration of a Java multi-tier application to AWS. The application is securely exposed over HTTPS through an ALB across three availability zones, backed by EC2 instances for Tomcat, MySQL, Memcached, and RabbitMQ. Private DNS via Route 53 simplifies internal communication, and the app tier is prepared for scale-out using a custom AMI, launch template, and Auto Scaling Group. The deployment was fully verified, including cache behavior, database reads, and RabbitMQ connectivity.
