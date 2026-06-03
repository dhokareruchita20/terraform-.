## RDS
```
provider "aws" {
  region = "us-east-1"
}

resource "aws_db_instance" "mydb" {
  allocated_storage    = 20
  db_name              = "mydatabase"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  username             = "admin"
  password             = "redhat123"
  publicly_accessible  = true
  skip_final_snapshot  = true

  tags = {
    Name = "MyRDS"
  }
}
```
