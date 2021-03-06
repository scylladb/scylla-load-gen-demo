# Scylla load test demo
This repository contains CloudFormation templates and supporting files to run an automated load test on a Scylla Cloud cluster for demo purposes.

(Click for YouTube)

[![Demo video](assets/demo.gif)](https://www.youtube.com/watch?v=M8eDQVNEeN8)

There are two CloudFormation templates:
1. VPC stack - VPC, subnet, routing table etc.
2. Workers stack - EC2 instances running the load test
In order to run the load test in a secure manner private VPC IP addresses are uses which means that VPC peering is required. VPC stack is separate so you can run it, peer it with Scylla cloud and then run the workers stack as needed to generate load.

# prerequisite
- [AWS CLI](https://aws.amazon.com/cli/)
- [Scylla Cloud](https://cloud.scylladb.com) Account
- AWS Account and credentials

# Running
## Scylla cloud
Run a new Scylla cloud cluster for the load test:
![](assets/scylla-cloud-new-cluster.png)

Select the node configuration suitable to the load - you may want to consult the [pricing and sizing calculator](https://www.scylladb.com/pricing/#pricing-calc) on the Scylla website.
![](assets/scylla-cloud-instance-config.png)

While the cluster is bootstrapping, run the CloudFormation VPC stack as follows.

## VPC stack
The latest version of the templates is published to S3 and can be used instead of the version from this repo:
- [VPC stack](https://scylla-devrel.s3.us-east-2.amazonaws.com/assets/autostress/stress-vpc.cfn.yaml)
- [Workers stack](https://scylla-devrel.s3.us-east-2.amazonaws.com/assets/autostress/stress-workers.cfn.yaml)

There are two alternatives from running a CloudFomration template:
- From the CLI
- From AWS web console

For each, you can use a template from S3 or this repo. 

Using the CLI:
```
aws cloudformation create-stack --stack-name test-stress-vpc --template-body file://./stress-vpc.cfn.yaml
```
Or using the S3 URL:
```
aws cloudformation create-stack --stack-name test-stress-vpc --template-url https://scylla-devrel.s3.us-east-2.amazonaws.com/assets/autostress/stress-vpc.cfn.yaml
```

Using the AWS web console, go to [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation/home) and run a new stack:
![New VPC stack](assets/cloudformation-vpc-run.png)

When done, the stack will have outputs with the needed parameters for VPC peering with Scylla cloud: VPC ID, Account ID, VPC CIDR block.
Using the web console:
![VPC stack outputs](assets/cloudformation-vpc-outputs.png)

## VPC peering
Configure the peering as explained in Scylla cloud (also, see the [docs](https://docs.scylladb.com/scylla-cloud/cloud-setup/vpc-peering/)). Scylla cloud will initiate a VPC peering request you will need to accept - this can be done using the CLI:
```
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-XXXXXXXXXXXXXXXX
```

Or using the web console:
![VPC peering accept](assets/vpc-peering-accept.png)

After accepting the VPC peering request you will need to add routes to Scylla cloud VPC through the peering connection. The load test VPC has a routing table named `ScyllaStressTest` to which the routes should be added. Using the CLI:
```
aws ec2 create-route --route-table-id rtb-XXXXXXXXXXXXXXXXXX --vpc-peering-connection-id pcx-XXXXXXXXXXXXXXXXXX --destination-cidr-block 172.31.0.0/16
```

Using the web console:
![VPC routes add](assets/vpc-peering-routes-add.png)

You are now ready to run the workers stack which will apply the load.
## Workers stack

There are two alternatives from running a CloudFomration template for Workers stack:
- From the CLI
- From AWS web console

Using the CLI (note the use of a [parameters file](https://aws.amazon.com/blogs/devops/passing-parameters-to-cloudformation-stacks-with-the-aws-cli-and-powershell/)):
```
aws cloudformation create-stack --stack-name test-scylla-workers --template-body file://./stress-workers.cfn.yaml --parameters file://./workers-params.json
```
Or using the S3 URL:
```
aws cloudformation create-stack --stack-name test-stress-workers --template-url https://scylla-devrel.s3.us-east-2.amazonaws.com/assets/autostress/stress-workers.cfn.yaml --parameters file://./workers-params.json
```
The stack parameters can be provided on command line as `--parameters ParameterKey=name,ParameterValue=xxxxx ParameterKey=name2,ParameterValue=yyyyy` or using parameter file such as
```json
[
    {
        "ParameterKey": "VPCStackName",
        "ParameterValue": "test-stress-vpc"
    },
    {
        "ParameterKey": "ScyllaNodes",
        "ParameterValue": "172.31.0.11,172.31.1.11,172.31.2.11"
    },
    {
        "ParameterKey": "ScyllaPassword",
        "ParameterValue": "xxxxxxx"
    }
]
```
Update `workers-params.json` and make sure to modify the parameters to the proper values for your environment.
Use the **PRIVATE** IPs of the Scylla cluster, available in Clutser::Connect::Instructions.


Using the web console:
![CloudFormation run workers stack](assets/cloudformation-workers-run.png)

The stack requires the following parameters - other parameters are optional:
- `VPCStackName` - The name you used for the VPC stack, e.g. `test-stress-vpc`
- `ScyllaNodes` - A list of Scylla cloud nodes private IPs, copy pasted from the Scylla cloud "Connect" tab
- `ScyllaPassword` - The password of the `scylla` user, copy pasted from the Scylla cloud "Connect" tab

The stack will create 4 EC2 instances which will automatically start generating load on the cluster.
