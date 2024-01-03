# VPC
resource "aws_vpc" "lms-vpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "lms-vpc"
  }
}

# Public Subnet
resource "aws_subnet" "lms-pub-sn" {
  vpc_id     = aws_vpc.lms-vpc.id
  cidr_block = "10.0.0.0/24"
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = "true"
  tags = {
    Name = "lms-public-subnet"
  }
}

# Private Subnet
resource "aws_subnet" "lms-pvt-sn" {
  vpc_id     = aws_vpc.lms-vpc.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "ap-south-1b"
  map_public_ip_on_launch = "false"
  tags = {
    Name = "lms-private-subnet"
  }
}

# Internet gateway
resource "aws_internet_gateway" "lms-igw" {
  vpc_id = aws_vpc.lms-vpc.id

  tags = {
    Name = "lms_internet_gateway"
  }
}

# Public Route table
resource "aws_route_table" "lms-pub-rt" {
  vpc_id = aws_vpc.lms-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.lms-igw.id
  }

  tags = {
    Name = "ecomm-public-route-table"
  }
}

# Public route table association
resource "aws_route_table_association" "lms-pub-asc" {
  subnet_id      = aws_subnet.lms-pub-sn.id
  route_table_id = aws_route_table.lms-pub-rt.id
}

# Private Route table
resource "aws_route_table" "lms-pvt-rt" {
  vpc_id = aws_vpc.lms-vpc.id

  tags = {
    Name = "lms-private-route-table"
  }
}

# Public route table association
resource "aws_route_table_association" "lms-pvt-asc" {
  subnet_id      = aws_subnet.lms-pvt-sn.id
  route_table_id = aws_route_table.lms-pvt-rt.id
}

# Public NACL
resource "aws_network_acl" "lms-pub-nacl" {
  vpc_id = aws_vpc.lms-vpc.id
  subnet_ids = [aws_subnet.lms-pub-sn.id]

  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 65535
  }

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 65535
  }

  tags = {
    Name = "lms-pub-nacl"
  }
}

# Private NACL
resource "aws_network_acl" "lms-pvt-nacl" {
  vpc_id = aws_vpc.lms-vpc.id
  subnet_ids = [aws_subnet.lms-pvt-sn.id]

  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 65535
  }

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 65535
  }

  tags = {
    Name = "lms-pvt-nacl"
  }
}

# Public Security Group
resource "aws_security_group" "lms-pub-sg" {
  name        = "lms-web"
  description = "Allow SSH & HTTP inbound traffic"
  vpc_id      = aws_vpc.lms-vpc.id

  ingress {
    description      = "ssh from web"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  ingress {
    description      = "HTTP from web"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "lms-pub-firewall"
  }
}

# Private Security Group
resource "aws_security_group" "lms-pvt-sg" {
  name        = "lms-db"
  description = "Allow SSH & MYSQL inbound traffic"
  vpc_id      = aws_vpc.lms-vpc.id

  ingress {
    description      = "ssh from vpc"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["10.0.0.0/16"]
  }

  ingress {
    description      = "MYSQL from vpc"
    from_port        = 3306
    to_port          = 3306
    protocol         = "tcp"
    cidr_blocks      = ["10.0.0.0/16"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "lms-db-firewall"
  }
}

# EC2 Public Instance
resource "aws_instance" "web" {
  ami           = "ami-0a7cf821b91bcccbc"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.lms-pub-sn.id
  vpc_security_group_ids = [aws_security_group.lms-pub-sg.id]
  key_name = "DLkey"
  #user_data = file ("lms.sh")

  tags = {
    Name = "lms-web-server"
  }
}

# EC2 Private Instance
resource "aws_instance" "database" {
  ami           = "ami-0a7cf821b91bcccbc"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.lms-pvt-sn.id
  vpc_security_group_ids = [aws_security_group.lms-pvt-sg.id]
  key_name = "DLkey"
  #user_data = file ("lms.sh")

  tags = {
    Name = "lms-db-server"
  }
}

# EC2 Private Instance
resource "aws_instance" "backend" {
  ami           = "ami-0a7cf821b91bcccbc"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.lms-pvt-sn.id
  vpc_security_group_ids = [aws_security_group.lms-pvt-sg.id]
  key_name = "DLkey"
  #user_data = file ("lms.sh")

  tags = {
    Name = "lms-be-server"
  }
}
