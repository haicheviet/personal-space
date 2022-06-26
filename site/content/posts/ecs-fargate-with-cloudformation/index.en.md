---
title: "ECS Fargate With Cloudformation"
subtitle: ""
date: 2022-06-25T16:57:12+07:00
lastmod: 2022-06-25T16:57:12+07:00
author: ""
description: ""

resources:
- name: "featured-image"
  src: "featured-image.png"
---

Deploying individual containers is not difficult. However, when you need to coordinate many container deployments, a container management tool like ECS can greatly simplify the task (no pun intended).

For serverless approach, the best combination recipe for ECS cluster is AWS on-demnd and spot Fargate. The blog will describe how we can deploy ECS cluster by infrastructure as code (IAC) with optimal cost

<!--more-->

## Introduction

{{< admonition >}}

AWS Fargate is a technology that you can use with Amazon ECS to run containers without having to manage servers or clusters of Amazon EC2 instances. With Fargate, you no longer have to provision, configure, or scale clusters of virtual machines to run containers. This removes the need to choose server types, decide when to scale your clusters, or optimize cluster packing. [Ref](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)

{{< /admonition >}}

**Insert fargate.png**

ECS consists out of a few components:

* Elastic Container Repository (ECR): A Docker repository to store your Docker images (similar as DockerHub but now provisioned by AWS).
* Task Definition: ECS refers to a JSON formatted template called a Task Definition  that describes one or more containers making up your application or service. The task definition is the recipe that ECS uses to run your containers as a task on your EC2 instances or AWS Fargate.
* ECS Cluster: The Cluster definition itself where you will specify how many instances you would like to have and how it should scale.
* Service: Based on a Task Definition, you will deploy the container by means of a Service into your Cluster.

You will create a Docker image for a basic Nginx applicaion, upload it to ECR, create a full working solution ECS cluster with autoscaling and capacity provider strategy for optimal cost

*Insert design.png*

TLDRL The sources being used in this blog are available at [GitHub]().

## Cloudforamtion IAC structure

Our cloud formation was structure by these following components:

* Description
The template Description enables you to provide arbitrary comments about your template. It is a literal variable string that is between 0 and 1024 bytes in length; its value cannot be based on a parameter or function.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'VPC: public and private subnets in two availability zones'
..
```

* Parameters and Metadata

The optional Parameters section enables you to pass values into your template at stack creation time. Parameters let you create templates that can be customized for each stack deployment. When you create a stack from a template containing parameters, you can specify values for those parameters. Within the template, you can use the “Ref” intrinsic function to specify those parameter values in properties values for resources. For example, you can pass parameters in the AWS CLI or you can type the values in the AWS console while creating an instance.

You can use the optional Metadata section to include arbitrary JSON or YAML objects that provide details about the template. For example, you can include template implementation details about specific resources, as shown in the following snippet:

```yaml
Parameters:
  ClassB:
    Description: 'Class B of VPC (10.XXX.0.0/16)'
    Type: Number
    Default: 0
    ConstraintDescription: 'Must be in the range [0-255]'
    MinValue: 0
    MaxValue: 255
Metadata:
  'AWS::CloudFormation::Interface':
    ParameterGroups:
    - Label:
        default: 'VPC Parameters'
      Parameters:
      - ClassB
```

* Resouce

In the Resources section you declare the AWS resources you want as part of your stack. Each resource must have a logical name unique within the template. This is the name you use elsewhere in the template as a dereference argument. Because constraints on the names of resources vary by service, all resource logical names must be alphanumeric [a-zA-Z0-9] only. You must specify the type for each resource. Type names are fixed according to those listed in [Resource Property Types](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html) reference on the AWS website. If a resource does not require any properties to be declared, you can omit the Properties section of that resource.

```yaml
Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: !Sub '10.${ClassB}.0.0/16'
      EnableDnsSupport: true
      EnableDnsHostnames: true
      InstanceTenancy: default
      Tags:
      - Key: Name
        Value: !Sub '10.${ClassB}.0.0/16'
