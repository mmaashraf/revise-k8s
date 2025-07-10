This project is to:


🌠 Launch and connect to an EC2 instance.

✨ Create your very own Kubernetes cluster.

☁️ Monitor cluster creation with CloudFormation.

🔑 Access your cluster using an IAM access entry.

💎 (Secret Mission) Test the resilience of your Kubernetes cluster.



Amazon EKS is not AWS Free Tier eligible. AWS charges $0.10 for every hour my EKS cluster is running. To minimise costs, I will delete all my resources today.

1. Lauch ec2

compute section


    Port 22 is the default port for SSH (Secure Shell) connections, which is what we use to connect securely to our EC2 instance.

    Back when we set up our EC2 instance, we used the default security group which allows SSH access from any IP address. This opens your instance to the entire internet, which makes it more vulnerable to attacks.

    💡 Is using the default security group security best practice?
    We wouldn't do this for production environments or long-term use, but for now the default security group makes it easier to use EC2 Instance Connect.

    If we don't use the default security group, we'd have to manually edit our instance's inbound rules to let in Instance Connect's range of IP addresses, which is different depending on your AWS Region.


challenge: 
    Try editing your security group settings to make it more secure (while letting you use EC2 Instance Connect).


    added IP address range that restrict machines that can ssh into ec2 instance


    Once created, i see a message like


Success
    Successfully initiated launch of instance (i-0d1c9c53d62294f16)

        Initializing requests
        Succeeded
        Creating security groups
        Succeeded
        Creating security group rules
        Succeeded
        Launch initiation
        Succeeded


Done -
    Launch an EC2 instance with free tier eligible settings and the default security group.
        Added some security by reducing the IP range


- Connect to your EC2 instance using EC2 Instance Connect

Connect  Info

Connect to an instance using the browser-based client.

user: ec2-user

I see details
    ec2-k8s-project
    i-0d1c9c53d62294f16
        
    Running
    t2.micro

    us-east-2c

I see a browser client that's lauched to run cmds

3. Launch EKS Cluster


    Now for the main event – creating our Kubernetes cluster with Amazon EKS!

    Once you've created containers (e.g. using Docker), Kubernetes helps keep them running smoothly by automatically handling tasks like load balancing, scaling, and restarting containers if they fail. It doesn’t build the containers—that’s Docker’s job—but it’s great for keeping them running without you having to do it manually.


    💡 What is Amazon EKS?
    While Kubernetes makes it easier to work with containers, Amazon EKS makes it easier to work with Kubernetes itself!

    Setting up Kubernetes from scratch can be quite a time consuming and complex thing to do, because you'd need to configure Kubernetes' networking, scaling, and security settings on your own. Amazon EKS handles all of these set up tasks for you and helps you integrate Kubernetes with other AWS services.

    eksctl is a tool specifically for working with EKS in the command line 

    eksctl create cluster \
    --name nextwork-eks-cluster \
    --nodegroup-name nextwork-nodegroup \
    --node-type t2.micro \
    --nodes 3 \
    --nodes-min 1 \
    --nodes-max 3 \
    --version 1.31 \
    --region [YOUR-REGION]


    💡 What is eksctl?
    eksctl is an official AWS tool for managing Amazon EKS clusters in your terminal. It's much, much easier to use compared to setting up a Kubernetes cluster using the AWS CLI!

    If we were using the AWS CLI, we'd have to create a lot of other components manually before getting to deploy a cluster.


    OUTPUT: Command was not found.

    Solution: install the tool

    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv -v /tmp/eksctl /usr/local/bin


    I see an Error:

    ~ $ eksctl create cluster --name nextwork-eks-cluster --nodegroup-name nextwork-nodegroup --node-type t2.micro --nodes 3 --nodes-min 1 --nodes-max 3 --version 1.31 --region us-east-2c
    Error: checking AWS STS access – cannot get role ARN for current session: operation error STS: GetCallerIdentity, https response error StatusCode: 0, RequestID: , request send failed, Post "https://sts.us-east-2c.amazonaws.com/": dial tcp: lookup sts.us-east-2c.amazonaws.com on 127.0.0.11:53: no such host


    Seems like a role issues

    We’ll need to set up an IAM role that lets your EC2 instance communicate with services like EKS to create the cluster.

    Create an IAM role for your EC2 instance with AdministrorAccess, and attach it to the instance. You can do this in the AWS Management Console.



    💡 Is granting AdministratorAccess best practice?
    Great question! Granting AdministratorAccess is powerful but not ideal for long-term use. We’re using it now because we don’t know all the services that will be involved in the project yet.

    Granting the most powerful permissions now means we avoid running into permission issues that could slow us down. Once we know which services your EC2 instance really needs for this project to work, we can dial back to more specific, secure settings.

    Solution Architects typically plan security settings before they build anything, aiming to follow the principle of least privilege—only giving the exact permissions needed for the job.


    Created role
        - entity is ec2
        - gave the adminaccess permission

    For the ec2 instance,
        - updated security to have the new IAM role


    Since i generate keys are the begging, i can used them to ssh into the machine as the ec2-user into

    e.g. ssh -i "ec2-lauch-k8.pem" ec2-user@ec2-18-220-224-245.us-east-2.compute.amazonaws.com

    ah.. i was passing in the availablity zone, not the region. there is an extra char


    this command will:

        Set up an EKS cluster named nextwork-eks-cluster
        Launch a node group called nextwork-nodegroup.
        Use t2.micro EC2 instances as nodes.
        Start your node group with 3 nodes and automatically scale between 1 (minimum) and 3 nodes (maximum) based on demand.
        Use Kubernetes version 1.31 for the cluster setup.

4. Track How AWS Creates Your EKS Cluster

💡 What is CloudFormation?
CloudFormation is AWS’s service for setting up infrastructure as code. You write a template describing the resources you need (like an instruction manual), and CloudFormation handles creating and configuring those resources.


💡 Why are we in CloudFormation?
eksctl actually uses CloudFormation under the hood to create your EKS cluster. 


what actually happened?

    When you ran the eksctl create cluster command, eksctl sets up a CloudFormation stack to automate the creation of all the necessary resources for the EKS cluster.

💡 What are these Events about?
    The Events tab gives you a timeline of each action CloudFormation is taking to set up your resources. It’s a live update of what’s happening, which can be helpful for tracking progress or identifying any issues that come up during creation.



💡 What are these resources?
    You might notice in your Resources tab all sorts of networking resources - VPC, subnets, route tables, security groups, NAT gateways and internet gateways!

    These resources set up a private, secure network for your containers to connect with each other and the internet while keeping your app private. 


The node group is a group of nodes that share the same configuration, like instance type and scaling settings. In this case, our node group has three nodes, which means there are three EC2 instances that will run containers like a single unit.


----------

Delete Nodes (and watch them regenerate)

💡 What will happen once you delete nodes in a node group?
When you delete nodes in a node group, Kubernetes automatically detects the change and launches new nodes to keep the cluster running at its desired state (more on this in a second).


💡 What do desired state, minimum size, and maximum size mean?
The desired size is the number of nodes you want running in your node group.

The minimum size is the minimum number of nodes your group needs to have to keep your app available at all times (even in low-demand periods).

The maximum size is the maximum number of nodes that you'll allow inside this group. This also lets EKS scale up your node group from the desired size in high-demand periods.



DELETE all Resouces
cloud formations

ec2