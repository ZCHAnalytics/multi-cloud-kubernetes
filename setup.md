 <img src="https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/a797bfca-62d2-44e9-b631-ff3eaaf41315" >



# Terraform workflow to:
- Provision Kubernetes clusters in both Azure and AWS environments using their respective providers
- Configure Consul federation with mesh gateways across the two clusters using the Helm provider
- Deploy microservices across the two clusters to verify federation
</p>

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/8d3a8f57-f752-41cc-8eee-d09a7e8995ea)

Prerequisites: Terraform 0.14+ installed locally, AWS CLI, the Azure CLI, kubectl

Root
|-- aks
|   |-- main.tf
|   |-- outputs.tf
|   |-- terraform.tfvars
|   |-- variables.tf
|   |-- versions.tf
|-- consul
|   |-- dc1.yaml
|   |-- dc2.yaml
|   |-- main.tf
|   |-- variablesproxy_defaults.tf
|   |-- versions.tf
|-- counting-service
|   |-- Subfolder1
|-- eks
|   |-- kubernetes.tf
|   |-- main.tf
|   |-- outputs.tf
|   |-- variables.tf
|   |-- versions.tf

# Provision clusters
## Provision an EKS Cluster
AWS: Provision the worker nodes for the cluster.
The configuration creates a designated network to place the EKS resource. It also provisions an EKS cluster and an autoscaling group of workers to run the workloads.

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/b2ac8f0e-8a40-4415-854d-40c222a0d0d8)

## Provision an AKS Cluster
- Create an Active Directory service principal account to authenticate the Azure provider.
```
az login
az ad sp create-for-rbac --skip-assignment
az aks get-versions --location westus2 # first attempt at applying terraform apply resulted in error, saying that the kubernetes version is not available in the chosen location, so this command helps to find available versions 
```
![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/e96df87a-1a25-41f4-9354-37adac836b51)

## Consul federation configuration
To allow services across the two clusters to communicate, set up Consul datacenters in both Kubernetes clusters then federate them. 
Consul enables a secure multi-cloud infrastructure by creating a secure service mesh that facilitates encrypted communication between your services.
The configuration will deploy Consul to both  EKS and AKS clusters using the Consul Helm chart and designate the EKS Consul datacenter as the primary. 
It also uses the Kubernetes provider to share the federation secret across the clusters and to provision the ProxyDefaults configuration as a custom resource.

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/6ef722a3-9813-4cb1-b918-4c50678b214e)

## The consul_dc1 Helm release

The consul_dc1 resource deploys a Consul datacenter to the EKS cluster using the hashicorp/consul Helm chart. This is the primary Consul datacenter in this tutorial and the one in which you generate the federation secret.

Warning: This Consul configuration disables ACLs and does not use gossip encryption. Do not use it in production environments.

### The eks_federation_secret data source
The Kubernetes eks_federation_secret data source accesses the federation secret created in the primary Consul datacenter in your EKS cluster. The depends_on meta-argument explicitly defines the dependency.

### The aks_federation_secret resource
The Kuberenetes aks_federation_secret resource uses the Kubernetes provider authenticated against your AKS cluster to load the federation secret from your EKS cluster into your AKS cluster. The resource dependency is implicit since it references the eks_federation_secret data source from the EKS cluster.

### The consul_dc2 Helm release
The consul_dc2 Helm release deploys the secondary Consul datacenter to your AKS cluster. The configuration is similar to that of the primary, but rather than generating the federation secret, the configuration for the secondary datacenter references values from the consul-federation secret you imported.

This configuration leverages Terraform's resource dependency graph to ensure that your resources are created in proper order.

### The eks_proxy_defaults and aks_proxy_defaults Kubernetes manifests
The eks_proxy_defaults and aks_proxy_defaults resources use Kubernetes custom resources to create Consul ProxyDefaults for each datacenter. The Consul Helm release creates the CRDs the ProxyDefaults use, so you must provision these resources as a separate step.

