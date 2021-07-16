#                                  Deploy Dockerised Application on AWS ECS With Terraform

# Creating the Dockerfile
## Packaging the java based application into a Dockerfile
### Creating the image repo and tag the Image and Push the Image to repo
#### Create an K8's Cluster (self Managed/AWS ECS cluster)
##### Create an Deployment manifeast.yaml 
###### Create a Service for Deployment (Service.yaml)
###### Ceeate a Load balancer


** Kubernetes **
** Docker **
** Terraform **

# prerequisites
 ##  AWS 
 ## Docker 
 ## Terraform


# Creating the sample web application and convert into Docker Image 

```
Dockerfile
FROM tomcat:8-jre8

RUN rm -rf /usr/local/tomcat/webapps/*

COPY target/gameoflife.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]

```

# Build the Dockerfile to create the Docker Image 

```
Docker image build -t mygol:0.1 .

```

# Push the docker image to repo (ECR/Dockerhub/Private)

```
Docker push image name and tagname:0.1 repo name 

```
## By using the Terraform we can perform same acton to push Dockerimg to Repo by creating the file called main.tf 

```

provider "aws" {
  version = "~> 3.0"
  region  = "eu-west-2" # Setting my region to London. Use your own region here
}

resource "aws_ecr_repository" "my_first_ecr_repo" {
  name = "my-first-ecr-repo" # Naming my repository
}

```

## Run the main.tf in terminal by following commands 

```

terraform init
terraform apply

```
** We can observe the output by verifing the Repo **

# Create the kubernetes cluster as mentioned above 

```

resource "aws_ecs_task_definition" "my_first_task" {
  family                   = "my-first-task" # Naming our first task
  container_definitions    = <<DEFINITION
  [
    {
      "name": "my-first-task",
      "image": "${aws_ecr_repository.my_first_ecr_repo.repository_url}",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3000,
          "hostPort": 3000
        }
      ],
      "memory": 512,
      "cpu": 256
    }
  ]
  DEFINITION
  requires_compatibilities = ["FARGATE"] # Stating that we are using ECS Fargate
  network_mode             = "awsvpc"    # Using awsvpc as our network mode as this is required for Fargate
  memory                   = 512         # Specifying the memory our container requires
  cpu                      = 256         # Specifying the CPU our container requires
  execution_role_arn       = "${aws_iam_role.ecsTaskExecutionRole.arn}"
}

resource "aws_iam_role" "ecsTaskExecutionRole" {
  name               = "ecsTaskExecutionRole"
  assume_role_policy = "${data.aws_iam_policy_document.assume_role_policy.json}"
}

data "aws_iam_policy_document" "assume_role_policy" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role_policy_attachment" "ecsTaskExecutionRole_policy" {
  role       = "${aws_iam_role.ecsTaskExecutionRole.name}"
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

                                          
```

# Create a Service 

```

resource "aws_ecs_service" "my_first_service" {
  name            = "my-first-service"                             # Naming our first service
  cluster         = "${aws_ecs_cluster.my_cluster.id}"             # Referencing our created Cluster
  task_definition = "${aws_ecs_task_definition.my_first_task.arn}" # Referencing the task our service will spin up
  launch_type     = "FARGATE"
  desired_count   = 3 # Setting the number of containers we want deployed to 3
}

```
 
## Create loadbalancer 


```                                          
resource "aws_lb_target_group" "target_group" {
  name        = "target-group"
  port        = 80
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = "${aws_default_vpc.default_vpc.id}" # Referencing the default VPC
  health_check {
    matcher = "200,301,302"
    path = "/"
  }
}

resource "aws_lb_listener" "listener" {
  load_balancer_arn = "${aws_alb.application_load_balancer.arn}" # Referencing our load balancer
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = "${aws_lb_target_group.target_group.arn}" # Referencing our tagrte group
  }
}

                                          
```
# as we need to able the access our container through load balancer 
