**Prerequisites**
To start creating the free Kubernetes cluster on Oracle Cloud using Terraform you’ll need the following things in Sand Box.

1) An Oracle Cloud account
2) A compartment you want the resources to provision in – it can be the root compartment if that’s okay with you
3) A user that have permissions to access the necessary resources – can be your root user if you want to
4) Optionally an SSH key you want to use to provision your Kubernetes worker nodes and wanna make sure you have some way to access them if needed
5) OCI CLI installed
6) Terraform CLI installed
7) kubectl installed

Once these are in Place, clone this project to user home directory.

**CLI Access Configuration**

First of all, when you have OCI CLI installed on your machine, execute the following command, otherwise you need to setup the OCI CLI

oci setup config

This will prompt you all the data it’ll need to generate the proper configuration for you.
1) OCID
2) Tenancy ID
3) Region 
4) Private key lcoation

**Note:** for some reason I wasn’t able to get Terraform work with a passphrase protected private key because there’s some random issue there so I decided to go without a password.

After you’re done with the key generation, there’s one more thing we need to do. Associating the public key that was generated during setup with the user. Go back to your user in the Oracle Cloud web console, click on API keys on the left and click on Add API Key. Upload your public key’s pem file and you’re done.

You can verify that everything is configured properly by running the following command:
**
$ oci iam compartment list -c <tenancy-ocid>**

Where <tenancy-ocid> is your tenancy’s OCID. If you don’t get some authorization error but your compartments’ data, you’re good to go.

**Setting Up The Terraform Variables**
 
Now that the access is settled, let’s talk about the Terraform scripts we’re gonna use to build the free Kubernetes cluster on Oracle Cloud.

Let’s split our Terraform scripts into folders. We’ll have one to set up the OCI infrastructure and a separate one that will create the load balancers and configure a few things in the K8S cluster.

Respectively, create an oci-infra folder. In that, let’s have a variables.tf file where we’ll store the Terraform variables. And then, we’ll have the full infrastructure Terraform file, call it infra.tf. Plus, we’ll want to have some outputs for further things, so let’s create an outputs.tf too.
 
  For Variable.tf file, we need to export the 3 variables
  
$ export TF_VAR_compartment_id= "your compartment ocid"
 
$ export TF_VAR_region= "your region"
 
$ export TF_VAR_ssh_public_key= "your public key"
 
  
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
  
  Now all the configurations are in place, Lets trigger the terraform to create the cluster. Make sure you are in **oci-infra** folder
  
1) Execute a **terraform init**
2) Execute a **terraform apply**

 If you did everything right, after waiting for some time, your cluster should be ready. 
 
**Note** that the cluster nodes are relatively slow to provision because the Oracle Container Engine for Kubernetes is updating them with the proper packages and that takes some time. For me it took around 20 minutes to provision everything. You can monitor the state of the cluster nodes on the Oracle Cloud web console.
 
  And then we’ll create a kubeconfig for kubectl to access the cluster. Let’s execute the following command:

$ oci ce cluster create-kubeconfig --cluster-id <cluster OCID> --file ~/.kube/free-k8s-config --region <region> --token-version 2.0.0 --kube-endpoint PUBLIC_ENDPOINT
  
  This will create a free-k8s-config file in the ~/.kube folder that contains the keys and all the configuration for kubectl to access the cluster.

Let’s set this configuration through the KUBECONFIG environment variable for kubectl.

$ export KUBECONFIG=~/.kube/free-k8s-config
And then let’s try to list the available nodes in the cluster.

**$ kubectl get nodes**
  
And if you see the 2 nodes that the node pool has provisioned, you’re done. The cluster is ready.

**Building A Custom Application**
 
 Create a custom Node JS application along with a mongodb database, once the app is running, it will display a web page "Hello world, insert the data into databse", souce code is placed in the  **nodedocker_app**
 
 Lets build an image using docker compose to test the applicaiton local machine.
 docker-compose up
 It will create 2 containsers respectively Node JS and Mongo DB, we can access the application with localhost
 http://localhost:3000
 
 Awesome. The Docker image can be built now.
 
 
**Free Private Docker Registry**
 
 The next thing we wanna do is to create a free private Docker registry. You can for sure put everything into public registries but if you need to, you can also do it privately on Oracle Cloud.

Let’s create the registry using Terraform. Open the oci-infra/infra.tf file from the previous article and add the following:

// ...previous things are omitted for simplicity
resource "oci_artifacts_container_repository" "docker_repository" {
  compartment_id = var.compartment_id
  display_name   = "free-kubernetes-nginx"
  is_immutable = false
  is_public    = false
}
Then a quick terraform apply and your registry is ready to go.

Let’s grab the username and password for the private registry.
 
** Note:** I Have already updated the infra.tf file, so docker registry will be created as part of cluster set up.
 
 The URL for the private registry can be found here: Availability by Region. The URL solely depends on which region you put the docker registry in. In my case, it’s eu-frankfurt-1 so my URL is gonna be fra.ocir.io.
 

The next thing is the username. It’s gonna consist of 2 things. The tenancy’s Object Storage Namespace and the user that has the permissions to access the Docker registry in the account.
 
The tenancy’s Object Storage Namespace can be found on the Tenancy’s page:
 
 And at the bottom, you can find the Object Storage Namespace, it’s gonna be some random string.

Next, we should get the name of the user we’re planning to use for interacting with the docker registry. Go to Identity/Users and choose from the available users or create a new one if you wish but don’t forget to add the neessary permissions to access the registry.

The username for the registry comes together from those 2 in the form of <object-storage-namespace>/<username> so for example it could be abcdefgh/test123.

The last thing we need to do is to generate a password for programmatic authentication, so let’s go to the user’s details on the cloud console and select Auth Tokens from the left.

Click on Generate token and at the end you’ll get a password, save that.

To test the whole thing, open a terminal window and type the following:

$ docker login -u <object-storage-namespace>/<username> <docker server>
This will prompt you for the password you generated on the Auth Tokens page. If everything is set up correctly, you should see the Login succeeded message.

Note: it might take a minute for the auth token to become active so give it some time if you see Unauthorized messages from docker login
 
 Alright, now we need to make the docker access available to the Kubernetes cluster so it can download images from this private registry.

Open up your terminal window and do the following:

$ kubectl -n <namespace> create secret docker-registry free-registry-secret --docker-server=<docker-server> --docker-username='<object-storage-namespace>/<username>' --docker-password='<password>'
secret/free-registry-secret created
This will create a secret named free-registry-secret which will be the docker access for that particular namespace to our registry.
 
**Automatic Building And Deployment**
 
 Alright, the next thing we should do is to create the GitHub Action to deploy the app to the cluster upon pushing to the Git repository.

Create the file .github/workflows/cicd.yaml with the following content:
 
 This is the workflow definition which will trigger every time anything gets changed in the nodedocker_app folder and on the master branch.
 
 **Let me sum up the steps:**
1. Git Checkout
2. Docker Buildx setup
3. Installing OCI CLI
4. Installing kubectl
5. Verifying if kubectl is configured by listing all pods
6. Authenticate to Docker registry
7. Logging all the available platforms we can build Docker images for Building and uploading the image
8. Install the Helm for application deployment
