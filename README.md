# THREE-TIER-ARCHITECTURE
## Prerequisites
- Install https://learn.hashicorp.com/tutorials/terraform/install-cli
- Install the https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
- Sign up for an https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/
- Your preferred IDE (You can use visual studio code)


## Notes Before Getting Started: 
- You can clone this the full code from here
- A typical 3-tier architecture uses a web, application and database layer. In this project, an application layer was created, instances in the application layer was deployed and a Security Group of the application layer was created and some Security Group rules modified and created an application layer security group.
- RDS has Multi-AZ set to true for high availability. For demo purposes to save money, you'd be sure to set this to false


## Input Variables and the Count Parameter: 
- Having repeated static values in your Terraform code can create more work for you in the future. Imagine that you have the same value repeated throughout your file and one day you have a request to change that value. That is where variables (https://www.terraform.io/docs/language/values/variables.html#suppressing-values-in-cli-output) come in to make your job easier. Variables allow you to have a central source where you can import values from. Instead of updating the value in each location of your code, you only have to update them once in a central source. 
- Using the count (https://www.terraform.io/docs/language/meta-arguments/count.html) parameter on resources can simplify your configurations and lets you scale resources by incrementing. The count parameter acts similar to a loop. For instance if you’d like to create two EC2 instances, you could either duplicate your code twice or you could include count = 2.

 ### Website Script
 1. Create a new directory for this Terraform Project.
 2. Inside the new directory create a file named install_apache.sh. This will install apache web server on our instances and create a unique landing page for each so we can verify the Application Load Balancer is working.
 
 ### Configure Provider
 Providers are plugins that Terraform requires so that it can install and use for your Terraform configuration. Terraform offers the ability to use a variety of Providers, so it doesn’t make sense to use all of them for each file. We will declare our Provider as AWS.
 1. Create a main.tf file and add each of the following sections to the main.tf file.
 2. From the terminal in the Terraform directory containing install_apache.sh and main.tf run terraform init
 
 ### Create VPC and Subnets with thier variables
 1. Our first Resource is creating our VPC with their variables
 2. web-subnet-1 and web-subnet-2 resources create our web layer in two availability zones. Notice that we have map_public_ip_on_launch = true
 3. application-subnet-1 and application-subnet-2 resources create our application layer in two availability zones. This will be a private subnet.
 4. database-subnet-1 and database-subnet-2 resources create our database layer in two availability zones. This will be a private subnet.
 - Once again we are going to add the variables. Rather than making a variable for each Availability Zone and a variable for each Web Subnet CIDR, giving us a total of four variable blocks, we will create two variable blocks with lists for their defaults.
 - When we add our variables we can reference the variable list index numbers to get our desired variable. For example our var.availability_zone_names[0] will return the first list item in the variable “availability_zone_names” block, which is “ca-central-1a”. Remember that indexes start with 0, so the first item in a list has an index of 0 and the second has an index of 1 and so on
 - Adding variables helps clean up our code a little but we can still do more. We are still using two resources that are very similar except for the CIDR and Availability Zone. That’s where count comes in. When we add a count, it will basically loop through the number of times specified. Here we added a count of 2. When it loops through the first time the count.index will be 0 pulling the first item in each of the variable lists. For the tags we use ${ } because they are needed when using count.index in a string. Here we want the tag name to be Web-1 and Web-2. Since indexes start with 0, we add 1 to the count.index to return 1 & 2 instead of 0 & 1.
 
 ### Create Internet Gateway and Route Table
 1. Our first resource block will create an Internet Gateway. We will need an Internet Gateway to allow our public subnets to connect to the Internet.
 2. Just saying that our subnets are public does not make it so. We will need to create a route table and Associate our Web Layer subnets.
 3. The web-rt route table creates a route in our VPC to our Internet Gateway
 4. The next two blocks are associating web-subnet-1 and web-subnet-2 with the web-rt route table. I feel like you should be able to associate more than one subnet in a single block but I kept getting errors and wasn’t able to find any helpful documentation.
 
 ### Create Web Servers
 1. webserver1 resource creates a Linux 2 EC2 instance in the us-east-1a Availability Zone.
 2. ami is set to the ami id for the Linux 2 AMI for the us-east-1 Region. If using a different Region then you’d need to update.
 3. vpc_security_group_ids is set to a not yet created Security Group, which will be created in the next section for our Application Load Balancer. Terraform doesn’t
 4. create infrastructure in order. It is smart enough to know what needs to be created before others (for the most part, I’ll talk more about this later).
user_data is used to boot strap our instance. Rather than type our the code directly, we will reference the install_apache.sh file we created earlier.
5. webserver2 is almost identical except that availability_zone is set to us-east-1b.
 
 ### Create Security Groups
 1. Create a Security Group named web-sg with inbound rule opening HTTP port 80 and allowing all outbound traffic.
 2. Create a Security Group named webserver-sg with inbound rule opening HTTP port 80, but this time it’s not open to the world. Instead we are only allowing traffic from our web-sg Security Group.
 3. Create a Security Group named database-sg with inbound rule opening MySQL port 3306 and once again we keep security tight by only allow the inbound traffic from the webserver-sg Security Group. We open outbound traffic to all the ephemeral ports.
 
 ### Create Application Load Balancer
 1. Create an external Application Load Balancer.
- internal is set to false, making it an external Load Balancer.
- load_balancer_type is set to application designating it an Application Load Balancer.
- security_groups is set to our web-sg Security Group which allows access from the internet over port 80.
- subnets is set to both of our web subnets. This designates where the ALB will send traffic and requires a minimum of two subnets in two different AZs.
2. Create an Application Load Balancer Target Group.
3. The aws_lib_target_group_attachment Resource attaches our instances to the Target Group. Note that I’ve added depends_on to both of these. I kept experiencing an issue where my instances kept showing as unhealthy in the Target Group because they weren’t done initializing. By setting the depends_on to their respective web server, the issue was resolved.
4. Add a listener on port 80 that forwards traffic to our Target Group.
 
 ### Create RDS Instance
 1. Create an MySQL RDS Instance. Some attributes to note:
- db_subnet_group_name is a required field and is set to the aws_db_subnet_group.default.
- instance_class is set to a db.t2.micro.
- multi_az is set to true for high availability, but if you’d like to keep costs low, set this to false.
- username & password will need to be changed.
- vpc_secuiryt_group_ids is set to our database-sg Security Group.
2. Create a DB Subnet Group. subnet_ids identifies which subnets will be used by the Database.
 
 ### RDS Code Block
 The RDS block required a slightly different variable set up. To change everything over to variables we use a map(any) type. This will allow us to quickly create all the needed parameters rather than making a variable block for each. If you notice I set up two variable blocks. One for rds_instance and another for user_information. This is because the user_information contains our username and password for the RDS instance which has been marked as sensitive. This will prevent the information from being displayed in the output. To ensure these sensitive values are not uploaded to a public repo, you may want to move them to a terraform.tvars file.
 
 To access the mappings from the rds_instance and user_information variables we will use the following configuration: var.<variable name>.<map key value>.

### Output
 After your infrastructure completes, Output will print out the requested values. We will use output to print out our ALB DNS so we can test our web servers.

### Provision Infrastructure
1. If you didn’t do so earlier or you just want to do it again, from the terminal run terraform init .
2. Run terraform fmt . This ensures your formatting is correct and will modify the code for you to match.
3. Run terraform validate to ensure there are no syntax errors.
4. Run terraform plan to see what resources will be created.
5. Run terraform apply to create your infrastructure. Type Yes when prompted.

### Testing
1. After your infrastructure has been created there should be an Output displayed on your terminal for the Application Load Balancer DNS Name.
2. Copy and paste (without quotations) into a new browser tab. Refresh the page to see the load balancer switch between the two instances.

### Clean Up
To delete our infrastructure run terraform destroy . When prompted type Yes. This command will delete all the infrastructure that we created.
