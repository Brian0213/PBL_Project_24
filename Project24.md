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

[Providers Tf](./Screenshots/providers-tf.png)

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

#### DEPLOY APPLICATIONS WITH HELM

In Project 22, you experienced the use of manifest files to define and deploy resources like pods, deployments, and services into Kubernetes cluster. Here, you will do the same thing except that it will not be passed through kubectl. In the real world, Helm is the most popular tool used to deploy resources into kubernetes. That is because it has a rich set of features that allows deployments to be packaged as a unit. Rather than have multiple YAML files managed individually – which can quickly become messy.

A Helm chart is a definition of the resources that are required to run an application in Kubernetes. Instead of having to think about all of the various deployments/services/volumes/configmaps/ etc that make up your application, you can use a command like:

`helm install stable/mysql`

Fetching the script:$ `curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3`
Changing the permission of the script:$ `chmod 700 get_helm.sh`
Executing the script:$ `./get_helm.sh`

[Helm Install](./Screenshots/helm-install.png)

Lets begin to gradually walk through how to use Helm (Credit – https://andrewlock.net/series/deploying-asp-net-core-applications-to-kubernetes/)
Parameterising YAML manifests using Helm templates
Let’s consider that our Tooling app have been Dockerised into an image called tooling-app, and that you wish to deploy with Kubernetes. Without helm, you would create the YAML manifests defining the deployment, service, and ingress, and apply them to your Kubernetes cluster using kubectl apply. Initially, your application is version 1, and so the Docker image is tagged as tooling-app:1.0.0. A simple deployment manifest might look something like the following:

[Helm Install](./Screenshots/helm-install.png)

Now lets imagine you produce another version of your app, version 1.1.0. How do you deploy that? Assuming nothing needs to be changed with the service or ingress, it may be as simple as copying the deployment manifest and replacing the image defined in the spec section. You would then re-apply this manifest to the cluster, and the deployment would be updated, performing a rolling-update as I described in my first post.

The main problem with this is that all of the values specific to your application – the labels and the image names etc – are mixed up with the "mechanical" definition of the manifest.

Helm tackles this by splitting the configuration of a chart out from its basic definition. For example, instead of baking the name of your app or the specific container image into the manifest, you can provide those when you install the chart into the cluster.

For example, a simple templated version of the previous deployment might look like the following:

[Helm Install](./Screenshots/helm-install.png)

This example demonstrates a number of features of Helm templates:

The template is based on YAML, with {{ }} mustache syntax defining dynamic sections.
Helm provides various variables that are populated at install time. For example, the {{.Release.Name}} allows you to change the name of the resource at runtime by using the release name. Installing a Helm chart creates a release (this is a Helm concept rather than a Kubernetes concept).
You can define helper methods in external files. The {{template "name"}} call gets a safe name for the app, given the name of the Helm chart (but which can be overridden). By using helper functions, you can reduce the duplication of static values (like tooling-app), and hopefully reduce the risk of typos.

You can manually provide configuration at runtime. The {{.Values.image.name}} value for example is taken from a set of default values, or from values provided when you call helm install. There are many different ways to provide the configuration values needed to install a chart using Helm. Typically, you would use two approaches:

A values.yaml file that is part of the chart itself. This typically provides default values for the configuration, as well as serving as documentation for the various configuration values.

When providing configuration on the command line, you can either supply a file of configuration values using the -f flag. We will see a lot more on this later on.

Now lets setup Helm and begin to use it.

According to the official documentation here, there are different options to installing Helm. But we will build the source code to create the binary.

1. Download the tar.gz file from the project’s Github release page. Or simply use wget to download version 3.6.3 directly

`wget https://github.com/helm/helm/archive/refs/tags/v3.6.3.tar.gz`

2. Unpack the tar.gz file

`tar -zxvf v3.6.3.tar.gz`

3. cd into the unpacked directory:

`cd helm-3.6.3`

4. Build the source code using make utility:

`make build`

If you do not have make installed or for any other reason, you cannot install the tool, simply use the official documentation here for other options.

5. Helm binary will be in the bin folder. Simply move it to the bin directory on your system. You cna check other tools to know where that is. fOr example, check where pwd utility is being called from by running which pwd. Assuming the output is /usr/local/bin. You can move the helm binary there.

`sudo mv bin/helm /usr/local/bin/`

6. Check that Helm is installed:

`helm version`

[Helm Install](./Screenshots/helm-version.png)

##### DEPLOY JENKINS WITH HELM

Before we begin to develop our own helm charts, lets make use of publicly available charts to deploy all the tools that we need.

One of the amazing things about helm is the fact that you can deploy applications that are already packaged from a public helm repository directly with very minimal configuration. An example is Jenkins.

1. Visit Artifact Hub to find packaged applications as Helm Charts

2. Search for Jenkins

3. Add the repository to helm so that you can easily download and deploy:

`helm repo add jenkins https://charts.jenkins.io`

[Jenkins Add To Helm](./Screenshots/jenkins-add-helm.png)

4. Update helm repo:

`helm repo update`

[Helm Update](./Screenshots/helm-update.png)

5. Install the chart

`helm install myjenkins jenkins/jenkins --kubeconfig kubeconfig`

[Helm Jenkins Sync](./Screenshots/helm-install-jenkins.png)

6. Check the Helm deployment:

Running some commands

`helm ls --kubeconfig kubeconfig`

[Helm Jenkins Sync](./Screenshots/helm-ls-kubeconfig.png)

Check Pods:

`kubectl get pods --kubeconfig kubeconfig`




Deploying Artifactory With Helm:

`helm repo add jfrog https://charts.jfrog.io`

`helm repo update`

[Helm Artifactory](./Screenshots/helm-artifactory.png)

Installing the chart:

`helm upgrade --install artifactory --namespace artifactory jfrog/artifactory`

[Install Artifactory Chart](./Screenshots/install-artifactory.png)

Deploying Hashicorp Vault With Helm:

- Adding the Hashicorp's repository to helm:

`helm repo add hashicorp https://helm.releases.hashicorp.com`

- Updating helm repo:

`helm repo update`

- Installing the chart:

`helm install vault hashicorp/vault`

[Install Hashicorp Chart](./Screenshots/install-hashicorp.png)

- Inspecting the installation:

`alias k=kubectl`

`k get pod`

`k get svc`

[Install Hashicorp Chart](./Screenshots/install-output.png)



Deploying Prometheus With Helm:

- Adding the prometheus's repository to helm:

`helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

`helm repo update`

[Install Prometheus](./Screenshots/helm-prometheus.png)

`helm install myprometheus prometheus-community/prometheus`

[Install Prometheus](./Screenshots/prometheus1.png)

[Install Prometheus](./Screenshots/prometheus2.png)

Inspecting the installation shows that there are various pods and services created

Port forwarding to access prometheus for alert manager from the UI:



