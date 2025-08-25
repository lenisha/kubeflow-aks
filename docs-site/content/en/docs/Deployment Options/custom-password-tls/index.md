---
categories: ["getstarted"]
tags: ["test", "sample", "docs"]
title: "Deploy Kubeflow with Password, Ingress and TLS"
linkTitle: "Deploy TLS and Custom Password"
date: 2025-08-23
weight: 4
description: >
  Deploying Kubeflow on AKS with Custom Password and TLS
---

## Background

In this lab, you will use the [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) to deploy an [Azure Kubernetes Service (AKS) Automatic](https://learn.microsoft.com/en-us/azure/aks/intro-aks-automatic) cluster. AKS Automatic offers a simplified, managed Kubernetes experience with automated node management, scaling, and security configurations. For more details, see the [AKS Automatic documentation](https://learn.microsoft.com/en-us/azure/aks/intro-aks-automatic). Note that AKS Automatic is currently in preview, while it provides faster setup and less manual configuration, it is not recommended for production use. For production workloads or when advanced features and customization are required, use regular AKS instead.
You will then install Kubeflow with a custom password and TLS configuration. This deployment option uses a self-signed certificate and an ingress controller. Replace the self-signed certificate with your own CA certs for production workloads.

You can follow these same instructions to deploy Kubeflow on a non-automatic AKS cluster. 

## DeployAKS Automatic
## Deploy AKS Automatic

Use the [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) to deploy an [AKS Automatic](https://learn.microsoft.com/en-us/azure/aks/intro-aks-automatic) cluster. 

{{< alert color="primary" >}}💡Note: In order to complete this deployment, you will need to have either  following permissions on Resource Group:
- Microsoft.Authorization/policyAssignments/write
- Microsoft.Authorization/policyAssignments/read.{{< /alert >}}


For detailed instructions on installing AKS Automatic, please refer to the [AKS Automatic installation documentation](https://learn.microsoft.com/en-us/azure/aks/automatic/quick-automatic-managed-network?pivots=azure-cli).

Login to the Azure CLI.
```bash
az login
```

{{< alert color="primary" >}}💡Note: If you have access to multiple subscriptions, you may need to run the following command to work with the appropriate subscription: `az account set --subscription <NAME_OR_ID_OF_SUBSCRIPTION>`.{{< /alert >}}

Set up your environment variables

```bash
RGNAME=kubeflow
CLUSTERNAME=kubeflow-aks-automatic
LOCATION=eastus
```

Create the resource group

```bash
az group create -n $RGNAME -l $LOCATION
```

Add or Update AKS extension
```bash
az extension add --name aks-preview
```
This article requires the `aks-preview` Azure CLI extension version **9.0.0b4** or later.

Create an AKS Automatic cluster
```bash
az aks create \
    --resource-group $RGNAME \
    --name $CLUSTERNAME \
    --location $LOCATION \
    --sku automatic \
    --generate-ssh-keys 
```

{{< alert color="primary" >}}💡Note: AKS Automatic is in Preview and requires feature to be registered in subscription.
```bash
az feature register --namespace Microsoft.ContainerService --name AutomaticSKUPreview
```
{{< /alert >}}

### Connect to AKS Automatic Cluster
After the cluster is created, you can connect to it using the Azure CLI. The following command retrieves the credentials for your AKS cluster and configures `kubectl` to use them.

```bash
az aks get-credentials --resource-group $RGNAME --name $CLUSTERNAME
```

Verify connectivity to the cluster. This should return a list of nodes.

```bash
kubectl get nodes
```

{{< alert color="primary" >}}💡Note: With AKS Automatic, you don't need kubelogin as the cluster uses managed identity for authentication.{{< /alert >}}

## Deploy Kubeflow with Password, Ingress and TLS

Clone this repo which includes the [kubeflow/manifests](https://github.com/kubeflow/manifests) repo as [Git Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)

```bash
git clone --recurse-submodules https://github.com/Azure/kubeflow-aks.git
```
{{< alert color="primary" >}}💡Note: The `--recurse-submodules ` flag helps to get manifests from git submodule linked to this  repo{{< /alert >}} 

Change directory into the newly cloned directory

```bash
cd kubeflow-aks
```

From the root of the repo, ensure you're using the v1.10-branch:

```bash
cd manifests/
git checkout v1.10-branch
cd ..
```

- Copy the TLS deployment files:

```bash
cp -a deployments/tls manifests/tls
```


### Configure Custom password
{{< alert color="warning" >}}⚠️ Warning: For production deployments, configure Azure AD integration instead of static passwords.{{< /alert >}}

In the next steps generate password hash for your custom password and replace it in the dex-passwords.yaml file.

First generate Password/Hash by following steps described in `kubeflow` docs [using python to generate bcrypt hash](https://github.com/kubeflow/manifests/blob/master/README.md#change-default-user-password). Or for simplicity you can use an online tool like [bcrypt-generator](https://www.bcrypt-generator.com/) to create a new hash.

```bash
PASSWORD="your_custom_password"
PASSWORD_HASH=$(python3 -c "import bcrypt; print(bcrypt.hashpw(b'$PASSWORD', bcrypt.gensalt()).decode())")
```

Update the password hash in the `manifests/tls/dex-passwords.yaml` secret:
```bash
sed -i "s|<YOUR_DEX_USER_PASSWORD>|$PASSWORD_HASH|g" manifests/tls/dex-passwords.yaml
```

{{< alert color="primary" >}}💡Warning: Always change default password before exposing Kubeflow dashboard externally.{{< /alert >}}

## Install Kubeflow


- Deploy Kubeflow

```bash
cd manifests/  
while ! kustomize build tls | kubectl apply --server-side=true -f -; do echo "Retrying to apply resources"; sleep 10; done
```

{{< alert color="primary" >}}💡Note: The `--server-side=true` flag helps with large CRDs that may exceed annotation size limits. The retry loop handles dependency ordering issues during installation.{{< /alert >}}

- Once the command has completed, check the pods are ready

```bash
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
kubectl get pods -n knative-eventing
kubectl get pods -n knative-serving
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow-user-example-com
```
{{< alert color="primary" >}}💡Note: Depending on the VM SKU automatic picked for your region it might scale up to 4 nodes to run all the Kubeflow components{{< /alert >}}

## Expose the Kubeflow dashboard using Ingress with TLS 

There are couple options to expose your Kubeflow cluster with proper HTTPS using Ingress. See note in Kubeflow docs [NodePort / LoadBalancer / Ingress](https://github.com/kubeflow/manifests/blob/master/README.md#nodeport--loadbalancer--ingress) In this example we will use the **nginx ingress controller** which is included as part of the **app-routing-system** addon in AKS Automatic.

### Step 1: Create TLS Certificate

We can create a self-signed certificate for the Kubeflow with IP available on Nginx ingress LoadBalancer or assign DNS Label 

## Step 1. Find IP or DNS Label of Nginx ingress

- Obtain Nginx IP

```bash
NGINX_IP=$(kubectl get svc -n app-routing-system -o jsonpath='{.items[?(@.spec.type=="LoadBalancer")].status.loadBalancer.ingress[0].ip}')
echo "Nginx IP: $NGINX_IP"
```

- Optional: Use Azure DNS for a friendly URL
You can also configure a custom domain name assigned to the Nginx ingress service using Azure DNS:

```bash
kubectl annotate service nginx -n app-routing-system \
  service.beta.kubernetes.io/azure-dns-label-name=my-kubeflow-cluster
```

This will make Kubeflow accessible at: `my-kubeflow-cluster.$LOCATION.cloudapp.azure.com`
{{< alert color="primary" >}}💡Note: DNS Label must be unique for the Azure region.{{< /alert >}}

### Step 2: Create TLS Certificate

If using IP address create following certificate:

```bash
echo "apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kubeflow-tls-cert
  namespace: app-routing-system
spec:
  secretName: kubeflow-tls-secret
  ipAddresses:
    - $NGINX_IP
  isCA: false
  issuerRef:
    name: kubeflow-self-signing-issuer
    kind: ClusterIssuer
    group: cert-manager.io" | kubectl apply -f -
```

If using DNS label use following definition (replace `my-kubeflow-cluster` with your unique dns label)

```bash
echo "apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kubeflow-tls-cert
  namespace: app-routing-system
spec:
  secretName: kubeflow-tls-secret
  dnsNames:
    - my-kubeflow-cluster.$LOCATION.cloudapp.azure.com
  isCA: false
  issuerRef:
    name: kubeflow-self-signing-issuer
    kind: ClusterIssuer
    group: cert-manager.io" | kubectl apply -f -
```


- Deploy the certificate:

```bash
kubectl apply -f tls/certificate.yaml
```

Wait for the certificate to be ready:

```bash
kubectl wait --for=condition=Ready certificate/kubeflow-tls-cert -n istio-system --timeout=300s
```

## Step 3. Configure Ingress
Create and apply an ingress manifest to expose the Kubeflow components:

```bash
echo 'apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubeflow-ingress
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  tls:
  - secretName: kubeflow-tls-secret
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: istio-ingressgateway
            port:
              number: 80' | kubectl apply -f -
```

Verify Ingress:

```bash
kubectl get ingress kubeflow-ingress -n istio-system
```

Wait for the `ADDRESS` field to show an external IP address (this may take a few minutes).

```
NAME               CLASS                                HOSTS   ADDRESS       PORTS   AGE
kubeflow-ingress   webapprouting.kubernetes.azure.com   *       xxx.149.0.222   443      16m
```

## Access Kubeflow Dashboard

You can now access the Kubeflow dashboard at `https://$NGINX_IP` or `https://my-kubeflow-cluster.$LOCATION.cloudapp.azure.com` if DNS was configured.

{{< alert color="warning" >}}⚠️ Warning: You will see a browser warning about the self-signed certificate. Click "Advanced" and proceed to access the dashboard.{{< /alert >}}

Log in using:
- Email: user@example.com (or the email you configured)
- Password: The password you used to generate the hash

## Testing the deployment with a Notebook server

You can test that the deployments worked by creating a new Notebook server using the GUI.

1. Click on "Create a new Notebook" on the Kubeflow dashboard
    ![creating a new Notebook](./images/create-new-notebook.png)
1. Click on "+ New Notebook" in the top right corner of the resulting page
1. Enter a name for the server
1. Leave the "jupyterlab" option selected
1. Feel free to pick one of the images available, in this case we choose the default
1. Set Requested CPU to 0.5 and requested memory in Gi to 1
1. Under Data Volumes click on "+ Add new volume"
1. Expand the resulting section
1. Set the name to datavol-1. The default name provided would not work because it has characters that are not allowed
1. Set the size in Gi to 1
1. Uncheck "Use default class"
1. Choose a class from the provided options. In this case I will choose "azurefile-premium"
1. Choose ReadWriteMany as the Access mode. Your data volume config should look like the picture below
    ![data volume config](./images/data-volume-config.png)
1. Click on "Launch" at the bottom of the page. A successful deployment should have a green checkmark under status, after 1-2 minutes.
    ![deployment successful](./images/server-provisioned-successfully-tls.png)
1. Click on "Connect" to access your jupyter lab
1. Under Notebook, click on Python 3 to access your jupyter notebook and start coding

## Destroy the resources

Run the command below to destroy the resources you just created after you are done testing

```azurecli
az group delete -n $RGNAME
```

