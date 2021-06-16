---
title: Azure Kubernetes Service Pipelines
date: 2021-06-15 22:00:00
tags: [cloud, kubernetes, devops, pipelines]
---

In the last months I have been working with [Azure DevOps](https://azure.microsoft.com/en-us/services/devops) and completed the [Devops Engineer certification](https://docs.microsoft.com/en-us/learn/certifications/devops-engineer) from Microsoft. Recently I've also been learning Kubernetes and wanted to build an end to end pipeline to deploy a sample Go application, with Ingress and TLS, to the managed [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/en-us/services/kubernetes-service). I will describe the steps I've taken and share the GitHub [repo](https://github.com/ruial/aks-devops) with all the Infrastructure as Code (IaC) using Terraform.

## Azure DevOps

Azure Devops is Microsoft's solution to automate the software development lifecycle. It has the following services:

- **Azure Boards** to manage work items, similar to Jira.

- **Azure Pipelines** for CI/CD automation, like Jenkins or GitHub Actions.

- **Azure Repos** is a repository service for Git or TFVC.

- **Azure Test Plans** for manual testing (really expensive)

- **Azure Artifacts** to publish packages, an alternative to Artifactory.

Using a SaaS instead of rolling your own comes at a cost. Pricing is a bit high at 5€/user and 12.65€/parallel job (for a self hosted agent), with 5 free users and 1 free job. For bigger enterprises, using self hosted agents instead of Microsoft hosted agents is usually the better option as parallel jobs are cheaper, builds should start faster, and you can make use of incremental builds, internal networking and [managed identities](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview). When choosing Azure Devops instead of other CI/CD tools, pricing has to be considered as the free tier limits are reached very soon and pipelines start to queue, however there is less infrastructure and code to manage.

## Kubernetes

In recent years, [Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes) became the standard to manage applications deployed as containers. It simplifies operations like high availability, autoscaling, rolling updates, rollbacks, service discovery, among others. Some of the most important concepts are:

- **Pods** - groups of containers that share the same compute resources and network. Each pod is scheduled to run on nodes inside the cluster and gets assigned an IP address.

- **Deployments** - describe the desired state of pods, like the number of replicas and rollout strategy.

- **Services** - to expose Pods for consumption inside (ClusterIP) or outside (NodePort, LoadBalancer) the cluster, as a network service.

Kubectl is the command line interface to manage the cluster, which interacts with the Kubernetes API. Deploys can be performed with imperative commands, but declarative yaml manifests are the way to go for a reproducible environment.

When it comes to exposing services to the outside world, an alternative to using a Service of type LoadBalancer is to combine ClusterIP Services and an Ingress. A LoadBalancer Service per application results in a new public IP and public load balancer on the cloud, which can get expensive. An Ingress, such as [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/#azure) is often used to route HTTP and HTTPS traffic to internal services, based on the host header or request path, using a single public IP and cloud Load Balancer. With projects like [external-dns](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md) and [cert-manager](https://cert-manager.io/docs/installation/kubernetes), the management of DNS records and TLS certificates is fully automated.

## Proof of concept

As I was following a course on [Udemy](https://github.com/stacksimplify/azure-aks-kubernetes-masterclass), I've adapted some of the examples for a simple use case of deploying a Go application and having the entire infrastructure managed with Terraform. With the [Azure Devops Provider](https://registry.terraform.io/providers/microsoft/azuredevops/latest/docs) and [Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs), I was able to automate most of the provisioning, only had to configure a few things manually in Azure DevOps (add the Kubernetes Environment, create the pipelines) and edit some variables in the pipeline declarations after having the resources provisioned. You would be able to reproduce everything by cloning the repo. I will only describe the global structure, as each file is easy to interpret:

```
.
# go app with multi-stage container
├── goproject # go app with multi-stage container
│   ├── Dockerfile
│   ├── go.mod
│   ├── main.go
│   └── main_test.go
# manifests for the go app and the ingress, in some orgs these
# could even be managed in different repos by different teams
├── kubernetes
│   ├── gomanifests
│   │   ├── deployment.yml
│   │   └── service.yml
│   └── ingressmanifests
│       ├── azure.json
│       ├── cert-manager-kube-system.yml
│       ├── cert-manager.yml
│       ├── cluster-issuer.yml
│       ├── dns-secret.yml
│       ├── external-dns.yml
│       ├── ingress-nginx.yml
│       └── ingress.yml
# azure devops pipelines definitions
├── pipelines
│   ├── go-master-pipeline.yml
│   └── ingress-pipeline.yml
# terraform configuration files
└── terraform
    ├── ado.tf
    ├── akscluster.tf
    ├── main.tf
    ├── output.tf
    └── variables.tf
```

I've opted to include the cert-manager and ingress-nginx manifests in the repo but they could be deployed with their GitHub URL or using [Helm](https://helm.sh). The *azure.json* file has the details of the managed identity created by AKS, that also has the RBAC assignment to edit DNS records on the Azure DNS zone created by Terraform. To get the required IDs for the config file:

```sh
# managed identity client id
az aks show -g aks-devops-we -n aks-devops-we-cluster --query "identityProfile.kubeletidentity.clientId"
# subscription id and tenant id
az account list
# encode as base64 the secret azure.json on dns-secret.yml
cat azure.json | base64
```

After deploying the resources, I've created the NS records *aks.briefbytes.com* pointing to the assigned Azure DNS nameservers to delegate this zone to Azure. A limitation of Azure Devops KubernetesManifest tasks required me to break the cert-manager manifest in 2 because it was not able to deploy Kubernetes objects in 2 different namespaces in the same task.

Terraform is responsible for creating the Kubernetes cluster, the container registry (ACR) to store images, the public dns zone, role assignments and configuring the project and service connections on Azure Devops. This is achieved by running the following:

```sh
# this could also be on a pipeline using service principal or managed identity
export AZDO_PERSONAL_ACCESS_TOKEN=<Generate your PAT with full access permissions>
export AZDO_ORG_SERVICE_URL=https://dev.azure.com/<Your org>
az login
terraform plan -out plan
terraform apply plan

# to login on the cluster (context saved in ~/.kube/config)
az aks get-credentials --resource-group aks-devops-we --name aks-devops-we-cluster
kubectl cluster-info
```

Adding an environment on Azure Devops will create another service connection and deployments from different pipelines will be visible there. This must be done manually, as the Terraform provider does not yet support it.

{% asset_img "aks-env.png" "Azure Devops AKS Environment" %}

After adding the environment, make sure to create the pipeline and edit the ACR name, its service connection ID and of course set your own DNS zone. Your pipeline should run successfully:

{% asset_img "go-pipeline-stages.png" "AKS Pipeline Stages" %}

The creation of DNS records is fast but TLS certificates from [Let's Encrypt](https://letsencrypt.org) can take a few minutes to be issued. Take care when changing domain names in production to avoid downtime. You can use the following commands to inspect the logs:

```sh
kubectl logs -n cert-manager -f <kubectl-pod>
kubectl get certificate
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

The final result should look like this:

{% asset_img "aks-result.png" "AKS Result" %}

With this setup, when code is pushed or merged to the master branch, the pipeline will publish a new image if the unit tests pass and do the rolling update without downtime. To save on AKS costs, do not enable logging as Log Analytics is [costly](https://feedback.azure.com/forums/914020-azure-kubernetes-service-aks/suggestions/38495200-aks-has-a-high-log-analytics-cost-azure-monitor), however in production make sure to setup proper monitoring with Prometheus and logging with the Elastic stack or their alternatives. After testing, the cluster can be destroyed with Terraform or shutdown through Azure CLI:

```sh
az extension add --name aks-preview
az aks stop -g aks-devops-we -n aks-devops-we-cluster
```

## Conclusion

The cloud and Kubernetes provide massive productivity, scalability and reliability improvements and their adoption will surely continue to grow. There are so many more topics about Kubernetes and AKS that could be covered but I will finish this post with some useful docs:

- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet)
- [AKS Ingress](https://docs.microsoft.com/en-us/azure/aks/ingress-tls)
- [AKS Networking](https://docs.microsoft.com/pt-pt/azure/aks/configure-kubenet)
