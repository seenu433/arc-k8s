# Azure Arc enabled Kubernetes Workshop

## Agenda
1. Create a k8s cluster using Kubeadm 
1. Onboard the cluster to Arc for k8s
1. Features walkthrough: Policies, GitOps, Monitoring
1. Private preview extensions: Extensions: OSM, Monitoring, Defender
1. Cluster Connect walkthrough


## Prerequisites

1. Share your github handle to get access to [private preview documentation](https://github.com/Azure/azure-arc-kubernetes-preview)
1. Create a kubernetes cluster using kubeadm
1. Setup for remote access

### Creating a kubernetes cluster using kubeadm

Create Resource Group, Virtual network, Subnet and Virtual Machine

```azurecli
    az group create --name k8s --location eastus
    
    ## Create a VNet and make a note of SUBNETID
    az network vnet create --name k8snet --resource-group k8s --location  eastus --address-prefixes 172.10.0.0/16 --subnet-name k8s-subnet1 --subnet-prefixes 172.10.1.0/24

    az vm create --name kube-master --resource-group k8s --location eastus --image UbuntuLTS --admin-user azureuser --generate-ssh-keys --size Standard_DS3_v2 --data-disk-sizes-gb 10 --public-ip-address-dns-name k8s-kube-master-lab --subnet <SUBNETID>

    ## Make a note the fqdns and the public IP address
```

SSH into the VM and run the script to initailize a kubernetes cluster

```bash
    ssh azureuser@<fqdns>

    vi prepare-cluster-node.sh
```

Copy and paste contents below in the vim editor

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

Save the file __ESC :wq__

Execute the script

```bash
    sudo bash ./prepare-cluster-node.sh

    sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-cert-extra-sans <fqdns>,<publicIPAddress>

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

    kubectl taint nodes --all node-role.kubernetes.io/master-

    cat $HOME/.kube/config

    ## Copy the contents to be used next
```

Exit from the Shell

### Access the cluster remotely

Navigate to the kubeconfig file on the local machine at C:\Users\USERNAME\.kube

Edit/Replace the config file

Update the config file for the IP of the API Server to the public IP of the VM

Run kubectl get nodes

The cluster is private at this point the API server is not accessible from outside the vnet

Add a rule to the NSG to allow TCP traffic on port 6443

Run kubectl get nodes

**Reference:** [Create Cluster with kubeadm](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/create-cluster-with-kubeadm.md)

## Onboard the cluster to Arc

Install CLI extensions  and register the providers. We will use the [preview extensions](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/k8s-extensions.md#prerequisites) to enable private preview features.

```azurecli
    az group create --name arc -l EastUS -o table

    az connectedk8s connect --name arc-k8s --resource-group arc

    ## Watch for the rollout of agents

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
      allowPrivilegeEscalation: false
      privileged: true
```

## Extensions

1. Install preview CLI extensions  and register the providers. (Should have been done earlier)
2. Install the Extensions  (Should have been done earlier)
3. View in Portal (https://aka.ms/azmon-containers-arc-extensions-preview)

Th extensions blade is behind a hide-key and will be visible if the portal is opened using the link provided above.

Below are the extensions:

### Monitor
Steps to deploy the [Monitor Extension](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/k8s-extensions-azure-monitor.md#create-azure-monitor-extension-instance)

### [Optional] Defender
Needs Master nodes to be designated separately. We need a multi node cluster with separate Master and Worker Nodes.

Steps to deploy the [Defender Extension](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/k8s-extensions-azure-defender.md)

[Alerts](https://docs.microsoft.com/en-us/azure/security-center/alerts-reference#alerts-containerhost) available from Defender

### [Optional] OSM
Steps to deploy the [OSM Extension](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/osm/k8s-extensions-openservicemesh.md)

## Cluster Connect

A Kubernetes Cluster onboarded to Arc will still not have direct access to the API server. Cluster feature helps in scenarios needing line of sight to the API Server.

- No Inbound connection required
- Proxies the commands to the cluster
- Service Account Token / AAD Auth

Note that the AAD option of Connect required you to have AD admin rights to grant access to the consent. You can onboard the cluster(in your AIRS) to your personal subscription with admin access.

[Steps](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/cluster-connect.md) to onboard a cluster with Connect feature

## AKS GitOps

1. [Install](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/use-gitops-in-aks-cluster.md) the cli extension and register the providers
2. Enable Add-on on the AKS cluster
3. Create a SourceControlConfiguration
4. View the GitOps blade on the Portal (https://aka.ms/AKSGitOpsPreview)
5. Save the SSH public key on the Git Repo

Note the Repo url [formats](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-connected-cluster#use-a-private-git-repository-with-ssh-and-flux-created-keys) for http/ssh and the root as user/repo
