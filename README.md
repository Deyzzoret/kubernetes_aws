# kubernetes_aws  

This repository contains a detailed example of how to launch a kubernetes cluster in AWS cloud provider. 
For this, there are two methods. The latter more direct than the first one, but the first one helps us to understand how things workd under the hood.

First method AWS Cli

As we are goning to interact with our AWS account, we must install AWS Cli in our machine. Another solution for this would be to run a docker container with all the needed packages in it.
For this guide, i would use the aws-cli docker image.  

# Run Amazon CLI
docker run -it --rm -v ${PWD}:/work -w /work --entrypoint /bin/sh amazon/aws-cli:2.0.17 

Once we are inside the container, we must install jq package so that we can work with json files , as must of the AWS files have this format.

yum install jq gzip nano tar git  

Now we must configurte our AWS credentials inside the container, please checkout the documentation for this.

https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html  

Checkout how to do a multi region deployment for AWS.

Choose your AWS region, we need to know where our cluster will be deployed.  
You can use **Amazon Elastic Kubernetes Service (Amazon EKS)** to run applications in multiple AWS Regions¹. You can connect applications hosted in Amazon EKS clusters in multiple AWS Regions over a private network using inter-region VPC peering and employ a data replication strategy using AWS Database and Storage services that fit your recovery point (RPO) and recovery time (RTO) objectives¹.

(1) Operating a multi-regional stateless application using Amazon EKS. https://aws.amazon.com/blogs/containers/operating-a-multi-regional-stateless-application-using-amazon-eks/ Accessed 24/6/2023.
(2) Scale applications using multi-Region Amazon EKS and Amazon Aurora .... https://aws.amazon.com/blogs/database/part-1-scale-applications-using-multi-region-amazon-eks-and-amazon-aurora-global-database/ 
(3) Run an active-active multi-region Kubernetes application with AppMesh .... https://aws.amazon.com/blogs/containers/run-an-active-active-multi-region-kubernetes-application-with-appmesh-and-eks/ 

In my case, i would like to create my cluster in Paris, so the code associated to that region is eu-west-3


Know, we have to log in into the aws container and configure it.
Type  
aws configure

For this step, you will have to provided the generated credentials and then specify the region you will be working on. 

IAM Role / Policies 

Before starting to run a brand new cluster, it is important to notice than AWS uses IAM roles to perform operations on behalf of us. This IAM roles must have attach a set of rights that they will have overt a service. So in this case, as Kubernetes at certain point will need to spin up some containes, loadbalancers and other resources. It would be benifitial for us to have this role. 

We can manually create a new IAM role, but in this case we will create it using AWS Cli.
The command to do so is : 
# create our role for EKS
We always have to assume a policy when creating a role. In this case, we are using the policy 
role_arn=$(aws iam create-role --role-name eks-role --assume-role-policy-document file://assume-policy.json | jq .Role.Arn | sed s/\"//g) 
Now we have to attach a role policy to this brand new role. AWS already has a big bunch of predifined policies.

Attach policy to role  

aws iam attach-role-policy --role-name eks-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# create the cluster VPC  
This will create a cluodformation file defining a VPC divided in 3 subnets.

curl https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-05-08/amazon-eks-vpc-sample.yaml -o vpc.yaml  

This will create our VPC resource in our AWS accout.  
For more information about the generate file, please checkout the documentation.  

I would say that stack means set of configuration/resource create from a file. So that we can identify them.

aws cloudformation deploy --template-file vpc.yaml --stack-name getting-started-eks

Now with our VPC deployed we can deploy our cluster. Before that we must get the information related to the resources just created (ids). This will be helpful while creating the cluster because we will specify in which network our nodes should be deployed.  
To get all the informatio about the resources just created associated to the stack name defined at the end of the command, execute the following command:  
# grab your stack details 
aws cloudformation list-stack-resources --stack-name getting-started-eks > stack.json  


At this point we already have all the needed components to create our cluster. We must specify the subnets id where our nodes will spin up. 
Attach the previous generated role id.  
endpointPublicAccess set to true will allow us to access our cluster.

 aws eks create-cluster  --name first-eks  --role-arn $role_arn  --resources-vpc-config subnetIds=subnet-03c4ce8f6e1d2183b,subnet-03e8a21ba97754bc4,subnet-059a6545787b4f584,securityGroupIds=sg-06f7a352439a554ce,endpointPublicAccess=true,endpointPrivateAccess=false  

 In case you want more information about your current cluster. 

aws eks list-clusters
aws eks describe-cluster --name first-eks 

Get the configi file related to our cluster.  
For that, execute the following command: 
This command will generate the config file in ~/.kube/config .

aws eks update-kubeconfig --name first-eks --region eu-west-3

#grab the config if you want it
cp ~/.kube/config . 

Install kubectl in the current aws cli container

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl  

chmod +x ./kubectl && mv ./kubectl /usr/local/bin/kubectl  

In case the previous version of kubectl present some problem, try with 

curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.6/bin/linux/amd64/kubectl  




Second method EKS Cli
