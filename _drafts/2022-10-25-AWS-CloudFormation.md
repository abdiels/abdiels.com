---
layout: post
title: "AWS CloudFormation"
subtitle: "automating infrastructure."
background: '/img/posts/aws-lift-and-shift/post-header.png'
---

**TODO: Insert "Executive Summary" Point to the github page where the scripts are create a readme.md that gives instructions as to the parameters needed to run the code.**

Everything should be automated and nothing should be done manually more than once.  Cloud computing has made it easier to setup complete infrastructure and applications.  You can write scripts to create your whole data center with the click of a button.  You can stand up anything you need and destroy it the same day.  It is now very common, but pretty amazing.
Even if you are simply prototyping you can stand up a separate network with all the necessary subnets and security around it as you wish and then you can setup your machines or your containers to run your applications.  Everything should be scripted and automated.  This world plays real well with us lazy developers that don't like to do the same thing twice.  Usually we like the challenge of solving problems over the mundane manually repeated tasks.
This post is going to show a set of AWS CloudFormation scripts that will generate a network, a database and an application.  I have chosen to stand up a Wordpress blog since it was something I was recently looking into.
I was inspired by this article: https://aws.amazon.com/blogs/containers/running-wordpress-amazon-ecs-fargate-ecs/ on the AWS blog.  The author shows how to setup a Wordpress blog using AWS CLI and some CloudFormation.  I have taken the same idea, but turned it all into CloudFormation.  I implemented the database using Amazon Aurora Serverless.  The complete architecture looks like this:

**TODO: Insert a picture of the full architecture.**

## The Network

The previous architecture describes a Amazon Virtual Private Cloud (Amazon VPC) instantiated on an AWS region.  The VPC contains two subnets, each on different Availibity Zones.  In production or even prototyping, there is no need to create the network for each application.  There migth be reasons that one may want to keep different VPCs within the account.  I don't intend to go into VPC design or AWS account organization, but rather show an example where everything is created.
I am also not going to cover most of the reasons for the architectural desicions on the present architecture.  I will mention the components created by the network CloudFormation. 
As mentioned above we have:
- A Region which is a big geographical zone that is independent from other big netwoks other big geographical zone.  When I say big, I mean of continenatal size.
- Amazon Virtual Private Cloud (Amazon VPC) **INSERT LINK** which is a virtual network you define.
- Two Availibility Zones.  Availibility Zones (AZ) are isolated locations within a Region.
- Four Subnets. Subnets are a range of IP addresses in your VPC.  One of these subnets is public, which means it can be hit from outside our VPC network.  It is open to the public as its name implies.  Then there is a private subnet which is not accesible from outside of the VPC network.  Notice that there are two Subnets per AZ.
- Internet Gateway. The Internet Gateway allows communication between the VPC and the internet.
- A NAT Gateway. Network Address Translation (NAT) gateway is NAT service which allows instances in the private subnet to connect outside the VPC, but external sources cannot initiate a connection to the private instances.
There are other components created by this CloudFormation template such as an EFS drive and its corresponding mount targets.
The CloudFormation looks like this:

```
---
  AWSTemplateFormatVersion: "2010-09-09"

  Description: "Creates a VPC with Managed NAT, a RDS MySQL instance, an ALB, and an EFS filesystem)"
  Parameters:
    VPCName:
      Description: The name of the VPC being created.
      Type: String
      Default: "Wordpress on Fargate base infrastructure"

  Mappings:
    SubnetConfig:
      VPC:
        CIDR: "10.0.0.0/16"
      Public0:
        CIDR: "10.0.0.0/24"
      Public1:
        CIDR: "10.0.1.0/24"
      Private0:
        CIDR: "10.0.2.0/24"
      Private1:
        CIDR: "10.0.3.0/24"
```
Here we are taking a parameter for the Name of the VPC we will be creating and defining the CIDR ranges for the VPC and its subnets. This is a good resource to learn more about CIDRs:  **TODO: Insert Resource for CIDR definition**
The next section is the Resources section which defines all the resources to be created by the CloudFormation.  There you are going to see the VPC being created and all the information needed to create it.  Then there is the definition of all the Subnets.  After the VPC and subnets have been created the Internet Gateway is created and linked to the VPC.  Then it creates a Route Table in which all the routes in and out of the network will be defined. Notice that now that we are going to make associations of items in the CloudFormation, we need to denote which items the resrouces being created depend on as CloudFormation executes all the resources mostly simultaniously and not in sequence.  For exmample in this code:

```
    PublicRoute:
      Type: "AWS::EC2::Route"
      DependsOn: "GatewayToInternet"
      Properties:
        RouteTableId:
          Ref: "PublicRouteTable"
        DestinationCidrBlock: "0.0.0.0/0"
        GatewayId:
          Ref: "InternetGateway"
```
We are defining a public route. The route depends on a resource called GatewayToInternet:

```
    GatewayToInternet:
      Type: "AWS::EC2::VPCGatewayAttachment"
      Properties:
        VpcId:
          Ref: "VPC"
        InternetGatewayId:
          Ref: "InternetGateway"
```
Which is attaching the InternetGateway to the VPC.  That needs to happend in order to add a reference to the InternetGateway on the route table.



 ![Mailbox structure differences](/img/posts/aws-lift-and-shift/mailbox-layout.png)

