**Prerequisites**
To start creating the free Kubernetes cluster on Oracle Cloud using Terraform you’ll need the following things:

An Oracle Cloud account
A compartment you want the resources to provision in – it can be the root compartment if that’s okay with you
A user that have permissions to access the necessary resources – can be your root user if you want to
Optionally an SSH key you want to use to provision your Kubernetes worker nodes and wanna make sure you have some way to access them if needed
OCI CLI installed
Terraform CLI installed
kubectl installed

**CLI Access Configuration**

First of all, when you have OCI CLI installed on your machine, execute the following command, otherwise you need to setup the OCI CLI

oci setup config

This will prompt you all the data it’ll need to generate the proper configuration for you.
1) OCID
2) Tenancy ID
3) Region 
4) Private key lcoation
Note: for some reason I wasn’t able to get Terraform work with a passphrase protected private key because there’s some random issue there so I decided to go without a password.

After you’re done with the key generation, there’s one more thing we need to do. Associating the public key that was generated during setup with the user. Go back to your user in the Oracle Cloud web console, click on API keys on the left and click on Add API Key. Upload your public key’s pem file and you’re done.

You can verify that everything is configured properly by running the following command:

$ oci iam compartment list -c <tenancy-ocid>

Where <tenancy-ocid> is your tenancy’s OCID. If you don’t get some authorization error but your compartments’ data, you’re good to go.

**Setting Up The Terraform Variables**
 
Now that the access is settled, let’s talk about the Terraform scripts we’re gonna use to build the free Kubernetes cluster on Oracle Cloud.

Let’s split our Terraform scripts into folders. We’ll have one to set up the OCI infrastructure and a separate one that will create the load balancers and configure a few things in the K8S cluster.

Respectively, create an oci-infra folder. In that, let’s have a variables.tf file where we’ll store the Terraform variables. And then, we’ll have the full infrastructure Terraform file, call it infra.tf. Plus, we’ll want to have some outputs for further things, so let’s create an outputs.tf too.
 
  For Variable.tf file, we need to export the 3 variables
  
$ export TF_VAR_compartment_id=<your compartment ocid>
$ export TF_VAR_region=<your region>
$ export TF_VAR_ssh_public_key=<your public key>
  
**Network Resources For The Free Kubernetes Cluster**
 
  The first thing we need to tell Terraform is which provider to use. There’s one provider for Oracle Cloud called oci.

Into the infra.tf file:

provider "oci" {
  region = var.region
}
This will configure the oci provider for the specified region which is coming from the variable configured.
Then, let’s create the Virtual Cloud Network (VCN) and corresponding configurations in the infra.tf file
 
**Getting Access To The Kubernetes Cluster**
  Lets create one more file outputs.tf file.
  
  Noe all the configurations are in place, Lets trigger the terraform to create the cluster.
  terraform init
  terraform apply
  
 If you did everything right, after waiting for some time, your cluster should be ready. Note that the cluster nodes are relatively slow to provision because the Oracle Container Engine for Kubernetes is updating them with the proper packages and that takes some time. For me it took around 20 minutes to provision everything. You can monitor the state of the cluster nodes on the Oracle Cloud web console.
 
  And then we’ll create a kubeconfig for kubectl to access the cluster. Let’s execute the following command:

$ oci ce cluster create-kubeconfig --cluster-id <cluster OCID> --file ~/.kube/free-k8s-config --region <region> --token-version 2.0.0 --kube-endpoint PUBLIC_ENDPOINT
  
  This will create a free-k8s-config file in the ~/.kube folder that contains the keys and all the configuration for kubectl to access the cluster.

Let’s set this configuration through the KUBECONFIG environment variable for kubectl.

$ export KUBECONFIG=~/.kube/free-k8s-config
And then let’s try to list the available nodes in the cluster.

$ kubectl get nodes
  
And if you see the 2 nodes that the node pool has provisioned, you’re done. The cluster is ready.
  
  
