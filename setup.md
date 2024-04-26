![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/7d4da6cd-6ca4-4643-abe5-0d7e0d21e22d)

# Terraform workflow
- Provision Kubernetes clusters in both Azure and AWS environments using their respective providers
- Configure Consul federation with mesh gateways across the two clusters using the Helm provider
- Deploy microservices across the two clusters to verify federation

<img src="https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/8d3a8f57-f752-41cc-8eee-d09a7e8995ea" width="500">

Prerequisites: Terraform 0.14+ installed locally, AWS CLI, the Azure CLI, kubectl

File Structure
```bash
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
|   |-- proxy_defaults.tf
|   |-- versions.tf
|-- counting-service
|   |-- main.tf
|   |-- versions.tf
|-- eks
|   |-- kubernetes.tf
|   |-- main.tf
|   |-- outputs.tf
|   |-- variables.tf
|   |-- versions.tf
```
# Provision clusters
## Provision an AWS EKS Cluster
The configuration creates a designated network to place the EKS resource. It also provisions an EKS cluster and an autoscaling group of workers to run the workloads. In eks directory, run terraform init and terrafrom apply.

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/b2ac8f0e-8a40-4415-854d-40c222a0d0d8)

## Provision an Azure AKS Cluster
First, we need to create an Active Directory service principal account to authenticate the Azure provider.
```
$ az login
$ az ad sp create-for-rbac --skip-assignment
$ az aks get-versions --location westus2 # first attempt at applying terraform apply resulted in error, saying that the kubernetes version is not available in the chosen location, so this command helps to find available versions 
```
Then, in aks directory, run terraform init and terrafrom apply.

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/e96df87a-1a25-41f4-9354-37adac836b51)

## Configure kubectl
Once Terraform provisions both clusters, navigate to ekse and aks directories and verify their respective Consul datacenters.
```
$ aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name) --alias eks
```
![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/52c14036-7c03-410b-ba19-2a2f3b6b737e)

```
$ az aks get-credentials --resource-group $(terraform output -raw resource_group_name) --name $(terraform output -raw kubernetes_cluster_name) --context aks
```
![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/85999fdb-5b5a-441e-8fc5-9f5722e38de4)

## Deploy Consul and configure cluster federation

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/6ef722a3-9813-4cb1-b918-4c50678b214e)

Once Terraform finishes provisioning both clusters, run and apply the configuration using the Consul Helm chart. This will: 
- Deploy the primary Consul datacenter and proxy defaults to EKS
- Load the federation secret into the AKS cluster
- Deploy the secondary Consul cluster

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/5d61f07c-affd-4902-89ec-231a7079e88c)

## Deploy ProxyDefaults
The ProxyDefaults use Custom Resource Definitions and must created after deploying the Consul datacenters. Apply the configuration to create the ProxyDefaults in both clusters. 

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/d936c9f6-445e-4508-8cf9-b4bf6c9bfef0)

### Verify cluster federation
List the pods in the default namespace in the EKS cluster to confirm that the Consul pods are running.
```
$ kubectl get pods --context eks
$ kubectl get proxydefaults --context eks
$ kubectl get pods --context aks
$ kubectl get proxydefaults --context aks
```
Warning: The ProxyDefaults may show as False due to an open issue in the Kubernetes provider kubernetes_manifest resource. Do not use this configuration in production.

Finally, verify that the clusters are federated by listing the servers in Consul's Wide Area Network (WAN).
```
$ kubectl exec statefulset/consul-server --context aks -- consul members -wan
```
The Consul members list includes nodes from both the dc1 and dc2 centers, confirming datacenter federation.

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/12a1678c-0bda-4260-a16d-7bc21cda3cae)

## Deploy an application
Deploy a two-tier application that communicates across the Kubernetes clusters using the federated Consul datacenters. The application counts how many times a user accesses it and consists of:
- a backend service named counting that increments a counter, deployed to your AKS cluster
- a frontend service named dashboard that calls the counting service and displays the counter value, deployed to the EKS cluster

Similarly to the Consul configuration from the previous section, this configuration uses the terraform_remote_state data source to access the contents of the AKS and EKS workspace's state files.
The configuration passes the attributes to the Kubernetes providers to authenticate against each cluster.
The counting service consists of a pod that Terraform deploys to the AKS cluster by referencing the aks aliased provider.
The dashboard service consists of a pod and service that Terraform deploys to the EKS cluster by referencing the eks aliased provider. 
This pod has the consul.hashicorp.com/connect-service-upstreams annotation to configure the service dependency on the counting service in the other Consul datacenter.

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/d555dd66-f1cc-43fe-b879-7d6bcd659f26)

Once Terraform deploys the resources to the clusters, visit the dashboard to verify the configuration. Enable port forwarding for the EKS cluster's dashboard pod to access the dashboard locally.
```
$ kubectl port-forward dashboard 9002:9002 --context eks
```
Navigate to http://localhost:9002/ in the web browser. 

![image](https://github.com/ZCHAnalytics/multi-cloud-kubernetes/assets/146954022/c09d0780-e45a-4863-b01c-f2bc3e4886ec)

# Clean up resources
Run `terraform destroy` separately in each of the project directories( aks, eks, consul, counting). 
