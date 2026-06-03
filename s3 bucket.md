## s3 bucket
```
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "mybucket" {
  bucket = "my-unique-demo-bucket-0000"

  tags = {
    Name        = "MyBucket"
    Environment = "Dev"
  }
}
```
