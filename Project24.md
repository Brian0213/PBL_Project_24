# BUILDING ELASTIC KUBERNETES SERVICE (EKS) WITH TERRAFORM

Since project 21 and 22, you have had some fragmented experience around kubernetes bootstraping and deployment of containerised applications. This project seeks to solidify your skills by focusing more on real world set up.

1. You will use Terraform to create a Kubernetes EKS cluster and dynamically add scalable worker nodes
2. You will deploy multiple applications using HELM
3. You will experience more kubernetes objects and how to use them with Helm. Such as Dynamic provisioning of volumes to make pods stateful
4. You will improve upon your CI/CD skills with Jenkins.

In Project 21, you created a k8s cluster from Ground-Up. That was quite painful, but very necessary to help you master kubernetes. Going forward, you will not have to do that. Even in the real world, you will hardly ever have to do that. given that cloud providers such as AWS have managed services for kubernetes, they have done all the hard work, and with a few API calls to AWS, you can have a production grade cluster ready to go in minutes. Therefore, in this project, you begin by focusing on EKS, and how to get it up and running using Terraform. Before moving on to other things.

## Building EKS with Terraform

At this point you already have some Terraform experience. So, you have some work to do. But, you can get started with the steps below. If you have terraform code from Project 16, simply update the code and include EKS starting from number 6 below. Otherwise, follow the steps from number 1.

1. Open up a new directory on your laptop, and name it eks:

[EKS Dir.](./Screenshots/eks-dir.png)

2. Use AWS CLI to create an S3 bucket by running this code:

`aws s3api create-bucket --bucket eks-terraform-buck --region eu-west-1`

3. Create a file – backend.tf Task for you, ensure the backend is configured for remote state in S3

[Backend Tf](./Screenshots/backend-tf.png)

4. Create a file – network.tf and provision Elastic IP for Nat Gateway, VPC, Private and public subnets. 
Create VPC using the official AWS module.

[Network Tf](./Screenshots/network-tf1.png)

[Network Tf](./Screenshots/network-tf2.png)

Note: The tags added to the subnets is very important. The Kubernetes Cloud Controller Manager (cloud-controller-manager) and AWS Load Balancer Controller (aws-load-balancer-controller) needs to identify the cluster’s. To do that, it querries the cluster’s subnets by using the tags as a filter.

For public and private subnets that use load balancer resources: each subnet must be tagged:

`Key: kubernetes.io/cluster/cluster-name
Value: shared`

For private subnets that use internal load balancer resources: each subnet must be tagged:

`Key: kubernetes.io/role/internal-elb
Value: 1`

For public subnets that use internal load balancer resources: each subnet must be tagged

`Key: kubernetes.io/role/elb
Value: 1`

5. Create a file – variables.tf

[Variables Tf](./Screenshots/variables-tf.png)

6. Create a file – data.tf – This will pull the available AZs for use.

[Data Tf](./Screenshots/data-tf.png)

### BUILDING ELASTIC KUBERNETES SERVICE (EKS) WITH TERRAFORM – PART 2

7. Create a file – eks.tf and provision EKS cluster (Create the file only if you are not using your existing Terraform code. Otherwise you can simply append it to the main.tf from your existing code) Read more about this module from the official documentation here – Reading it will help you understand more about the rich features of the module.

Creating the eks.tf file that will provision EKS cluster:

[Eks Tf](./Screenshots/eks-tf.png)

8. Create a file – locals.tf to create local variables. Terraform does not allow assigning variable to variables. There is good reasons for that to avoid repeating your code unecessarily. So a terraform way to achieve this would be to use locals so that your code can be kept DRY:

Creating the locals.tf file for local variables because Terraform does not allow assigning variable to variables:

[Locals Tf](./Screenshots/locals-tf1.png)

[Locals Tf](./Screenshots/locals-tf2.png)

9. Add more variables to the variables.tf file:

[Updated Variables Tf](./Screenshots/update-variables-tf.png)

10. Create a file – terraform.tfvars to set values for variables.

[Terraform Tfvars](./Screenshots/terraform-tfvars.png)

11. Create file – provider.tf

[Provides Tf](./Screenshots/providers-tf.png)

12. Create a file – variables.tfvars to set values for variables.

[Variables Tfvars](./Screenshots/variables-tfvars.png)

13. Run terraform init

`terraform init`

[Terraform Init](./Screenshots/terraform-init1.png)

[Terraform Init](./Screenshots/terraform-init2.png)

14. Run Terraform plan – Your plan should have an output

`terraform plan`

[Terraform Plan](./Screenshots/terraform-plan.png)

15. Run Terraform apply

`terraform apply`

This will begin to create cloud resources, and fail at some point with the error

[Terraform Apply Error](./Screenshots/terraform-apply-error.png)

That is because for us to connect to the cluster using the kubeconfig, Terraform needs to be able to connect and set the credentials correctly.

Let fix the problem in the next section.

### FIXING THE ERROR

To fix this problem

Append to the file data.tf

[Data Tf Append](./Screenshots/append-datatf.png)

Append to the file providers.tf:

[Providers Tf Append](./Screenshots/append-providerstf.png)

Run the init and plan again – This time you will see:

[Terraform Init Apply](./Screenshots/terra-init-apply.png)

16. Create kubeconfig file using awscli:

`aws eks update-kubeconfig --name tooling-app-eks --region eu-west-1 --kubeconfig kubeconfig`

[Terraform Init Apply](./Screenshots/kubeconfig.png)