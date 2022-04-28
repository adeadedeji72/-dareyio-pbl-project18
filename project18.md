# AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 3 – REFACTORING #

### Introducing Backend in Terrafrom on S3 ###

Each Terraform configuration can specify a backend, which defines where and how operations are performed, where state snapshots are stored, etc.
*Local backend* – requires no configuration, and the states file is stored locally. This mode can be suitable for learning purposes, 
but it is not a robust solution, so it is better to store it in some more reliable and durable storage.

The second problem with storing this file locally is that, in a team of multiple DevOps engineers, other engineers will not have access to a state file 
stored locally on your computer.
To solve this, we will need to configure a backend where the state file can be accessed remotely other DevOps team members. There are plenty of different 
standard backends supported by Terraform that you can choose from. Since we are already using AWS – we can choose an S3 bucket as a backend.

Another useful option that is supported by S3 backend is State Locking – it is used to lock your state for all operations that could write state. 
This prevents others from acquiring the lock and potentially corrupting your state. State Locking feature for S3 backend is optional and requires 
another AWS service – *DynamoDB*.

### **Create S3 and DynamoDB Resources** ###
1. Create a backend.tf file
1. Add this snippet to the file
~~~
# Note: The bucket name may not work for you since buckets are unique globally in AWS, so you must give it a unique name.
resource "aws_s3_bucket" "terraform_state" {
  bucket = "masterclass-s3-bucket"
  # Enable versioning so we can see the full revision history of our state files
  versioning {
    enabled = true
  }
  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
~~~
Terraform stores secret data inside the state files. Passwords, and secret keys processed by resources are always stored in there. Hence, you must consider to 
always enable encryption.

create a DynamoDB table to handle locks and perform consistency checks. In previous projects, locks were handled with a local file as shown in 
terraform.tfstate.lock.info. Since we now have a team mindset, causing us to configure S3 as our backend to store state file, we will do the same 
to handle locking. Therefore, with a cloud storage database like DynamoDB, anyone running Terraform against the same infrastructure can use a central 
location to control a situation where Terraform is running at the same time from multiple different people.

Create a DynamoDB resouce in the main.tf file with this code:
~~~
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
~~~

Terraform expects that both S3 bucket and DynamoDB resources are already created before we configure the backend. 
So, let us run terraform apply to provision resources.

### Configure S3 Backend ###

~~~
terraform {
  backend "s3" {
    bucket         = "dev-terraform-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
~~~
