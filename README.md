# On-Premises Migration to AWS

This repository documents the steps taken to set up an AWS environment with EC2, RDS, and VPCs/Subnets. The project includes creating a VPC with subnets, launching an EC2 instance, creating an RDS database, configuring security groups, and deploying a simple web application.

## Table of Contents

1. [Creating VPC and Subnets](#creating-vpc-and-subnets)
2. [Creating an EC2 Instance](#creating-ec2-instance)
3. [Creating an RDS Database](#creating-rds-database)
4. [Connecting to the EC2 Instance](#connecting-to-ec2-instance)
5. [Using MySQL Application in EC2 Instance](#using-mysql-application-in-ec2-instance)
6. [Conclusion](#conclusion)

![Image 1](https://i.imgur.com/OAssGhy.png)
![Image 2](https://i.imgur.com/k4kF8SE.png)

## Creating VPC and Subnets

### GUI Steps

1. **Create VPC:**
   - Name: `vpc-production`
   - CIDR Block: `10.0.0.0/16`

2. **Create Subnets:**
   - `vpc-production-pu-1` (Public) in `us-east-1a`, `10.0.0.0/24`
   - `vpc-production-pv-1` (Private) in `us-east-1a`, `10.0.1.0/24`
   - `vpc-production-pv-2` (Private) in `us-east-1b`, `10.0.2.0/24`

3. **Create Internet Gateway:**
   - Name: `igw-prod`
   - Attach to `vpc-production`

4. **Update Route Table:**
   - Edit routes to add `0.0.0.0/0` with `igw-prod`

### VSCode Steps

```python
import boto3

# Create VPC
ec2 = boto3.client('ec2')
vpc = ec2.create_vpc(CidrBlock='10.0.0.0/16')
vpc_id = vpc['Vpc']['VpcId']

# Create Subnets
public_subnet = ec2.create_subnet(CidrBlock='10.0.0.0/24', VpcId=vpc_id, AvailabilityZone='us-east-1a')
private_subnet1 = ec2.create_subnet(CidrBlock='10.0.1.0/24', VpcId=vpc_id, AvailabilityZone='us-east-1a')
private_subnet2 = ec2.create_subnet(CidrBlock='10.0.2.0/24', VpcId=vpc_id, AvailabilityZone='us-east-1b')

# Create Internet Gateway and attach to VPC
igw = ec2.create_internet_gateway()
igw_id = igw['InternetGateway']['InternetGatewayId']
ec2.attach_internet_gateway(InternetGatewayId=igw_id, VpcId=vpc_id)

# Create Route Table and add route
route_table = ec2.create_route_table(VpcId=vpc_id)
route_table_id = route_table['RouteTable']['RouteTableId']
ec2.create_route(RouteTableId=route_table_id, DestinationCidrBlock='0.0.0.0/0', GatewayId=igw_id)
```

# Creating EC2 Instance

## Instance Details:
- **Name:** awsuse1app01
- **AMI:** Ubuntu 22.04
- **Instance Type:** t2.micro
- **Key Pair:** Use/Create key pair

## Network Settings:
- **VPC:** vpc-production
- **Subnet:** vpc-production-pu-1
- **Auto-assign Public IP:** Enable

## Security Group:
- **Name:** app-sg01
- **Description:** Allow SSH and HTTP

### Inbound Rules:
- **SSH (Port 22)**
- **Custom TCP (Port 8080)**

## VSCode Steps
```python
ec2 = boto3.resource('ec2')

# Create Security Group
sec_group = ec2.create_security_group(GroupName='app-sg01', Description='Allow SSH and HTTP', VpcId=vpc_id)
sec_group.authorize_ingress(IpPermissions=[
    {'IpProtocol': 'tcp', 'FromPort': 22, 'ToPort': 22, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
    {'IpProtocol': 'tcp', 'FromPort': 8080, 'ToPort': 8080, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]}
])

# Launch EC2 Instance
instance = ec2.create_instances(
    ImageId='ami-12345678', # Replace with actual Ubuntu 22.04 AMI ID
    InstanceType='t2.micro',
    KeyName='your-key-pair',
    NetworkInterfaces=[{
        'SubnetId': public_subnet['Subnet']['SubnetId'],
        'DeviceIndex': 0,
        'AssociatePublicIpAddress': True,
        'Groups': [sec_group.group_id]
    }],
    MinCount=1,
    MaxCount=1
)
```

# Creating RDS Database

## Instance Details:
- Standard Create, MySQL 5.7.44, Free Tier
- **DB Instance Identifier:** awsuse1db01
- **Master Username:** admin
- **Master Password:** admin123456
- **DB Instance Class:** db.t2.micro
- **Storage:** 40GB SSD

## Connectivity:
- **VPC:** vpc-production
- **Public Access:** No
- **Subnet Group:** Default
- **Security Group:** sec-group-db-01
- **Availability Zone:** us-east-1a
- **Database Port:** 3306

## VSCode Steps
```python
rds = boto3.client('rds')

# Create Security Group for RDS
db_sec_group = ec2.create_security_group(GroupName='sec-group-db-01', Description='RDS Security Group', VpcId=vpc_id)

# Create RDS Instance
response = rds.create_db_instance(
    DBInstanceIdentifier='awsuse1db01',
    AllocatedStorage=40,
    DBInstanceClass='db.t2.micro',
    Engine='MySQL',
    MasterUsername='admin',
    MasterUserPassword='admin123456',
    VpcSecurityGroupIds=[db_sec_group.group_id],
    AvailabilityZone='us-east-1a',
    DBSubnetGroupName='default',
    Port=3306
)
```

# Connecting to EC2 Instance

## SSH into EC2 Instance:
```sh
ssh ubuntu@public-ip -i ssh-keypair.pem
Run Preliminary Commands:
sh
Copy code
sudo sed -i "/#\$nrconf{restart} = 'i';/s/.*/\$nrconf{restart} = 'a';/" /etc/needrestart/needrestart.conf
cat /etc/needrestart/needrestart.conf | grep -i nrconf{restart}
sudo apt update
sudo apt install python3-dev -y
sudo apt install python3-pip -y
sudo apt install build-essential libssl-dev libffi-dev -y
sudo apt install libmysqlclient-dev -y
sudo apt install unzip -y
sudo apt install libpq-dev libxml2-dev libxslt1-dev libldap2-dev -y
sudo apt install libsasl2-dev libffi-dev -y
pip install Flask==2.3.3
export PATH=$PATH:/home/ubuntu/.local/bin/
pip3 install wtforms
sudo apt install pkg-config
pip3 install flask_mysqldb
pip3 install passlib
sudo apt-get install mysql-client -y
```


## Using MySQL Application in EC2 Instance

1. **Download and Unzip Application:**
   ```sh
   wget https://tcb-bootcamps.s3.amazonaws.com/bootcamp-aws/en/wikiapp-en.zip
   wget https://tcb-bootcamps.s3.amazonaws.com/bootcamp-aws/en/module3/dump-en.sql
   unzip wikiapp-en.zip
   ```

2. **Setup MySQL Database:**
   ```sh
   mysql -h <rds_endpoint> -P 3306 -u admin -p
   Password: admin123456
   create database wikidb;
   source dump-en.sql;
   ```

3. **Create MySQL User:**
   ```sql
   CREATE USER wiki@'%' IDENTIFIED BY 'admin123456';
   GRANT ALL PRIVILEGES ON wikidb.* TO wiki@'%';
   ALTER USER 'wiki'@'%' IDENTIFIED BY 'wiki123456';
   FLUSH PRIVILEGES;
   EXIT;
   ```

4. **Edit `wiki.py` File:**
   - Change `MYSQL_HOST` to `<rds_endpoint>`

5. **Run Application:**
   ```sh
   cd wikiapp
   python3 wiki.py
   ```

6. **Access Application:**
   - Go to `http://public-ec2:8080`
   - Login: `admin/admin`
   - Add a new article

## Conclusion

This project demonstrated how to set up an AWS environment with VPC, EC2, and RDS. We created a custom VPC with subnets, launched an EC2 instance, set up an RDS MySQL database, configured security groups for secure access, and deployed a web application. The hands-on experience gained through this project reinforces key AWS concepts and skills, including network configuration, instance management, and database connectivity.
