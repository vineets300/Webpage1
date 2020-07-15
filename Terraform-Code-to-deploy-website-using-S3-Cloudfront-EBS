#Define the provider
provider "aws" {
    region = "ap-southeast-1"
}

#Create a virtual network
resource "aws_vpc" "my_vpc" {
    cidr_block = "10.0.0.0/16"
    tags = {
        Name = "MY_VPC"
    }
}

#Create your application segment
resource "aws_subnet" "my_app-subnet" {
    tags = {
        Name = "APP_Subnet"
    }
    vpc_id = aws_vpc.my_vpc.id
    cidr_block = "10.0.1.0/24"
    map_public_ip_on_launch = true
    depends_on= [aws_vpc.my_vpc]
    
}

#Define routing table
resource "aws_route_table" "my_route-table" {
    tags = {
        Name = "MY_Route_table"
       
    }
     vpc_id = aws_vpc.my_vpc.id
}

#Associate subnet with routing table
resource "aws_route_table_association" "App_Route_Association" {
  subnet_id      = aws_subnet.my_app-subnet.id 
  route_table_id = aws_route_table.my_route-table.id
}


#Create internet gateway for servers to be connected to internet
resource "aws_internet_gateway" "my_IG" {
    tags = {
        Name = "MY_IGW"  
    }
     vpc_id = aws_vpc.my_vpc.id
     depends_on = [aws_vpc.my_vpc]
}

#Add default route in routing table to point to Internet Gateway
resource "aws_route" "default_route" {
  route_table_id = aws_route_table.my_route-table.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.my_IG.id
}

#Create a security group
resource "aws_security_group" "App_SG" {
    name = "App_SG"
    description = "Allow Web inbound traffic"
    vpc_id = aws_vpc.my_vpc.id
    ingress  {
        protocol = "tcp"
        from_port = 80
        to_port  = 80
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress  {
        protocol = "tcp"
        from_port = 22
        to_port  = 22
        cidr_blocks = ["0.0.0.0/0"]
    }

    egress  {
        protocol = "-1"
        from_port = 0
        to_port  = 0
        cidr_blocks = ["0.0.0.0/0"]
    }
}

#Create a private key which can be used to login to the webserver
resource "tls_private_key" "Web-Key" {
  algorithm = "RSA"
}

#Save public key attributes from the generated key
resource "aws_key_pair" "App-Instance-Key" {
  key_name   = "Web-key"
  public_key = tls_private_key.Web-Key.public_key_openssh
}

#Save the key to your local system
resource "local_file" "Web-Key" {
    content     = tls_private_key.Web-Key.private_key_pem 
    filename = "Web-Key.pem"
}

#Create your webserver instance
resource "aws_instance" "Web" {
    ami = "ami-0adbe59da7d24a349"
    instance_type = "t2.micro"
    tags = {
        Name = "WebServer1"
    }
    count =1
    subnet_id = aws_subnet.my_app-subnet.id 
    key_name = "Web-key"
    security_groups = [aws_security_group.App_SG.id]

    provisioner "remote-exec" {
    connection {
        type = "ssh"
        user = "ec2-user"
        private_key = tls_private_key.Web-Key.private_key_pem
        host = aws_instance.Web[0].public_ip
    }    
    inline = [
       "sudo yum install httpd  php git -y",
       "sudo systemctl restart httpd",
       "sudo systemctl enable httpd",
    ]
  }

}

#Create a block volume for data persistence
resource "aws_ebs_volume" "myebs1" {
  availability_zone = aws_instance.Web[0].availability_zone
  size              = 1
  tags = {
    Name = "ebsvol"
  }
}

#Attach the volume to your instance
resource "aws_volume_attachment" "attach_ebs" {
  depends_on = [aws_ebs_volume.myebs1]
  device_name = "/dev/sdh"
  volume_id   = aws_ebs_volume.myebs1.id
  instance_id = aws_instance.Web[0].id
  force_detach = true
}

#Mount the volume to your instance
resource "null_resource" "nullmount" {
  depends_on = [aws_volume_attachment.attach_ebs]
    connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = tls_private_key.Web-Key.private_key_pem
    host     = aws_instance.Web[0].public_ip
  }
  provisioner "remote-exec" {
    inline = [
      "sudo mkfs.ext4 /dev/xvdh",
      "sudo mount /dev/xvdh /var/www/html",
      "sudo rm -rf /var/www/html/*",
      "sudo git clone https://github.com/vineets300/Webpage1.git  /var/www/html"
    ]
  }
}

