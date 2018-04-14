# Understanding the EC2 Autoheal pattern

## Introduction

When working with AWS you can easily create virtual machine instances with EC2 to run your applications and for more cloud native applications, you can leverage the Autoscale group to deploy redundant copies across multiple availability zones to handle instance or AZ failure as well as easily add more capacity.

However, with more traditional application workloads such as Legacy / COTS applications that are often heavily stateful and/or tightly coupled to network addresses for configuration or even licencing requirements, they too can benefit from the Autoscale group construct.

For these types of application topologies we still have a requirement to ensure that in the event of a failure of the underlying EC2 instance, that it is replaced with a new one and returned to service after ensuring that any external AWS resources that the instance requires (such as an EBS volume containing application data or an EIP tied to an applications licence) are correctly associated with the new instance prior to application start.

It is also critical to note that this needs to be done without human intervention to ensure that the application is returned to service as quickly as possible.

To acheive this outcome, comes the EC2 'Autoheal' pattern. The goal of this pattern to to provide resilience to an application that cannot autoscale, and to look to make it as robust in a single AZ as possible.

As part of this demonstration my goal was to show a simple implementation of the pattern, alongside providing the outcome in a single, self-contained Cloudformation Template that could be deployed into any AWS account for educational purposes.

_Please note that is a simple autoheal demonstration, and although it can be solved in a number of ways, this is the way I chose to do it, and it is in no way, the only, or best way. It is also worth noting that in a production environment, this would not reside in a single template, for ease of education and a personal challenge, I decided to do it this way._

## Autoheal Topology

In this demonstration topology the following is achieved via cloudformation

- Create a basic VPC with a subnet, security groups, internet gateways and routes to support the EC2 instance.
- Create a EC2 Instance via an Autoscale group 
- Create an EBS volume that is associated with the launched EC2 instance and mounted at /data
- Create an EIP that is associated with the instance once it is online
- Ensure that the deployment is done in a way in which AWS security best practices are leveraged
- Deploy all associated AWS resources that are leveraged to return the instance to the same stage in the event of an EC2 failure.


## Autoheal Solution

### Deploying the Autoheal template into your account
Detailed instructions about how to deploy this template into your AWS account are out of scope for this document, however, I suggest you start at [Getting started with AWS cloud formation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/GettingStarted.html) which should get you going quickly and, then in the console or via the CLI deploy this template into your account and tweak the various template parameters based on your region and AZ.

_Don't forget to create your SSH keypair for SSH access and reference it by name in the stack parameters else your stack will fail to create_

### Understanding the solution components and resources
When the template is succesfully deployed into your account, a number of resources are created to implement the solution. The following sections detail each of these resources broken down in various layers of the stack and what their function is.

#### VPC and Network layer
The resources below create a simple VPC for deploying the autohealing instance into.
This gives you a single public subnet with all the associated routing and security groups to allow ingress via SSH to the instance once it's launched later in the deployment.

Automation components will be used further down the stack to ensure that this EIP is re-associated with the instance once it is brought online.

Cloudformation Resources in this layer:

    - myVPC
    - PublicSubnetA
    - myInternetGateway
    - AttachGateway
    - PublicRouteTable
    - PublicSubnetRoute
    - PublicSubnetARouteTableAssociation
    - AutohealSecurityGroup
    - AutoHealEIP

#### Storage layer
The resource in this layer is solely an EBS volume that we will later mount at /data on the EC2 instance.

Automation components will be used further down the stack to ensure that this volume is re-associated with the instance once it is online.

Cloudformation Resources in this layer:

    - AutohealVolume


#### Compute and automation layer
The resources in this layer are where most of the action happens, so lets walk through them one by one;

##### The Autoheal Group
This resource will be an autoscale group of 1 once the stack is fully created, however we initially create it as 0 to ensure that it does not start launching EC2 instances before the entire stack has completed provisioning all the other resources.

##### The Autoheal Group Launch Configuration
The launch configuration contains all the information required to launch the EC2 instance. 