```

* Outputs

The optional Outputs section declares output values that you can import into other stacks (to create cross-stack references), return in response (to describe stack calls), or view on the AWS CloudFormation console. For example, you can output the S3 bucket name for a stack to make the bucket easier to find.

```yaml
Outputs:
  VPC:
    Description: 'VPC.'
    Value: !Ref VPC
    Export:
      Name: !Sub '${AWS::StackName}-VPC'
```

## Image registry

ECS support both public image and private image [ECR](https://aws.amazon.com/ecr/). For simplicity, we will use public registry [nginx](https://hub.docker.com/_/nginx).
To create private image ECR is somewhat simple, you can follwing [this guide](https://www.youtube.com/watch?v=Brm21SWac-I) to create private repositories

## Create VPC and Client-sg network

### VPC network

A VPC is a virtual network inside AWS where you can isolate your workload. A VPC consists of several subnets. Each subnet is bound to an Availability Zone. A **public** subnet has a direct route to/from the Internet. As long as your service have an public IPv4/IPv6 address, they **can communicate (in and out) with the Internet**. A private subnet does not have a IPv4 route to/from the Internet but an Ipv6 route to the Internet exists. Service in **private** subnets can not be accessed from the public Internet. If you want to access the Internet from a private subnet, you need to create a NAT gateway/instance or assign an IPv6 address. You can deploy a bastion host/instance to reduce the attack surface of internal applications.

*Insert image vpc-2azs.png*

### Client Security Group network

Some data stores are integrated into the VPC, others are only accessible via the AWS API. For VPC integration, you have to create a Client Security Group stack. The stack is used as a parent stack for security group ElastiCache, Elasticsearch, and RDS. To expand communicate with the data store from a EC2 instance, you have to attach the Client Security Group to the EC2 instance. The Security Group does not have any rules, but it marks traffic. The marked traffic is then allowed to enter the data store.

**Insert image target-group.png**

With this appoarch, we can well seperace concern and security our app. Only service that subscribers to our client sg can communicate in cluster stack.

The aws vpc and client security group yaml can find in github repo, below is bash cli to run these stack

`bash aws-init-vpc.sh`

```bash
PROJECT_NAME=${PROJECT_NAME?Variable not set}
APP_ENV=${APP_ENV?Variable not set}

echo ">>>>>>>>>>>VPC for $PROJECT_NAME - $APP_ENV<<<<<<<<<<<<"
sleep 1

aws cloudformation deploy \
    --template-file aws/vpc-2azs.yml \
    --stack-name vpc-$APP_ENV-$PROJECT_NAME \
    --capabilities CAPABILITY_NAMED_IAM \
    --tags $PROJECT_NAME-$APP_ENV-cluster=vpc

echo ">>>>>>>>>>>Client for $PROJECT_NAME - $APP_ENV<<<<<<<<<<<<"
sleep 1

aws cloudformation deploy \
    --template-file aws/client-sg.yml \
    --stack-name client-$APP_ENV-$PROJECT_NAME \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameter-overrides ParentVPCStack=vpc-$APP_ENV-$PROJECT_NAME \
    --tags $PROJECT_NAME-$APP_ENV-cluster=client
```

The cloudformation stack can be display in CloudFormation and navigate to &*Stack section**. In the Events tab of the stack, the progress of creating the stack can be followed.

*Insert image progress.png*

### Outputs

These stack return:

* VPC with public and private subnets in two availability zones

* Client security group

## Add ECS cluster and load balancer

### ECS cluster

This file starts from the template_task_definition.yaml file where you will add the cluster configuration. The official AWS documentation for creating the ECS Cluster will be the main source of information.

```yaml
Resources:
  Cluster:
    Type: 'AWS::ECS::Cluster'
    Properties:
      CapacityProviders: 
      - FARGATE
      - FARGATE_SPOT # Enable fargate spot provider
      ClusterName: !Sub ${AWS::StackName}