#Define S3 ID
locals {
 s3_origin_id = "s3-origin"
}

#Create a bucket to upload your static data like images
resource "aws_s3_bucket" "demonewbucket12345" {
  bucket = "demonewbucket12345"
  acl    = "public-read-write"
  region = "ap-southeast-1"
  
  versioning {
    enabled = true
  }

  tags = {
    Name = "demonewbucket12345"
    Environment = "Prod"
  }

 provisioner "local-exec" {
    command = "git clone https://github.com/vineets300/Webpage1.git web-server-image"
 }

}

#Allow public access to the bucket
resource "aws_s3_bucket_public_access_block" "public_storage" {
 depends_on = [aws_s3_bucket.demonewbucket12345]
 bucket = "demonewbucket12345"
 block_public_acls = false
 block_public_policy = false
}

#Upload your data to S3 bucket
resource "aws_s3_bucket_object" "Object1" {
  depends_on = [aws_s3_bucket.demonewbucket12345]
  bucket = "demonewbucket12345"
  acl    = "public-read-write"
  key = "Demo1.PNG"
  source = "web-server-image/Demo1.PNG"
}

#Create a Cloudfront distribution for CDN
resource "aws_cloudfront_distribution" "tera-cloufront1" {
    depends_on = [ aws_s3_bucket_object.Object1]
    origin {
        domain_name = aws_s3_bucket.demonewbucket12345.bucket_regional_domain_name
        origin_id = local.s3_origin_id
    }   
    enabled = true
      default_cache_behavior {
        allowed_methods = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
        cached_methods = ["GET", "HEAD"]
        target_origin_id = local.s3_origin_id

        forwarded_values {
            query_string = false
        
            cookies {
               forward = "none"
            }
        }
        viewer_protocol_policy = "allow-all"
        min_ttl = 0
        default_ttl = 3600
        max_ttl = 86400
    }

    restrictions {
        geo_restriction {
           restriction_type = "none"
        }
    }

     viewer_certificate {
        cloudfront_default_certificate = true

    } 
}

#Update the CDN image URL to your webserver code.
resource "null_resource" "Write_Image" {
    depends_on = [aws_cloudfront_distribution.tera-cloufront1]
    connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = tls_private_key.Web-Key.private_key_pem
    host     = aws_instance.Web[0].public_ip
     }
  provisioner "remote-exec" {
        inline = [
            "sudo su << EOF",
                    "echo \"<img src='http://${aws_cloudfront_distribution.tera-cloufront1.domain_name}/${aws_s3_bucket_object.Object1.key}' width='300' height='380'>\" >>/var/www/html/index.html",
                    "echo \"</body>\" >>/var/www/html/index.html",
                    "echo \"</html>\" >>/var/www/html/index.html",
                    "EOF",    
        ]
  }

}

#success message and storing the result in a file
resource "null_resource" "result" {
    depends_on = [null_resource.nullmount]
    provisioner "local-exec" {
    command = "echo The website has been deployed successfully and >> result.txt  && echo the IP of the website is  ${aws_instance.Web[0].public_ip} >>result.txt"
  }
}

#Test the application
resource "null_resource" "running_the_website" {
    depends_on = [null_resource.Write_Image]
    provisioner "local-exec" {
    command = "start chrome ${aws_instance.Web[0].public_ip}"
  }
}
