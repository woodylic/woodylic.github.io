---
title: 翻译：Working with AWS ECS
date: 2017-10-12 22:33:45
tags: [AWS, ECS, docker]
---

**源文链接**：[Working with AWS ECS](https://blog.rackspace.com/working-aws-ecs)（翻译：[woodylic](https://woodylic.github.io)）

# 前言

In this blog post, we will look at AWS ECS and how it can be used to deploy Docker containers.

From a use-case perspective, ECS allows you to build a production-scale,
auto-scaling and monitored platform for running these types of containers.

If you’re already using AWS, outside of the cost for EC2 instances, there is no additional cost
associated with using ECS — it’s a low friction entry point for testing and proof of concepts.

At a high level, ECS is a cluster management framework that provides management of EC2 (Elastic Compute)
instances running as Docker Hosts. It includes a scheduler to execute processes/containers on EC2 instances
and has constructs to deploy and manage versions of the applications. You can also combine this with AWS
Autoscale Groups and Load Balancing (classic and application) services. In addition, ECS also has auto-scaling
at the ECS service level. The goal when using ECS is to achieve the following:

- Deploy two services into ECS, behind a single load balancer, with diﬀerent target groups.
- Deploy ECS tasks and services using ECS cli.

Architecturally, it will look like this:

{% qnimg aws_ecs_arch_sample.png %}

It is assumed that you already have an AWS account and a VPC to deploy ECS onto.
It is also assumed that you have some familiarity with Git, Docker and AWS services such as VPC, ELB and EC2.

# ECS概念

Before we go any further, let’s get familiar with some ECS concepts and terms.

**Cluster** — A logical grouping of EC2 container instances. The cluster is a skeleton structure around which you build and operate workloads.

**Container instance(s)** — This is actually an EC2 instance running on the ECS agent. The recommended option is to use AWS ECS AMI but any AMI can be used as long as you add the ECS agent to it. The ECS agent is also open source.

**Container agent** — This is the agent that runs on EC2 instances to form the ECS cluster. If you’re using the ECS optimized AMI, you don’t need to do anything as the agent comes with it. But if you want to run your own OS/AMI, you will need to install the agent. The container agent is open source and can be found here:
https://github.com/aws/amazon-ecs-agent

**Task definition** — An application containing one or more containers. This is where you provide the Docker images, the amount of CPU/Memory to use, ports etc. You can also link containers here, similar to a Docker command line.

**Task** — An instance of a task definition running on a container instance.

**Service** — A service in ECS allows you to run and maintain a specified number of instances of a task definition. If a task in a service stops, the task is restarted. Services ensure that the desired running tasks are achieved and maintained. Services can also include things like load balancer configuration, IAM roles and placement strategies.

**Container** — A Docker container that is executed as part of a task.

Service auto-scaling — This is similar to the EC2 auto scaling concept but applies to the number of containers you’re running for each service. The ECS service scheduler respects the desired count at all times. Additionally, a scaling policy can be configured to trigger a scale-out based on alarms.

# Prepare security groups

Now that you have familiarized yourself with the baseline terminology, you’re ready to prepare your security groups. Here are the steps needed:

- Create a security group named stage01-alb. In this group, allow port 80 on inbound.
- Create a security group named stage01-ecs. In this group, allow all ports from the stage01-alb security group.
- In both groups, you can allow other ports if you desire but that is not needed for this tutorial.

# Prepare application load balancer

ECS will work with both classic load balancers and application load balancers. An application load balancer is the best fit for ECS because of features like dynamic ports and URL-based target groups. For this example, we will create two ALB instances; auth and random:

- You have to decide on the ports you will use for the services. In our example, we chose port 10080 for the auth service and 10081 for the random service. This port number is defined separately at all layers: load balancer, task definition, service definition and the actual listener port inside the container. Bottom line, it needs to match.
- Create an application load balancer that listens on port 80 and choose the appropriate subnets as per your VPC.
- Create two target groups: auth will be our default target group and random will be the second. Configure proper health checks as you would with any instance. Do not add any EC2 instances to it. This will be done later on the ECS service side. Here are the details:
- “HealthCheckPort”: “traﬃc-port” is an important parameter as this one works in conjunction with the Dynamic ports on the ECS side.

# Prepare Docker images (optional)

This can be any Docker image but for this example, we built two Docker images that run Ubuntu, Nginx and PHP-fpm. We then added some custom code to it. All the source is available here: https://github.com/rackerlabs/ecs-playground.

You can use it to build the Docker images yourself and upload them to your Docker hub. This example uses public images but you can also embed Docker Hub credentials and use private images. You can also use the AWS Elastic Container Registry but for the purpose of this example, we decided not to use that. You can also use the images from the provided Dockerhub account and skip this step altogether.

# Build ECS cluster

Now let’s create the ECS cluster. In this example, we’re building this in the us-east-1 region. ECS is available in most regions, but not all regions are covered.

- Follow the AWS guide to get the necessary IAM roles in place for ECS: Setting Up with Amazon ECS.

- Create an ECS cluster. We will name our cluster stage01. Note, this step only creates a framework for ECS and does not provision any resources.

- Create a file called ecs-bootstrap.txt with the following content. Since we named the cluster stage01, we need to pass that value to the ECS agent. If no value is passed, it assumes the cluster is named default.

- Next, we add EC2 instances to it. You can do this in the console or via the command line. Make sure the IAM role is pre-created and you have security group id from above. Also to make a cluster, spin up at least 2 instances. If you want to distribute this across availability zones, then you need this separately for each subnet id.

- Describe the cluster to make sure registeredContainerInstancesCount is set to the correct value (2).

At this point, we have the ECS cluster running with two instances and it’s ready to take on some workload.

# Deploy ECS tasks and services

The first step is to create a task definition which is an application containing one or more containers. Note, these JSON files are provided in the git repo. Let’s first look at a task definition file.

Parameters explained:

**Name** — is the task name.

**Image** — is the public image on DockerHub. You can also use a private hub or ECR.

**CPU** — is the number of CPU units to allocate (there are 1024 units per CPU core).

**Memory** — is the amount of memory to allocate in MB. Note, if your process consumes more than this, it is killed. This may require some testing to get it right. It’s also a good guardrail to contain the process. For example, if this belongs to a dev role, you can restrict its usage.

**PortMappings** — the one-to-one mapping for the EC2 host port and container port. This can be an array, so you have more than one. If you leave the host port as 0, ECS will automatically select a host port. You will need the host port to be set to 0 for dynamic ports to be able to work with the corresponding load balancer configuration. For our example, we’re using 10080 for Auth service and 10081 for the Random service. The host port is 0 and this allows ECS to pick a dynamic host port.

**DockerLabels** — Labels are a mechanism for applying metadata to Docker objects

**Family** — family is similar to a name for multiple versions of the task definition. The above list is a small subset of the parameters and is just scratching the surface. There are tons of diﬀerent options you can use in a task definition. You can read about all the parameters in detail here.

Register the task definition. This does not instantiate the task.

Verify using the cli.

Next, we create the services that use these task definitions. Here’s an example of the service file. Note, these JSON files are provided in the git repo.

Parameters explained:

**Cluster** — the ECS cluster name.

**serviceName** — The service name

**taskDefinition** — the task definition name

**loadBalancers** — this is where we plug in the load balancer target group from before. This is the ARN for it. The container name matches the task and you pick the same port you selected in the definition. Note, you can have multiple tasks inside a service.

**Role** — the ECS service IAM role. DesiredCount — the number of tasks you want to be always running. ECS will maintain this number and so any containers that crash will be replaced.

**deploymentConfiguration**ß — This group plays a key part in rolling updates.

**minimumHealthyPercent** — ensures that the number of running tasks never falls below it. The maximumPercent is an upper limit on the number of your tasks that are allowed in the RUNNING or PENDING state. This parameter enables you to define the deployment batch size. For example, if your service has a desiredCount of four tasks and a maximumPercent value of 200 percent, the scheduler may start four new tasks before stopping the four older tasks (provided that the cluster resources required to do this are available). The default value for maximumPercent is 200 percent.

**placementStrategy** — The algorithm for task placement or termination. There are 3 options: binpack, which places tasks based on the least available amount of CPU or memory, random, which places tasks randomly and spread, which places tasks evenly based on the specified attribute.

- Create the services. Make sure to replace the targetGroupArn and role in the json file with the respective ARNs. Finally, after all this work, we will have some containers running.

As per definitions, auth service will be started with two tasks(containers) and random will be started with four tasks. You can check the status via command line.

If all is working you will see a runningCount of two and the corresponding load balancer is now serving traﬃc. If you used the Docker images that were provided, then you can also access /discover.php which will print out some details.

Random service will have four running tasks and can be accessed via /random/.

# Ongoing management of ECS tasks and services

In a typical workflow, change to the application will start from a change to the Docker image. This image needs to be pushed to the Docker hub under a new tag. Then, you update the JSON file for the task definition with the new tag.

The only thing that changes in the JSON file is the image link/version. If you don’t use tags and rely on the latest Docker image, then no change is needed to the JSON file. The process to update is the same as registering the tasks.

Then we update the service to use the new version. In an update scenario, minimumHealthyPercent plays an important part. This ensures the upgrade is rolled out in a rolling fashion and the service never goes below the minimumHealthyPercent.

https://gist.github.com/8ed459700d74dc683a36ade394c42d3f

# Parting thoughts

There is a lot more to ECS, I recommend you spend some time with the AWS console, and especially the metrics tab, to view the usage of your cluster. Here are some other points worth considering:

- AWS recommends using their EC2 AMIs for the EC2 instances. If you want more control over this or want to run CoreOS or similar, then you need to bake ECS agents into those AMIs. The ECS agent is open source and so you can easily do that. Always run the latest agent version or the Container agent. The earlier versions had a few bugs and ECS is a changing ecosystem.
- If you want to use AWS Linux as a Docker Image, please refer to instructions under https://hub.docker.com/_/amazonlinux/
- Service discovery, which is key in the micro-services world, is limited inside ECS. You can use environment variables in task definitions but that isn’t a service discovery, per say. You can run something like Consul or Weave and use it for service discovery. It is also possible that AWS will release some new features or services that solve for this use case.
- Use ECS schedulers for auto-recovery of services. ECS has few in-built options and also allows external schedulers. So you can use something like Mesos with ECS.
- There is no central Docker endpoint as far as the ECS cluster is concerned. As a developer, you cannot point your local Docker client to an ECS endpoint. Note, while some may see this as a limitation, the purpose of ECS is not to provide a Docker endpoint. The purpose of ECS is targetted at running Docker instances from an operations perspective.
- Windows containers are now available in beta. These are launched with the Microsoft Windows Server 2016 Base with Containers AMI. Note, Windows on ECS is not at feature parity with Linux. Also, the Windows images are much larger, which means the spin-up time is high the first time you run a Windows container. As with most AWS services, this will change and improve over time.

As always, visit Rackspace to find out more about our AWS-certified experts and the AWS features and services they recommend.

# References and Links

- http://docs.aws.amazon.com/cli/latest/reference/ecs/
- http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/processor_state_control.html