use the kubernetes_manifest resource to manage Kubernetes custom resources.

Note: The kubernetes_manifest resource type is in beta. You should not use it in production environments.

### Configure kubectl
Once Terraform provisions both clusters, use kubectl to verify their respective Consul datacenters.
```
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name) --alias eks
```
```
az aks get-credentials --resource-group $(terraform output -raw resource_group_name) --name $(terraform output -raw kubernetes_cluster_name) --context aks
```
### Deploy Consul and configure cluster federation
Once Terraform finishes provisioning both clusters, apply the configuration to:
- Deploy the primary Consul datacenter and proxy defaults to EKS
- Load the federation secret into the AKS cluster
- Deploy the secondary Consul cluster

Initialize Terraform directory to download the providers.

### Deploy ProxyDefaults
Apply the configuration to create the ProxyDefaults in both clusters. 

### Verify cluster federation
List the pods in the default namespace in the EKS cluster to confirm that the Consul pods are running.
```
kubectl get pods --context eks
```
```
kubectl get proxydefaults --context eks
```

list the pods in the default namespace in the AKS namespace to confirm that the Consul pods are running.
```
kubectl get pods --context aks
```
NAME                                           READY   STATUS    RESTARTS   AGE

verify that Terraform applied the proxy defaults.
```
kubectl get proxydefaults --context aks
```
Warning: The ProxyDefaults may show as False due to an open issue in the Kubernetes provider kubernetes_manifest resource. Do not use this configuration in production.
Finally, verify that the clusters are federated by listing the servers in Consul's Wide Area Network (WAN).
```
 kubectl exec statefulset/consul-server --context aks -- consul members -wan
```
The Consul members list includes nodes from both the dc1 and dc2 centers, confirming datacenter federation.

Using a single Terraform invocation, we created resources in multiple cloud providers. 
The configuration deployed Helm releases and managed Kubernetes resources across two clusters, each in a different cloud. 
Terraform's provider aliasing helped configure multiple instances of each provider, and Terraform's dependency graph enforced the appropriate order for resource creation.

### Deploy an application
Deploy a two-tier application that communicates across the Kubernetes clusters using the federated Consul datacenters. The application counts how many times a user accesses it and consists of:
- a backend service named counting that increments a counter, deployed to your AKS cluster
- a frontend service named dashboard that calls the counting service and displays the counter value, deployed to the EKS cluster

Similarly to the Consul configuration from the previous section, this configuration uses the terraform_remote_state data source to access the contents of the AKS and EKS workspace's state files.
The configuration passes the attributes to the Kubernetes providers to authenticate against each cluster.
The counting service consists of a pod that Terraform deploys to your AKS cluster by referencing the aks aliased provider.

The dashboard service consists of a pod and service that Terraform deploys to the EKS cluster by referencing the eks aliased provider. 
This pod has the consul.hashicorp.com/connect-service-upstreams annotation to configure the service dependency on the counting service in the other Consul datacenter.

terraform init

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. 
Include this file in the version control repository so that Terraform can guarantee to make the same selections by default for running "terraform init" in the future.

 terraform apply

Once Terraform deploys the resources to your clusters, visit the dashboard to verify the configuration. Enable port forwarding for the EKS cluster's dashboard pod to access the dashboard locally.

 kubectl port-forward dashboard 9002:9002 --context eks

Navigate to http://localhost:9002/ in your browser. The dashboard should display a positive number, confirming that the dashboard service can reach the counting service.

Counting service dashboard shows positive number to confirm Consul Federation


# Clean up resources
Run terraform destroy separately in each of the project directory: aks, eks, consul, counting 
In  counting-service directory, run terraform destroy to destroy the microservice Kubernetes resources.
In consul Destroy Consul resources
Run terraform destroy to destroy the Consul Helm and Kubernetes resources.
Destroy Kubernetes clusters - Run terraform destroy to deprovision the AKS cluster. 
