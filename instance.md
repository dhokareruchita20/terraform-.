## ec2 instance
```
provider "aws" {
  region = "us-east-1"
}
resource "aws_instance" "R1" {
  ami           = "ami-091138d0f0d41ff90"
  instance_type = "t3.micro"
  tags = {
    Name = "MYINSTANCE"
  }
}
```
