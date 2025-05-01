## CloudFormation Stack Documentation

## What It Does

- **Automatically Accepts Shared Resources:**  
  The stack includes a small Lambda function that automatically accepts an invitation.

- **Establishes a Secure Connection:**  
  It creates a secure network link (VPC endpoint) to connect your Kubernetes metrics to SKIE’s service.

- **Opens the Right Communication Port:**  
  A security rule is set up to allow traffic on port 3000, ensuring the data can be sent and received smoothly.

## How It Works in Simple Terms

1. **Automatic Sharing:**  
   The Lambda function takes care of accepting the data-sharing invitation so you don’t have to do it manually.

2. **Pausing Until Ready:**  
   The process waits until the Lambda function finishes its job. Once confirmed, it continues with the setup.

3. **Setting Communication Rules:**  
   The stack creates a rule that permits data to pass through on port 3000, ensuring everything works correctly.

4. **Creating the Secure Link:**  
   Finally, it establishes a secure connection for the metrics service within your existing network, ensuring that data is shared safely and efficiently.

This automated setup makes it easier for your team by handling all the technical work needed to securely share Kubernetes metrics, letting you focus on improving and optimizing how you use your resources.

### How to Deploy

To deploy this stack, you need to provide the following parameters:

- **VpcId**: The ID of the VPC where the endpoint and security group will be created.
- **SubnetIds**: Comma-separated list of subnet IDs for the endpoint (one per AZ).

Example deployment command using AWS CLI:
```sh
aws cloudformation create-stack --stack-name skie-endpoint-creator-stack --template-body file:///path/to/skie-endpoint-creator-stack.yml --parameters ParameterKey=VpcId,ParameterValue=vpc-12345678 ParameterKey=SubnetIds,ParameterValue=subnet-12345678,subnet-87654321
```

### How to get the private DNS to use in the skie-collector Helm chart
```sh
aws ec2 describe-vpc-endpoint-associations   --filters "Name=tag:Name,Values=skie-k8s-metrics-endpoint"  --query "VpcEndpointAssociations[0].DnsEntry.DnsName"  --region us-east-1
```
