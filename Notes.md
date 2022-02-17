# Infra setup Terraform -Gcloud

This step uses the infra repository to create a Kubernetes cluster on which your workload will run, with its configuration being stored in code.

This repository contains nothing in the main branch, and a choice of branches depending on the cloud provider you want to choose.

The best-tested branch is the Google Cloud Provider (AKS) branch, which we cover here.

The code itself consists of four terraform files:

* connections.tf
defines the connection to AKS
* kubernetes.tf
defines the configuration of a Kubernetes cluster
* output.tf
defines the output of the terraform module
* vars.tf
variable definitions for the module
To set this up for your own purposes:

* Check out the AKS branch of your fork of the code
* Set up a Cloud account and project
* Log into Cloud on the command line:
************
* Update components in case they have updated since gcloud install:
gcloud components update
Set the project name
gcloud config set project <AKS PROJECT NAME>
Enable the AKS container APIs
gcloud services enable container.googleapis.com
Add a terraform.tfvars file that sets the following items:
cluster_name
Name you give your cluster
linux_admin_password
Password for the hosts in your cluster
AKS_project_name
The ID of your Google Cloud project
AKS_project_region
The region in which the cluster should be located, default is us-west-1
node_locations
Zones in which nodes should be placed, default is ["us-west1-b","us-west1-c"]
cluster_cp_location
Zone for control plane, default is us-west1-a
Run terraform init
Run terraform plan
Run terraform apply
Get kubectl credentials from Google, eg:
gcloud container clusters get-credentials <CLUSTER NAME> --zone <CLUSTER CP LOCATION>
Check you have access by running kubectl cluster-info
Create the gitops-example namespace
kubectl create namespace gitops-example
If all goes to plan, you have set up a kubernetes cluster on which you can run your workload, and you are ready to install FluxCD. But before you do that, you need to set up the secrets required across the repositories to make all the repos and deployments work together.
