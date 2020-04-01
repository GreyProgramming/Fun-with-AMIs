# Fun-with-AMIs
Using GCP to provision an AWS AMI, and using terraform on that to create more, to demonstrate how to use terraform to create a swarm of machines.

## Setup an AWS VPC, Subnet, Gateway, Route table and EC2 instance

vpc_id=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)

echo ${vpc_id}

subnet_id=$(aws ec2 create-subnet --vpc-id {vpc_id} --cidr-block 10.0.1.0/24)

echo ${subnet_id}

gateway_id=$(aws ec2 create-internet-gateway)

echo ${gateway_id}

aws ec2 attach-internet-gateway --internet-gateway-id {gateway_id} --vpc-id {vpc_id}

route_id=$(aws ec2 create-route-table --vpc-id {vpc_id})

echo ${route_id}

aws ec2 create-route --route-table-id {route_id} --destination-cidr-block 0.0.0.0/0 --gateway-id {gateway_id}

security_id=$(aws ec2 create-security-group --group-name security --description "custom security group" --vpc-id {vpc_id})

ip_id=$($ curl https://checkip.amazonaws.com)

aws ec2 authorize-security-group-ingress --group-id sg-903004f8 --protocol tcp --port 22 --cidr $(ip_id)/32

aws ec2 create-key-pair --key-name AccessKey --query ‘KeyMaterial’ --output text > ~/.ssh/AccessKey.pem

chmod 400 ~/.ssh/AccessKey.pem

ip_address=$(aws ec2 run-instances --image-id ami-0ee246e709782b1be --count 1 --instance-type t2.micro --key-name AccessKey --subnet-id {subnet_id})

echo ${ip_address}

ssh -I ~/.ssh/AccessKey.pem ubuntu@{ip_address}


## Download terraform on that EC2 instance, unzip and remove download file, make infra and move in

wget https://releases.hashicorp.com/terraform/0.11.3/terraform_0.11.3_linux_amd64.zip

sudo unzip terraform_0.11.3_linux_amd64.zip -d /usr/local/bin/

rm terraform_0.11.3_linux_amd64.zip

mkdir infra && cd infra



## Setup the Main, Variables, outputs, etc 



cat > main.tf <<EOF
provider "aws" {
version = "< 2"
region  = "eu-west-2" # London
}



resource "aws_vpc" "web_vpc" {

count = "${var.instance_count}"
  cidr_block  = "192.168.100.0/24"
  enable_dns_hostnames = true

  tags {
    Name = "Web VPC ${count.index + 1}""
  }
}

resource "aws_subnet" "web_subnet" {
  ### Use the count meta-parameter to create multiple copies
  count = "${var.instance_count}"
  vpc_id            = "${aws_vpc.web_vpc.id}"
  ### cidrsubnet function splits a cidr block into subnets
  cidr_block        = "${cidrsubnet(var.network_cidr, 1, count.index)}"
  ### element retrieves a list element at a given index
  availability_zone = "${element(var.availability_zones, count.index)}"

  tags {
    Name = "Web Subnet ${count.index + 1}"
  }
}

resource "aws_instance" "web" {
  count         = "${var.instance_count}"
  ### lookup returns a map value for a given key
  ami           = "${lookup(var.ami_ids, " eu-west-2")}"
  instance_type = "t2.micro"
  ### Use the subnet ids as an array and evenly distribute instances
  subnet_id     = "${element(aws_subnet.web_subnet.*.id, count.index % length(aws_subnet.web_subnet.*.id))}"
  
  ### Use instance user_data to serve the custom website
  user_data     = "${file("user_data.sh")}"
  
  ### Attach the web server security group
  vpc_security_group_ids = ["${aws_security_group.web_sg.id}"]

  tags {
    Name = "Web Server ${count.index + 1}"
  }
}

EOF

cat > variables.tf <<'EOF'

variable network_cidr {
  default = "192.168.100.0/24"
}

variable availability_zones {
  default = ["eu-west-2a ", " eu-west-2b"]
}

variable instance_count {
  default = 2
}

variable ami_ids {
  default = {
    " eu-west-2a" = "ami-0fb83677"
    " eu-west-2b" = "ami-97785bed"
  }
}

EOF

cat >> outputs.tf <<'EOF'
output "site_address" {
  value = "${aws_elb.web.dns_name}"
}

output "ips" {
  ### join all the instance private IPs with commas separating them
  value = "${join(", ", aws_instance.web.*.private_ip)}"
}
EOF

cat >> user_data.sh <<'EOF'
#!/bin/bash
cat > /var/www/html/index.php <<'END'
<?php
$instance_id = file_get_contents("http://instance-data/latest/meta-data/instance-id");
echo "You've reached instance ", $instance_id, "\n";
?>
END
EOF

cat > networking.tf <<'EOF'
### Internet gateway to reach the internet
resource "aws_internet_gateway" "web_igw" {
  vpc_id = "${aws_vpc.web_vpc.id}"
}

### Route table with a route to the internet
resource "aws_route_table" "public_rt" {
  vpc_id = "${aws_vpc.web_vpc.id}"
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_internet_gateway.web_igw.id}"
  }

  tags {
    Name = "Public Subnet Route Table"
  }
}

### Subnets with routes to the internet
resource "aws_subnet" "public_subnet" {
  ### Use the count meta-parameter to create multiple copies
  count             = 2
  vpc_id            = "${aws_vpc.web_vpc.id}"
  cidr_block        = "${cidrsubnet(var.network_cidr, 2, count.index + 2)}"
  availability_zone = "${element(var.availability_zones, count.index)}"

  tags {
    Name = "Public Subnet ${count.index + 1}"
  }
}

### Associate public route table with the public subnets
resource "aws_route_table_association" "public_subnet_rta" {
  count          = 2
  subnet_id      = "${aws_subnet.public_subnet.*.id[count.index]}"
  route_table_id = "${aws_route_table.public_rt.id}"
}

EOF

cat > security.tf <<'EOF'
resource "aws_security_group" "elb_sg" {
  name        = "ELB Security Group"
  description = "Allow incoming HTTP traffic from the internet"
  vpc_id      = "${aws_vpc.web_vpc.id}"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ### Allow all outbound traffic
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "web_sg" {
  name        = "Web Server Security Group"
  description = "Allow HTTP traffic from ELB security group"
  vpc_id      = "${aws_vpc.web_vpc.id}"

  ### HTTP access from the VPC
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = ["${aws_security_group.elb_sg.id}"]
  }

  ### Allow all outbound traffic
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

EOF

cat > load_balancer.tf <<'EOF'
resource "aws_elb" "web" {
  name = "web-elb"
  subnets = ["${aws_subnet.public_subnet.*.id}"]
  security_groups = ["${aws_security_group.elb_sg.id}"]
  instances = ["${aws_instance.web.*.id}"]

  ### Listen for HTTP requests and distribute them to the instances
  listener { 
    instance_port     = 80
    instance_protocol = "http"
    lb_port           = 80
    lb_protocol       = "http"
  }

  ### Check instance health every 10 seconds
  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 3
    target = "HTTP:80/"
    interval = 10
  }
}

EOF

## Tell it to go

terraform init
terraform apply
terraform output ips
site_address=$(terraform output site_address)
watch curl -s $site_address