...
```

### Application load balancer (ALB)

The ALB configuration requires the configuration of the subnets to use. We will define ALB with enable both http/https and logging

```yaml
  LoadBalancer:
    Type: 'AWS::ElasticLoadBalancingV2::LoadBalancer'
    Properties:
      IpAddressType: !If [HasLoadBalancerSchemeInternal, 'ipv4', 'dualstack']
      LoadBalancerAttributes:
      - Key: 'idle_timeout.timeout_seconds'
        Value: !Ref LoadBalancerIdleTimeout
      - Key: 'routing.http2.enabled'
        Value: 'true'
      - Key: 'access_logs.s3.enabled'
        Value: !If [HasS3Bucket, 'true', 'false'] # Enable write log ALB to s3
      ...
  LoadBalancerSecurityGroupInHttpFromWorld:
    Type: 'AWS::EC2::SecurityGroupIngress'
    Condition: HasNotAuthProxySecurityGroup
    Properties:
      GroupId: !Ref LoadBalancerSecurityGroup
      IpProtocol: tcp
      FromPort: 80 # Enable http traffic
      ToPort: 80
      CidrIp: '0.0.0.0/0'
  LoadBalancerSecurityGroupInHttpsFromWorld:
    Type: 'AWS::EC2::SecurityGroupIngress'
    Condition: HasNotAuthProxySecurityGroupAndLoadBalancerCertificateArn
    Properties:
      GroupId: !Ref LoadBalancerSecurityGroup
      IpProtocol: tcp
      FromPort: 443 # Enable https traffic
      ToPort: 443
      CidrIp: '0.0.0.0/0'
  HTTPCodeELB5XXTooHighAlarm:
    Condition: HasAlertTopic
    Type: 'AWS::CloudWatch::Alarm'
    Properties:
      AlarmDescription: 'Application load balancer returns 5XX HTTP status codes'
      Namespace: 'AWS/ApplicationELB'
      MetricName: HTTPCode_ELB_5XX_Count
      ...
  RejectedConnectionCountTooHighAlarm:
    Condition: HasAlertTopic
    Type: 'AWS::CloudWatch::Alarm'
    Properties:
      AlarmDescription: 'Application load balancer rejected connections because the load balancer had reached its maximum number of connections'
      Namespace: 'AWS/ApplicationELB'
      MetricName: RejectedConnectionCount
      ...
...
```

To start ecs cluster stack, run the below command

`aws-init-cluster.sh`

```bash
PROJECT_NAME=${PROJECT_NAME?Variable not set}
APP_ENV=${APP_ENV?Variable not set}

echo ">>>>>>>>>>>Cluster for $PROJECT_NAME - $APP_ENV<<<<<<<<<<<<"
sleep 1

aws cloudformation deploy \
    --template-file aws/cluster-fargate.yml \
    --stack-name cluster-fargate-$APP_ENV-$PROJECT_NAME \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameter-overrides ParentVPCStack=vpc-$APP_ENV-$PROJECT_NAME \
    --tags $PROJECT_NAME-$APP_ENV-cluster=ecs-cluster
```

### Outputs

These stack return:

* ECS cluster with enable both fargate spot and on-demand provider

* Dedicate load balancer for this specific cluster

* HTTP/HTTPS ALB

* Logging to s3 (*Optional*)

* CloudWatch alarm 5xx and maximum number of connections (*Optional*)

## Task definitions

Configure task and ecs service in cloudformation.
 Set the property RequiresCompatibilities to FARGATE because of course you will run the task in a Fargate cluster.

```yaml
Resources:
  TaskDefinition:
    Type: 'AWS::ECS::TaskDefinition'
    Properties:
      ContainerDefinitions:
      - Name: app
        Image: !Ref AppImage
        Command: !If [HasAppCommand, !Split [',', !Ref AppCommand], !Ref 'AWS::NoValue']
        PortMappings:
        - ContainerPort: !Ref AppPort
          Protocol: tcp
        ...
      # Define resource
      Cpu: !FindInMap [CpuMap, !Ref Cpu, Cpu]
      Memory: !FindInMap [MemoryMap, !Ref Memory, Memory]
      NetworkMode: awsvpc
      RequiresCompatibilities: [FARGATE]
      ...
  Service:
    DependsOn: LoadBalancerListenerRule
    Type: 'AWS::ECS::Service'
    Properties:
      Cluster: {'Fn::ImportValue': !Sub '${ParentClusterStack}-Cluster'}
      CapacityProviderStrategy:
      - Base: 0
        CapacityProvider: FARGATE
        Weight: 2 # 2 tasks run on FARGATE
      - Base: 0
        CapacityProvider: FARGATE_SPOT
        Weight: 1 # 1 tasks run on FARGATE_SPOT
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
        DeploymentCircuitBreaker:
          Enable: true
          Rollback: true
      TaskDefinition: !Ref TaskDefinition
      HealthCheckGracePeriodSeconds: !Ref HealthCheckGracePeriod
      ...
  ScalableTarget:
    Condition: HasAutoScaling
    Type: 'AWS::ApplicationAutoScaling::ScalableTarget'
    Properties:
      MaxCapacity: !Ref MaxCapacity
      MinCapacity: !Ref MinCapacity
      ResourceId: !Sub
      - 'service/${Cluster}/${Service}'
      - Cluster: {'Fn::ImportValue': !Sub '${ParentClusterStack}-Cluster'}
        Service: !GetAtt 'Service.Name'
      RoleARN: !GetAtt 'ScalableTargetRole.Arn'
      ScalableDimension: 'ecs:service:DesiredCount'
      ServiceNamespace: ecs
