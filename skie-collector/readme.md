# SKIE Kubernetes Collector

The SKIE Kubernetes Collector is a tool that helps organizations saving costs and understand the usage of their Kubernetes environments. It is packaged as a Helm chart, making it simple to deploy in your clusters.

## What It Does

- **Collects Data:** The collector gathers important only metrics from your Kubernetes resources and nodes.
- **Sends Information:** The collected data is sent securely to SKIE’s observability platform, where it can be reviewed and analyzed.
- **Easy Deployment:** Packaged as a Helm chart, it’s easy for your team to install and run in your existing Kubernetes environment.

## How It Helps Your Business

- **Cost Savings:** By optimizing resource allocation, you can reduce unnecessary spending and lower operational costs.
- **Efficiency Boost:** The recommendations help maximize the performance of your applications without over-provisioning resources.
- **Ease of Use:** Designed for your technical and non-technical team, the tool does the heavy lifting by gathering and analyzing data, letting you focus on business decisions.


## Getting Started

With a few simple commands using Helm. The setup is straightforward and integrates seamlessly with your current Kubernetes deployments.

1. **Submit Your AWS Account ID:**  
   To get started, please send your AWS account ID to the SKIE team. This enables us to share necessary resources with your account.

2. **Deploy the CloudFormation Stack ( In case you want to use VPC Private Endpoint):**
   
   While the public endpoint is ready-to-go, configuring a Private VPC Endpoint offers advantages:
   
   #### Lower data transfer costs and traffic isolation
     
    - Traffic between your VPC and our service stays within the AWS network, reducing cross-AZ or internet egress fees.

    - No traffic ever traverses the public internet.

   Once your AWS account ID is shared, Skie team will provide a CloudFormation template so you can deploy the stack. This stack sets up the required AWS infrastructure for the private communication between your K8s and Skie platform.
   [CloudFormation Documentation](cloudformation/readme.md)

   **Want to use the default public endpoint? Just go directly to step 3.**

3. **Deploy the Helm Chart:**
   After the CloudFormation stack is successfully deployed, your team can install the advisor in your Kubernetes cluster using Helm with a few simple commands.
   [Helm Chart Documentation](helm-chart/readme.md)


## Support

For more information or help with deployment, please contact SKIE support or visit our documentation portal.
