# Assessment 1 -Twoge
This task involves demonstrating the utilization of AWS by a DevOps professional to deploy a Flask-based social networking site named Twoge on an EC2 instance.
# Delpopment Tools
- **AWS**
    - **AWS EC2**: Houses the Twoge application servers.
    - **AWS S3**: Acts as a repository for static assets, such as images and videos.
    - **AWS IAM**: Controls and organizes permissions and access to AWS resources.
    - **AWS VPC**: Establishes a secure and isolated network environment for Twoge.
    - **AWS ALB**: Distributes incoming traffic evenly among EC2 instances.
    - **AWS ASG**: Orchestrates the scaling of EC2 instances based on demand.
    - **AWS SNS**: Sends notifications related to the application's health.
    - **AWS RDS**: Optionally serves as the host for the database in a more secure configuration.
- **Dbeaver for Database**
- **Terminal for Linux**
# Step-by-Step List
- [Create VPC with Two Public Subnets](#create-vpc-with-two-public-subnets)
- [IAM Role for S3 Access](#iam-role-for-s3-access)
- [Launch an EC2 Instance](#launch-an-ec2-instance)
- [Create an S3 Bucket for Static Files](#create-an-s3-bucket-for-static-files)
- [Deploy RDS](#deploy-rds)
- [Deploy Twoge to EC2](#deploy-twoge-to-ec2)
- [Create Application Load Balancer (ALB)](#create-application-load-balancer-alb)
- [Create an Auto Scaling Group (ASG)](#create-an-auto-scaling-group-asg)
- [Creating a Dynamic Scaling Policy for your Auto Scaling Group (ASG)](#creating-a-dynamic-scaling-policy-for-your-auto-scaling-group-asg)
- [Run the Instance Stress Python Script](#run-the-instance-stress-python-script)
# Create VPC with Two Public Subnets
* Open the Amazon VPC. >> **Create VPC**.
* **VPC settings**: 
    - Resources to create: VPC only
    - IPv4 CIDR block: IPv4 CIDR manual input
    - IPv4 CIDR:```10.0.0.0/24```
    - IPv6 CIDR block: No IPv6 CIDR block
   - The rest is the default.
   - Create VPC
* Enable DNS hostnames and resolution:
   - Navigate to the VPC section and select "Actions."
   - Choose "Edit VPC settings."
   - Activate both "Enable DNS hostnames" and "Enable DNS resolution."
* Create Subnet1
   - Choose VPC ID that you just created.
   - Add a subnet name
   - Select one Availability Zone
   - IPv4 VPC CIDR block: ```10.0.1.0/24```.
* Create Subnet2
   - Choose VPC ID that you just created.
   - Get a Subnet name
   - Select the other Availability Zone
   - IPv4 VPC CIDR block: ```10.0.2.0/24```.
* **Create Internet Gateway and Attach to VPC**
   - Name: twoge-igw-prd.
   - Click "Create internet gateway".
   - Attach to VPC: xin-twoge-vpc-uw2.
   - Click "Attach internet gateway".

* **Create Route Table**
   - Name: twogee-rt-prd.
   - VPC: xin-twoge-vpc-uw2.
   - Go to Routes tab and Click Edit routes.
   - Add route:
     - Destination: 0.0.0.0/0.
     - Target: Internet Gateway - twoge-igw-prd.

* **Subnet Associate with Route Table**
   - Go to Subnet associations.
   - Add twoge-pub-2a and twoge-pub-2a and save it.
# IAM Role for S3 Access:
- Navigate to the IAM dashboard>>"Create role" > AWS service > EC2.
- search for "AmazonS3FullAccess" permission and attach it.
- Provide a name, add a description, and create the role.
# launch an EC2 Instance:
- **Access EC2 Dashboard:** Go to Services > EC2.
- **Launch Instance:** Launch an Amazon Linux 2 AMI instance, specifying a name and choosing a key pair and instance type (e.g., t2.micro).

- **Configuration:** Select the IAM role (ec2-twoge-s3) and configure security groups for HTTP/HTTPS and SSH traffic with Auto-assign IP enabled.

- **Review and Launch:** Review configurations and click "Launch."

# Create a S3 bucket for static files
- S3 >> Buckets >> Create Bucket
- General Configuratoion: Name, Region
- Object Ownership: ACLs disabled
- Unclick Block all public access >> Acknowlege
- Bucket versioning: Enabled
- Create bucket: Keep the rest default
- Go to the bucket you created >> Object >> Upload >> upload the static files you want 
- Permissions >> Create "bucket policy" which will allow Ec2 to have access to object/object version

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::erin-bucket2-test",
                "arn:aws:s3:::erin-bucket2-test/*"
            ]
        }
    ]
}
```
# Delpoy RDS
- Go to RDS >> Database >> Create Database
- Create Database:
    - Choose a database creation method: Standard create  
    - Engine options: PoostgreSQL
    - Engine Version: 15.3-r2
    - Templates: Free tier
- Setting:
    - DB instance identifier: enter a name for your database.
    - Master username & Master Password: Enter a personal username and password.
- Instance configuration: db.t3.micro 
- Storage: default >>  Storage autoscaling: deselect Enable Storage autoscaling
- Connectivity:
    - Don't connect to an EC2 compute resource >> IPv4.
    - Virtual private cloud: same VPC selected for your EC2.
    - DB subnet group as default.
    - Public Access: Yes.
    - VPC security group: create a new one.
    - Additional Configurations:
        - Database port: 5432
        -  Database authentication: Password authentication
- Additional Configuration
    - Initial database name
- Create Datebase(5-10min)
# Deploy Twoge to EC2
- SSH into EC2:
    ```
    ssh ec2-user@<Your_EC2_IP> 
    ``` 
- Deploy Twoge Flask application to EC2 instance:
    ```
    # Update system packages
    sudo yum update -y

    # Install git
    sudo yum install git -y

    # Clone the twoge repository
    git clone https://github.com/chandradeoarya/twoge
    cd twoge

    # Install git and Python dependencies
    sudo yum install git python3-pip -y

    # Create and activate a Python virtual environment
    python3 -m venv venv
    source venv/bin/activate

    # Install Python dependencies from requirements.txt
    pip3 install -r requirements.txt

    # Configure the database connection in .env file
    echo 'SQLALCHEMY_DATABASE_URI = "postgresql://username:password@endpoint/database"' > .env

    # Create and configure a systemd service for Gunicorn
    echo '
    Description=Gunicorn instance to serve twoge

    Wants=network.target
    After=syslog.target network-online.target

    [Service]
    Type=simple
    WorkingDirectory=/home/ec2-user/twoge
    Environment="PATH=/home/ec2-user/twoge/venv/bin"
    ExecStart=/home/ec2-user/twoge/venv/bin/gunicorn app:app -c /home/ec2-user/twoge/gunicorn_config.py
    Restart=always
    RestartSec=10

    [Install]
    WantedBy=multi-user.target' > twoge.service

    # Copy the systemd service file to the appropriate directory
    sudo cp twoge.service /etc/systemd/system/twoge.service

    # Reload systemd to apply changes
    sudo systemctl daemon-reload

    # Enable and start the twoge service
    sudo systemctl enable --now twoge

    # Check the status of the twoge service
    sudo systemctl status twoge
    # Install Nginx version 1 using Amazon Linux Extras
    sudo amazon-linux-extras install nginx1 -y

    # Create directories for Nginx configuration
    cd /etc/nginx
    sudo mkdir sites-available
    sudo mkdir sites-enabled

    # Navigate to the sites-available directory
    cd sites-available

    # Create a Nginx configuration file for the twoge service
    sudo vim twoge.service
    server {
        listen 80;
        server_name *;  # underscore as a catch-all

        location / {
            include proxy_params;
            proxy_pass twoge-app-load-balancer-1298083779.eu-west-2.elb.amazonaws.com;
        }
    }   
    # Create a symbolic link to enable the configuration
    sudo ln -s /etc/nginx/sites-available/twoge.service /etc/nginx/sites-enabled/

    # Reload Nginx to apply changes
    sudo systemctl daemon-reload

    # Restart Nginx to activate the new configuration
    sudo systemctl restart nginx

    # Check Nginx status
    sudo systemctl status nginx

    ```
# Create Application Load Balancer (ALB)
- EC2 Dashboard >> Load Balancer 
- Create load balancer:
    - load balncer type: Application Load Balancer 
    - Name >> Internet-facinng >> IPv4
    - Select you VPC
    - Create a new Security Group
        - Allow HTTP, HTTPS AND TCP with a port range 9876
    - Listeners and routing:  
        - Create a target group >> select instance >> name >> port: 9876 >> Your VPC >> Create Targe group
    - Create Load Balancer
# Create an Auto Scaling Group (ASG)
- EC2 Dashboard >> Load Balancer 
- Create load balancer:
    - Step 1: choose launch template
        - create a launch template >> name >> description >> Launch template contents: My AMIs (Create a AMI image in EC2 instance >> action >> image and templates >> create image >> keep default) >> instance type: t2.micro >> create launch template
    - Step 2: Choose instance launch options
        - VPC >> AZ and Subnet: select both
    - Step 3:Configure advanced options 
        - load balancing: attach the ASG to an existing load balancer 
        - select the one just created.
    - Step 4:Configure group size and scaling policies
        -Group size: Desired: 2, Minimum: 1,Maximum: 3
    - Step 5: Add notifications
        - Add Notification >> create topic >> send a notification to: SNNS topic>> with these recipients: email >> event types: Launch, Terminate
    - Create ASG

# Creating a dynamic scaling policy for your Auto Scaling Group (ASG)
- Navigate to ASG: Auto Scaling >> Auto Scaling Groups
- your Auto Scaling Group (ASG) >> automatic Scaling tab
- Dynamic Scaling Policy: Simple Scaling >>  Name your policy
- Configure CloudWatch Alarm >> Create a new CloudWatch alarm >> EC2 metrics, select your ASG >> choose the “CPUUtilization”
- Set the metric condition (e.g., 50% CPU utilization). Optionally, set up a notification topic for email alerts.
- Create Alarm
- Return to Dynamic Scaling Policy in ASG
    - Refresh the CloudWatch Alarm list.
    - Select the new alarm.
    - Set the scaling action (e.g., add 1 capacity unit) when the alarm triggers.
    - Set the cooldown period (e.g., 60 seconds).
- Create Dynamic Scaling Policy

Now, your ASG will automatically adjust its size in response to the CloudWatch alarm. When CPUUtilization goes above 50%, the policy adds capacity. The cooldown period prevents rapid scaling, ensuring new instances have time to impact the metric.
# Run the Instance Stress Python Script
This step  is to test see if the system can handle high load traffic, run following step by step to test:

```
    ssh -i /path/to/your-key.pem ec2-user@your-ec2-ip-address

    nano stress.py

    # Paste the following Python script into the nano editor:
    #!/usr/bin/env python
    """
    Produces load on all available CPU cores
    """

    from multiprocessing import Pool
    from multiprocessing import cpu_count

    def f(x):
        while True:
            x*x

    if __name__ == '__main__':
        processes = cpu_count()
        print('utilizing %d cores\n' % processes)
        pool = Pool(processes)
        pool.map(f, range(processes))

    chmod +x stress.py
   
```