Specifically the AMI, the Instance size, the SSH key ID and the userdata logic to ensure that the /data filesystem on the EBS volume created via the 'AutohealVolume' is mounted accordingly.

The userdata logic takes the following action on launch of an EC2 instance by the autoscale group.

In a loop of 6 times with a 10 second sleep between them; 
- Check if the block device /dev/xvdb (by default) is available on the EC2 instance yet. This represents the EBS volume that is created in the stack by the 'AutohealVolume' resource
- Once the device is available, check the device to determine if a filesystem is present on it
    - If there is no filesystem on it, create an ext4 filesystem on it
    - If there is a filesystem on it, continue to the mount phase
- Once the mount phase is reached, mount the filesystem at the /data mountpoint and exit 0

##### The Device Attachment layer
The device attachment layer contains all the resources to automate the reattachment of the EBS volume and EIP in the event that a new EC2 instance is launched due to a failure of the previous instance.

The componets in this layer are;
- An Autoscaling group lifecycle hook that puts the autoscaling group into a Pending:Wait state when a new instance is launched.
- An EventRule that is triggered for the Autoheal Autoscaling group when a launch event occurs, when this is triggered it executes the DeviceAttachment lambda
- The DeviceAttachment lambda is a block of Python code that take the following actions
    - Parses the event data that is sent to the lambda that contains the new EC2 instance ID
    - Polls the EC2 instance to ensure that it is in a ready state and sleeps 5 seconds until it is
    - When the instance is in a ready state, using the boto3 library
        - It attaches the EBS volume to the new EC2 instance
        - It attaches the EIP volume to the new EC2 instance
        - Completes the lifecycle hook action, returning the autoscaling group to an Inservice state where it will continue to boot and execute the userdata logic to mount the filesystem.
    - The DeviceAttachment lambda executes with the DeviceAttachmentRole which contains the permissions for it to associate the EIP and EBS volume with the instance
    - The DeviceAttachment lambda also requires AWS::Lambda::Permission  to allow it to execute which is created using the PermissionForEventsToInvokeLambda resource

Cloudformation Resources in this layer:

    - AutohealGroup
    - AutohealLaunchConfig
    - DeviceAttachmentHook
    - EventRule
    - PermissionForEventsToInvokeLambda
    - DeviceAttachment
    - DeviceAttachmentRole


#### Orchestration Layer
The Orchestration layer is unique to this solution as I set myself the challenge of getting the whole example into a single cloudformation stack.

Executed last using the DependsOn, the AutohealScaleup resource is a custom cloudformation resource that sets the Autoheal Autoscale group's desired instance count from 0 to 1

We do this because when an autoscalegroup is created, it immediately starts launching instances, as such, doing it immediately on autoscale group creation results in the instance being launched before the stack has entirely finished provisioning all it's resources, as such, the instance comes online prior to the lifecycle hooks, events and lambdas being provisioned, resulting in the EC2 instance that is launched not having the EBS volume or the EIP associated with it.

The custom cloudformation resource triggers the AutohealScaleup lambda function, which executes using the DeviceAttachmentRole which has permissions to set the Autoheal AutoscaleGroup's instance count to 1. 

Cloudformation Resources in this layer:

    - AutohealScaleup

## Seeing the Autoheal in action

When the stack is deployed, you can login to it with your keys and explore the system. When you are ready to see the autohealing in action, terminate the instance in the console and then when the new instance is in service, you can login to it via the same EIP where you should find the EBS volume mounted again at /data.

The EIP for the instance is returned to you as a stack output to make it easy for you to identify the correct IP address.

## Additional comments, thoughts,  learnings and enhancements
When building this example, I learnt a number of things, I figured that for educational purposes they were worth mentioning below:

### IAM, Lambdas and using CFN REF's
When building this solution, my goal was to ensure that the security policy in place to was as tailored to the resources in this stack as tightly as possible and use least privilege as much as possible.  