```

To start ecs service, run the below command

`bash aws-init-task.sh`

```bash
PROJECT_NAME=${PROJECT_NAME?Variable not set}
APP_ENV=${APP_ENV?Variable not set}
APP_IMAGE=${APP_IMAGE?Variable not set}

echo ">>>>>>>>>>> App for $PROJECT_NAME $APP_ENV<<<<<<<<<<<<"
aws cloudformation deploy \
    --template-file aws/task_definition/app-$APP_ENV.yml \
    --stack-name app-$APP_ENV-$PROJECT_NAME \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameter-overrides ParentVPCStack=vpc-$APP_ENV-$PROJECT_NAME \
        AppEnvironment1Key=APP_ENV \
        ParentClusterStack=cluster-fargate-$APP_ENV-$PROJECT_NAME \
        ParentClientStack1=client-$APP_ENV-$PROJECT_NAME \
        AppEnvironment1Value=$APP_ENV \
        AppImage=$APP_IMAGE \
        Spot=true \
        AutoScaling=true \
        Cpu=0.25 \
        Memory=0.5 \
        DesiredCount=3 \
    --tags $PROJECT_NAME-$APP_ENV-cluster=service-$APP_IMAGE
```

### Outputs

These stack return:

* ECS task with nginx image

* Fargate task with strategy `2(on-demand): 1(on-spot)`, task running should be `n+1` for [voting policy](https://www.continuent.com/resources/blog/why-does-mysql-mariadb-cluster-require-odd-number-nodes)

* Autoscaling group and healthcheck

Navigate to the cluster-stack, retrieve the public IP in Output section and verify whether the containers can be reached and return their host IP and a welcome message

## Cleanup

Cleaning up the resources is fairly easy now. Navigate to the Stack and click the Delete button. The progress of deletion can be tracked in the Events tab. This will take some minutes, but the good news is, that no manual deletion of resources must be done. Everything which has been created with the template is automatically removed.

## Summary

The ecs cluster can be shortly describe with these step:

`Step 1`: Export all the needed enviroment

```bash
export APP_ENV=demo
export PROJECT_NAME=nginx
export APP_IMAGE=nginx:latest
```

`Step 2`: Create VPC and Client-sg stack

```bash
bash aws-init-vpc.sh
```

`Step 3`: Create ECS cluster stack

```bash
bash aws-init-cluster.sh
```

`Step 4`: Create ECS service stack

```bash
bash aws-init-task.sh
```

`Step 5`: Verify ecs stack working

*Insert task running ecs.png*

*Insert healthcheck.png*

## Conclusion

You learnt how to create a CloudFormation template which creates an ECS Fargate cluster and runs a Dockerized nginx application. Creating a stack, uploading a template, deleting the stack, etc. are powerfull tools and the complete configuration is under version control. Learning hot to create the template can take some effort. The AWS documentation provides a good resource for this, but besides that, you will find yourself googling for examples. Quite some AWS documentation is provided with examples, but some of them are not. Besides that, it is a great way for managing infrastructure.
