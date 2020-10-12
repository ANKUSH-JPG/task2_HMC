# task2_HMC

LET'S DIRECTLY JUMP TO THE STEPS INVOLVED:

 1. First of all, create an IAM user by going to AWS GUI, and don't forget to download your Credentials, and store it somewhere safe, because if it gets lost(I mean accidentally deleted), then there is no way of recovering it whatsoever, and you need to make new IAM user, in my case I gave my IAM user Administrator Access.
 
# NOTE: Always make sure to not upload your Access Key ID and Secret Access Key, even if it is GitHub, because someone might use your key, and you may get a huge bill.

# TASK:

 1. Create a Security group which allow the port 80.
 2. Launch EC2 instance.
 3. In this Ec2 instance use the existing key or provided key and security group which we have created in step 1.
 4. Launch one Volume using the EFS service and attach it in your vpc, then mount that volume into /var/www/html
 5. The developer has uploaded the code into GitHub repo also the repo has some images.
 6. Copy the github repo code into /var/www/html
 7. Create an S3 bucket, and copy/deploy the images from GitHub repo into the s3 bucket and change the permission to public readable.
 8. Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to update in code in /var/www/html
 
# STEPS :

  1. Create a VPC, which we will be using it later, the command for the same is:
       
        resource "aws_vpc" "ankush2_vpc" {
                cidr_block = "192.168.0.0/16"
                instance_tenancy = "default"
                enable_dns_hostnames = true
                tags = {
                  Name = "ankush2_vpc"
                }
              }
              
  2. Now we will create subnet which we will be using for launching instance later, the command for the same is:
  
         resource "aws_subnet" "ankush2_subnet" {
                vpc_id = "${aws_vpc.ankush2_vpc.id}"
                cidr_block = "192.168.0.0/24"
                availability_zone = "ap-south-1a"
                map_public_ip_on_launch = "true"
                tags = {
                  Name = "ankush2_subnet"
                }
              }
              
  3. I will be using and making a custom security group with all of the required permissions, which I will be using to launch my instance later, the code for the same:
   
          resource "aws_security_group" "ankush2_sg" {

                name        = "ankush2_sg"
                vpc_id      = "${aws_vpc.ankush2_vpc.id}"


                ingress {

                  from_port   = 80
                  to_port     = 80
                  protocol    = "tcp"
                  cidr_blocks = [ "0.0.0.0/0"]

                }


                ingress {

                  from_port   = 2049
                  to_port     = 2049
                  protocol    = "tcp"
                  cidr_blocks = [ "0.0.0.0/0"]

                }



                ingress {

                  from_port   = 22
                  to_port     = 22
                  protocol    = "tcp"
                  cidr_blocks = [ "0.0.0.0/0"]

                }




                egress {

                  from_port   = 0
                  to_port     = 0
                  protocol    = "-1"
                  cidr_blocks = ["0.0.0.0/0"]
                }


                tags = {

                  Name = "ankush2_sg"
                }
              }
              
   4. In this step, we will be creating an EFS account and configure it:
      
            resource "aws_efs_file_system" "ankush2_efs" {
                creation_token = "ankush2_efs"
                tags = {
                  Name = "ankush2_efs"
                }
              }


              resource "aws_efs_mount_target" "ankush2_efs_mount" {
                file_system_id = "${aws_efs_file_system.ankush2_efs.id}"
                subnet_id = "${aws_subnet.ankush2_subnet.id}"
                security_groups = [aws_security_group.ankush2_sg.id]
              }
              
   5. In this step, we will create a Gateway and a Routing table, the command for the same is:
   
            resource "aws_internet_gateway" "ankush2_gw" {
                vpc_id = "${aws_vpc.ankush2_vpc.id}"
                tags = {
                  Name = "ankush2_gw"
                }
              }


              resource "aws_route_table" "ankush2_rt" {
                vpc_id = "${aws_vpc.ankush2_vpc.id}"

                route {
                  cidr_block = "0.0.0.0/0"
                  gateway_id = "${aws_internet_gateway.ankush2_gw.id}"
                }

                tags = {
                  Name = "ankush2_rt"
                }
              }


              resource "aws_route_table_association" "ankush2_rta" {
                subnet_id = "${aws_subnet.ankush2_subnet.id}"
                route_table_id = "${aws_route_table.ankush2_rt.id}"
              }
              
   6. Now the time has come to finally launch our instance. 
   
           resource "aws_instance" "test_ins" {
                        ami             =  "ami-052c08d70def0ac62"
                        instance_type   =  "t2.micro"
                        key_name        =  "ankush2_key"
                        subnet_id     = "${aws_subnet.ankush2_subnet.id}"
                        security_groups = ["${aws_security_group.ankush2_sg.id}"]


                       connection {
                          type     = "ssh"
                          user     = "ec2-user"
                          private_key = file("C:/Users/Naitik/Downloads/ankush2_key.pem")
                          host     = aws_instance.test_ins.public_ip
                        }

                        provisioner "remote-exec" {
                          inline = [
                            "sudo yum install amazon-efs-utils -y",
                            "sudo yum install httpd  php git -y",
                            "sudo systemctl restart httpd",
                            "sudo systemctl enable httpd",
                            "sudo setenforce 0",
                            "sudo yum -y install nfs-utils"
                          ]
                        }

                        tags = {
                          Name = "my_os"
                        }
                      }
                      
  7. Now, as our instance is launched, we will mount our EFS volume to /var/www/html folder, where all of our codes is stored, this will ensure that there is no data loss in case the instance is accidentally deleted or if they crash.
  
                resource "null_resource" "mount"  {
                depends_on = [aws_efs_mount_target.ankush2_efs_mount]
                connection {
                  type     = "ssh"
                  user     = "ec2-user"
                  private_key = file("C:/Users/ankush/Downloads/ankush2_key.pem")
                  host     = aws_instance.test_ins.public_ip
                }
              provisioner "remote-exec" {
                  inline = [
                    "sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ${aws_efs_file_system.naitik2_efs.id}.efs.ap-south-1.amazonaws.com:/ /var/www/html",
                    "sudo rm -rf /var/www/html/*",
                    "sudo git clone https://github.com/ANKUSH-JPG/task2_HMC.git /var/www/html/",
                    "sudo sed -i 's/url/${aws_cloudfront_distribution.my_front.domain_name}/g' /var/www/html/index.html"
                  ]
                }
              }
              
 8. Now, we create an S3 bucket on AWS. The code snippet for doing the same is as follows:
 
             resource "aws_s3_bucket" "sp_bucket" {
        bucket = "ankush2"
        acl    = "private"

        tags = {
          Name        = "ankush2314"
        }
        }
        locals {s3_origin_id = "myS3Origin"
            }
            
 9. As S3 bucket is created, we will upload images downloaded from GitHub to our local system in the above step. In this task, I will be uploading only one picture .
 
        resource "aws_s3_bucket_object" "object" {
          bucket = "${aws_s3_bucket.sp_bucket.id}"
          key    = "test_pic"
          source = "C:/Users/ankush/Pictures/picture1.jpg"
          acl    = "public-read"
        }
        
 10. Now, we create a CloudFront & connect it to our S3 bucket. The CloudFront ensures speedy delivery of content using the edge locations from AWS across the world.
 
          resource "aws_cloudfront_distribution" "my_front" {
           origin {
               domain_name = "${aws_s3_bucket.sp_bucket.bucket_regional_domain_name}"
               origin_id   = "${local.s3_origin_id}"

         custom_origin_config {

               http_port = 80
               https_port = 80
               origin_protocol_policy = "match-viewer"
               origin_ssl_protocols = ["TLSv1", "TLSv1.1", "TLSv1.2"] 
              }
            }
               enabled = true

         default_cache_behavior {

               allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
               cached_methods   = ["GET", "HEAD"]
               target_origin_id = "${local.s3_origin_id}"

         forwarded_values {

             query_string = false

         cookies {
                forward = "none"
               }
           }

                viewer_protocol_policy = "allow-all"
                min_ttl                = 0
                default_ttl            = 3600
                max_ttl                = 86400

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
      
11. Now, we write a terraform code snippet to automatically retrieve the public IP of our instance and open it in chrome. This will land us on the home page of our website that is present in /var/www/html.
 
        resource "null_resource" "local_exec"  {


        depends_on = [
            null_resource.mount,
          ]

          provisioner "local-exec" {
              command = "start chrome  ${aws_instance.test_ins.public_ip}"
                 }
        }
        
        
# CLI OUTPUT(OUTPUT FROM CMD):

1. head to your command prompt and run this command:
    
       terraform init
   
 ![1](https://user-images.githubusercontent.com/51692515/95707303-c83c4780-0c76-11eb-9ead-0d235480e868.png)
 
 2. Run the command:
  
        terraform apply
        
![2](https://user-images.githubusercontent.com/51692515/95707407-17827800-0c77-11eb-9520-596a6c72fd56.png)

![3](https://user-images.githubusercontent.com/51692515/95707409-18b3a500-0c77-11eb-95dd-bd09f22621dc.png)

![4](https://user-images.githubusercontent.com/51692515/95707410-194c3b80-0c77-11eb-97b9-b52586e8d3a4.png)

![5](https://user-images.githubusercontent.com/51692515/95707413-19e4d200-0c77-11eb-8ea1-73cbf213af05.png)

![6](https://user-images.githubusercontent.com/51692515/95707414-1a7d6880-0c77-11eb-8e4f-c530f7f5ed55.png)

![7](https://user-images.githubusercontent.com/51692515/95707415-1bae9580-0c77-11eb-9537-907659e435c1.png)


# GUI OUTPUT(OUTPUT FROM THE CONSOLE):

![8](https://user-images.githubusercontent.com/51692515/95707642-b7400600-0c77-11eb-83d5-aa13e7656394.png)

![9](https://user-images.githubusercontent.com/51692515/95707647-b909c980-0c77-11eb-93e9-4536cf5a7d32.png)

![10](https://user-images.githubusercontent.com/51692515/95707649-b909c980-0c77-11eb-9b35-0552c19b0a72.png)

![11](https://user-images.githubusercontent.com/51692515/95707650-b9a26000-0c77-11eb-9502-44b4b4c5e64e.png)

![12](https://user-images.githubusercontent.com/51692515/95707651-ba3af680-0c77-11eb-9627-2366bd4fd7f5.png)

![13](https://user-images.githubusercontent.com/51692515/95707653-bad38d00-0c77-11eb-91d6-6252dc5a1bf3.png)

![14](https://user-images.githubusercontent.com/51692515/95707654-bb6c2380-0c77-11eb-9e63-c6942b22dba8.png)

![15](https://user-images.githubusercontent.com/51692515/95707655-bc04ba00-0c77-11eb-8293-4fe065493a29.png)

![16](https://user-images.githubusercontent.com/51692515/95707656-bc9d5080-0c77-11eb-8a9f-3bc767fdcd51.png)

![17](https://user-images.githubusercontent.com/51692515/95707659-bd35e700-0c77-11eb-9c55-e342ce26681e.png)

![18](https://user-images.githubusercontent.com/51692515/95707660-bdce7d80-0c77-11eb-8070-7b551305e99f.png)

  
  