As such, I did the following;
- Avoid using EC2 instance roles. I wanted to have all the logic for the device attachment happen externally to the instance and avoid having the EC2 instance require any IAM access to AWS resouces. If the EC2 instance itself has no role, it can't be abused.
- Use cloudformation REF's to ensure that IAM policy would only apply to resources for that particular deployment and stack. In doing this, any issue in the lambda code that may exist would not be an issue because the IAM policy was hard coded to specific autoscale groups and EBS volumes. I did encounter some issues with some resources not allowing granular IAM policies, which is quite well known, however I handled this in other ways. (Note that in the real world i'd have each lambda have individual roles, rather than have a single one that contains all action polcies for further IAM seperation)
- Use cloudformation REF's to build lambdas that are hard coded to only operate on specific volumes, EIPs and other AWS resources. If you look at the DeviceAttachment lambda, the volume ID and EIP address are hard coded into the code of the lambda using environment variables. As such, it makes them extremely difficult to abuse because they reference minimal external input and are combined with strict resource bound IAM policy (where possible).  

### Adding healthchecks using ELBs
In this example, the Autoscale group only uses EC2 healthchecks.

In a non-lab environment, I would suggest adding an ELB healthcheck that validates the state of the application itself and terminates the instance if it is found to be unhealthy,replacing it with a new one in a known good state. Even if you don't use the ELB to route traffic through to your application due to static IP address requirements, it provides the ability to do deeper levels of application health validation than the default EC2 checks.

### Using Autoscale group metrics for single instance benefits
As the [Autoscalegroup metrics are now free](https://aws.amazon.com/about-aws/whats-new/2016/08/free-auto-scaling-group-metrics-with-graphs/), using the autoheal pattern allows you to now get more effective visbility of instances that have terminated and relaunched using tools such as Datadog.

### Using higher level languages and patterns to deliver this outcome
As you can see, this single outcome is weighing in at quite a bit of code. There are many other tools out there to assist with automating the generation of CFN and AWS resources, it would be much more effective to look to use them to automate this outcome when you need to have more than a single Autoheal instance to be deployed.
Some examples are [Terraform](https://www.terraform.io/), [Sparkleformation](https://www.sparkleformation.io/), [CFN DSL](https://github.com/cfndsl/cfndsl) and more

### Referencing Stack parameters in Userdata bash script
To prevent the bash variables in the userdata being attempted to be substituted / interpolated by CFN, I had to add an eclaimation mark to them in the launch config bash code.
You can read more about this at [Sub - Intrinsic function reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html)

### Deleting the stack when you are done
Sometimes, when deleting this sample stack it fails due to the EBS volume. Just attempt to delete the stack again and it will delete properly.

## References and further reading

### Additional Autohealing EC2 instance examples and references

- [How to setup a self healing EC2 instance? (Server Fault)](https://serverfault.com/questions/674752/how-to-setup-a-self-healing-ec2-instance)
- [AWS - Auto-Healing Public EC2 Instances](https://salsadigital.com.au/news/aws-auto-healing-public-ec2-instances)
- [KOPS deploys all single instances in ASG's for autohealing instances](https://kubernetes.io/docs/getting-started-guides/kops/)
- [AWS Tips I Wish I'd Known Before I Started](https://wblinks.com/notes/aws-tips-i-wish-id-known-before-i-started/)

### Useful AWS development documentation 

- [A great blog post on developing CFN custom resources](https://blog.jayway.com/2015/07/04/extending-cloudformation-with-lambda-backed-custom-resources/)
- [cfn-response for AWS Custom resources documentation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html)
- [Using Boto3 EC2 Waiters instead of sleeps](http://boto3.readthedocs.io/en/latest/reference/services/ec2.html#EC2.Waiter.InstanceRunning)
- [IAM Example for handling Volume attachment with EC2](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_ec2_volumes-instance.html)
- [Wordpress multi-AZ template that I pulled the AMI lookups from](https://s3-ap-southeast-2.amazonaws.com/cloudformation-templates-ap-southeast-2/WordPress_Multi_AZ.template)
