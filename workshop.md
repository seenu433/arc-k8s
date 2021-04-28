# Azure Arc enabled Kubernetes Workshop

## Agenda
1. Create a k8s cluster using Kubeadm 
1. Onboard the cluster to Arc for k8s
1. Features walkthrough: Policies, GitOps, Monitoring
1. Preview extensions: Extensions: OSM, Monitoring, Defender
1. Cluster Connect walkthrough


## Prerequisites

1. [Install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)/[Update](https://docs.microsoft.com/en-us/cli/azure/update-azure-cli) Azcli, Install [kubectl](https://kubernetes.io/docs/tasks/tools/), [Install](https://helm.sh/docs/intro/install/)/[Upgrade](https://helm.sh/blog/migrate-from-helm-v2-to-helm-v3/) to Helm3
2. Create a kubernetes cluster using kubeadm
3. Setup for remote access

### Creating a kubernetes cluster using kubeadm

Create Resource Group, Virtual network, Subnet and Virtual Machine

_Note: These commands can be executed in win command prompt_

```azurecli

## Create a resource group
az group create --name k8s --location eastus
    
## Create a VNet and make a note of SUBNETID
az network vnet create --name k8snet --resource-group k8s --location  eastus --address-prefixes 172.10.0.0/16 --subnet-name k8s-subnet1 --subnet-prefixes 172.10.1.0/24

## Create a VM and make a note the fqdns and the public IP address
## Update the placeholders for ALIAS and SUBNETID
az vm create --name kube-master --resource-group k8s --location eastus --image UbuntuLTS --admin-user azureuser --generate-ssh-keys --size Standard_DS3_v2 --data-disk-sizes-gb 10 --public-ip-address-dns-name k8s-kube-master-lab-<ALIAS> --subnet <SUBNETID>

```

SSH into the VM and run the script to initailize a kubernetes cluster

```bash
## Login to the VM. Use the fqdns or the Public IP address
ssh azureuser@<fqdns>

## Create a file on the VM
vi prepare-cluster-node.sh

## Press INSERT key to get into the insert mode
```

Copy (Select content below and mouse right click) and paste (mouse right click) contents below in the vim editor

```bash
#!/bin/bash

echo "Installing Docker..."
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io

echo "Configuring Docker..."

sudo cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker

echo "Installing Kubernetes components..."

sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add 
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Save the file __ESC key and :wq__

Execute the script

```bash
## Run the script to prepare the node for k8s installation
sudo bash ./prepare-cluster-node.sh

## Initiate the kubernetes cluster. Update the placeholders for fqdns and public IP
## Make a note of the Kube Join command from the output if we wish to add nodes later
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-cert-extra-sans <fqdns>,<publicIPAddress>

## Copy the kubeconfig file to .kube folder for kubectl access on the VM
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

## Install the network plugin
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

## remove the master taint as we are currently using a single node cluster
kubectl taint nodes --all node-role.kubernetes.io/master-

cat $HOME/.kube/config

## Copy the contents from the cat above to be used next (using mouse right click)
```

Exit from the Shell

### Access the cluster remotely

* Navigate to the config file on the local machine at C:\Users\\[USERNAME]\\.kube\\ take a backup of the file. Once the workshop is done, you can go back to this file

* Open the kube config file, replace the entire content with the content copied from the kube config file in the VM

* Update the kube config file, and replace the IP (172.x.x.x) with the public IP of the VM

* kubectl get nodes (it will time out)

* The cluster is private at this point the API server is not accessible from outside the vnet

* Add a rule to the NSG (Inbound security rules) to the kube-master VM to allow TCP traffic on port 6443
  * Destination port ranges: 6443
  * Protocol: TCP
  * Name: Port_6443

* kubectl get nodes (One kube-master node should be displayed)

**Reference:** [Create Cluster with kubeadm](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/create-cluster-with-kubeadm.md)

## Onboard the cluster to Arc

Install CLI extensions  and register the providers. 

```azurecli
## Add the azcli extensions
az extension add --name connectedk8s
az extension add --name k8s-extension

## Register the providers
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.ExtendedLocation

## Monitor the registration process. Registration may take up to 10 minutes.
az provider show -n Microsoft.Kubernetes -o table
az provider show -n Microsoft.KubernetesConfiguration -o table
az provider show -n Microsoft.ExtendedLocation -o table
```

```azurecli
## Create a resource group to hold the Arc enabled Kubernetes resource
az group create --name arc -l EastUS -o table

## Connect the Kubernetes cluster to Azure Arc
az connectedk8s connect --name arc-k8s --resource-group arc

## Open another Command Prompt window and watch for the rollout of agents. Eventually you should have 8 agents in the azure-arc namespace.
kubectl get po -A -w
```

Review the [purpose](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-agent-architecture) of each of the agents

You should now have a Azure Arc resources in the resource group.

**Reference:** [Quickstart Cluster Connect](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster)

## GitOps

1. Create a [SourceControlConfiguration](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-connected-cluster)
2. View the GitOps blade on the Arc resource in the Portal

**Other scenarios:**
* [CICD with GitOps](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-gitops-ci-cd)
* Try SSH with Private Repos / Azure Repos
* [Deploying Helm with GitOps](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/use-gitops-with-helm)

## Policy
Refer [Install Azure Policy for Arc](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes?toc=/azure/azure-arc/kubernetes/toc.yml#install-azure-policy-add-on-for-azure-arc-enabled-kubernetes)

Observer the policy pod in _kube-system_ and gatekeeper pods in _gatekeeper-system_ namespace

Assign a policy from the policy blade (Serach for _Kubernetes_ and assign the _Kubernetes cluster should not allow privileged containers_)

It takes a while to get the constraints pushed to the cluster. Run _kubectl get constraints_ to observe the policies pushed to the cluster as gatekeeper constraints

Test the policy by starting a privileged pod

```yml
apiVersion: v1
kind: Pod
metadata:
  name: bad-nginx
  labels:
    app: bad-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: true
      privileged: true
```
You will get error message as follows:

"Error from server ([denied by azurepolicy-container-no-privilege-1842c1ecbe6bf4a645ac] Privileged container is not allowed: nginx, securityContext: {"privileged": true, "allowPrivilegeEscalation": true}): error when creating "***Privileged Pod.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [denied by azurepolicy-container-no-privilege-1842c1ecbe6bf4a645ac] Privileged container is not allowed: nginx, securityContext: {"privileged": true, "allowPrivilegeEscalation": true}"


## Extensions

1. Install CLI extensions  and register the providers. (Should have been done earlier)
2. Install the Arc Extension
3. View in Extension Blade of the Arc resource in the Portal

Below are the extensions:

### Monitor
Steps to deploy the [Monitor Extension](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-enable-arc-enabled-clusters?toc=/azure/azure-arc/kubernetes/toc.json)

1. Create a Log Analytics Workspace
2. Deploy the Monitor extension 

```
az k8s-extension create --name azuremonitor-containers --cluster-name arc-k8s --resource-group arc --cluster-type connectedClusters --extension-type Microsoft.AzureMonitor.Containers --configuration-settings logAnalyticsWorkspaceResourceID=<armResourceIdOfExistingWorkspace>
```

### Defender

Steps to deploy the [Defender Extension](https://docs.microsoft.com/en-us/azure/security-center/defender-for-kubernetes-azure-arc?toc=%2Fazure%2Fazure-arc%2Fkubernetes%2Ftoc.json&tabs=k8s-deploy-asc%2Ck8s-verify-asc%2Ck8s-remove-arc)

```
az k8s-extension create --name microsoft.azuredefender.kubernetes --cluster-type connectedClusters --cluster-name arc-k8s --resource-group arc --extension-type microsoft.azuredefender.kubernetes --configuration-settings logAnalyticsWorkspaceResourceID=<armResourceIdOfExistingWorkspace>
```

[Alerts](https://docs.microsoft.com/en-us/azure/security-center/alerts-reference#alerts-containerhost) available from Defender

### [Optional] OSM
Steps to deploy the [OSM Extension](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/osm/k8s-extensions-openservicemesh.md)

## Cluster Connect

A Kubernetes Cluster onboarded to Arc will still not have direct access to the API server. Cluster Connect feature helps in scenarios needing line of sight to the API Server.

- No Inbound connection required
- Proxies the commands to the cluster
- Service Account Token / AAD Auth

```
## Enable the Cluster Connect on any Azure Arc enabled Kubernetes cluster by running the following command
az connectedk8s enable-features --features cluster-connect -n arc-k8s -g arc

## Create a service principal, this needs to be unique
az ad sp create-for-rbac --name arc-k8s-user-[YOUR ALIAS]
## Make a note of the appId, password and tenant

## Get the Object Id for the SP
az ad sp show --id APP_ID --query objectId -o tsv

## Create the cluster-admin role binding
kubectl create clusterrolebinding admin-user-binding --clusterrole cluster-admin --user=<objectId>
```

Disable the inbound rule on the cluster VM (kube-master) NSG by changing the rule to DENY for port 6443

```
## Login using hte service principal
az login --service-principal --username APP_ID --password PASSWORD --tenant TENANT_ID

## Run the proxy to initiate connectivity to the Arc resource
az connectedk8s proxy -n arc-k8s -g arc

## Open a new CMD window and run kubectl commands
kubectl get po -A
```

Notice that we are able to execute kubectl commands without connectivity to the API server.

[Steps](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/cluster-connect) to onboard a cluster with Connect feature

## AKS GitOps

1. [Install](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/use-gitops-in-aks-cluster.md) the cli extension and register the providers
2. Enable Add-on on the AKS cluster
3. Create a SourceControlConfiguration
4. View the GitOps blade on the Portal (https://aka.ms/AKSGitOpsPreview)
5. Save the SSH public key on the Git Repo

Note the Repo url [formats](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-connected-cluster#use-a-private-git-repository-with-ssh-and-flux-created-keys) for http/ssh and the root as user/repo
