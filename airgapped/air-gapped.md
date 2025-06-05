# VKS Deployment Guide for Air-Gapped Environments

## Introduction
The complete procedure to install a VKS cluster with additional Tanzu package add-ons within an air-gapped environment involves the following significant steps:

- Copy the relevant files and binaries to be moved to the air-gapped environment.
- Enable the Supervisor.
- Upload Kubernetes release OVAs to a Content Library.
- Create vSphere Namespace(s) for VKS Clusters(s).
- Configure the air-gapped Admin host in the air-gapped environment. 
- Mirror an Enterprise OCI registry with the relevant container images for the platform. 
- Install the relevant Supervisor Services.
- Deploy the VKS Cluster(s).
- Deploy the Tanzu Packages on the VKS Cluster(s).

The data flow of packages, binaries, and images between the internet-connected and air-gapped environment can be summarized by the picture below -

![image](/airgapped/ocireg-dataflow.png)

## Terminiology
* **Bastion Host** A host (preferably a Linux VM) that is connected to the Internet or has access to download packages, binaries, and images from the Internet.
* **Admin Host** A host (preferably a Linux VM) within the air-gapped environment without internet access. The Admin host generally has network connectivity to all the hosts in the air-gapped environment. Files downloaded from the Internet to the Bastion host are transferred to the Admin Machine, and administrators can use it to interact with the platform.
* **Tanzu CLI** A plugin-based CLI that is used to interact with the Supervisor and VKS clusters
* **Tanzu Packages** Tanzu Packages enable administrators and users to add and manage standard services and add-ons on Kubernetes clusters using the Tanzu CLI or Carvel Custom Resources.

## Prerequisites
An **Enterprise OCI-complaint registry** is assumed to be available in the air-gapped environment and accessible by all platform nodes, including the admin host. The registry must be accessible over HTTPS. The certificate can be signed by a trusted certificate authority or self-signed.  
**Note** If an Enterprise registry is unavailable in the air-gapped environment, please visit this [document](/airgapped/air-gapped-harbor.md) to install and configure Harbor as Bootstrap and Platform registries.

## Bill of Materials
The table below provides sample hostnames and versions used throughout the document for easy reference -

|Component|Version|Sample Hostname (where applicable)|
|---------|-------|----------------------------------|
|Bastion Host|Ubuntu 24.04.4|bastion.internet.lab.test|
|Admin Host (Air-gapped)|Ubuntu 24.04.4 (identical to the Bastion Host)|admin.env1.lab.test|
|vCenter|8.0U3d|vcenter.env1.lab.test|
|ESXi|8.0U3d|esxi[0..xxx].env1.lab.test|
|Supervisor|v1.29|supervisor0.env1.lab.test|
|Enterprise Registry||registry1.env1.lab.test|
|TKC|v1.29.4/v1.30.1/v1.31.1|workload-vsphere-vks1|
|Tanzu Package|v2024.8.21||
|TKG Service|3.1.1||

Additionally, it is crucial to have the following packages and binaries installed on both the Bastion and Admin Host - 
* `wget`
* `curl`
* `docker` (Preferably from the Docker website [https://docs.docker.com/engine/install/])
* `jq`
* `yq` (Some Linux distributions come with their implementation of yq. These packages are not the latest and/or provide the full functionalities. The newest version of the yq package can be downloaded from [https://github.com/mikefarah/yq/releases])
* `openssl` for certificate generation and validations. 
* Additional troubleshooting and diagnostic tools as needed. 

## 1. Download all required Plugins, Binaries, and Images
This document utilizes an Ubuntu 24.04.4-based Bastion host (**bastion.internet.lab.test**) for this stage. Below are the key plugins and packages that must be downloaded, each playing a critical role in the platform deployment process.

### 1a. Kubernetes Release OVA files
Kubernetes releases provide the Kubernetes software distribution for VKS clusters. VMware distributes Kubernetes releases as virtual machine templates, which you synchronize with the platform using a vCenter Content Library. Download the latest Kubernetes release files from https://wp-content.vmware.com/v2/latest/. The versions to be downloaded may depend on the workload requirements. It would be best to download three or more of the latest versions. Follow [Step #11 from the official docuentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/administering-kubernetes-releases-for-tkg-service-clusters/create-a-local-content-library-for-air-gapped-cluster-provisioning.html) to download the relevant Kubernetes release files for each version.  

### 1b. Tanzu CLI and Plugins
While the Tanzu CLI and its plugins will be installed on the Bastion host (`bastion.internet.lab.test`), they must also be copied to the Admin Machine as part of the file transfer process. When this document was written, Tanzu CLI v1.1.0 was supported with vSphere v8.0.3. The steps below involve installing the Tanzu CLI on the Bastion host, which is necessary to download the Tanzu CLI plugins. The installation of the Tanzu vSphere Plugin is mandatory. Other plugins can be downloaded and installed as required. Depending on your entitlements, the required Tanzu CLI binary can also be downloaded from the Broadcom website. Please have a look at the link for additional information. 

```bash
## Download Tanzu CLI
wget https://github.com/vmware-tanzu/tanzu-cli/releases/download/v1.1.0/tanzu-cli-linux-amd64.tar.gz

## Install Tanzu CLI on the bastion host
tar -xzvf ./tanzu-cli-linux-amd64.tar.gz

## Move to Tanzu CLI to Executable Path
sudo install ./v1.1.0/tanzu-cli-linux_amd64 /usr/local/bin/tanzu
```

With the Tanzu CLI installed on the Bastion host, we can use the `tanzu` command to download the Tanzu CLI vSphere Plugins.

```bash
## Command to list all available plugins
tanzu plugin group search --show-details

## Command to list the Plugins specific to vSphere
tanzu plugin group search -n vmware-vsphere/default --show-details

## Install Tanzu vSphere Plugins for the Bastion host
tanzu plugin install --group vmware-vsphere/default:v8.0.3

## Pull the plugin group into a tarball using the following command
tanzu plugin download-bundle --group vmware-vsphere/default --to-tar tanzu-cli-plugins.tar.gz
```

### 1c. Tanzu Packages
Tanzu Packages enable administrators and users to use the Tanzu CLI or Carvel Custom Resources to add and manage standard services and add-ons on Kubernetes clusters. With Tanzu Packages, you can deploy various packages to VKS Clusters, such as cert-manager, Contour, Prometheus, Grafana, and more. Download the relevant bundle using the following command -

```bash
tanzu imgpkg copy -b projects.registry.vmware.com/tkg/packages/standard/repo:v2024.8.21 --to-tar ./tanzu-packages.tar --cosign-signatures
```

### 1d. Binaries and YAML files required for Supervisor Services
Supervisor Services are Carvel packages and are defined by their configuration file. The configuration file contains the reference to the image that holds the package manifest. As a prerequisite for migrating the Supervisor Services to the air-gapped registry, we need to extract the value of the image containing the package manifest. To do so, we can download the configuration YAML file from the Supervisor Services list and look for the following key - `spec.template.spec.fetch.imgpkgBundle[].image` for the Package object. For example, in the argocd.yaml configuration file for the ArgoCD Operator, the value is the image that would need to be migrated to the air-gapped environment. 

```yaml
...
---
apiVersion: data.packaging.carvel.dev/v1alpha1
kind: Package
metadata:
  name: argocd-operator.fling.vsphere.vmware.com.0.12.0
spec:
  refName: argocd-operator.fling.vsphere.vmware.com
  template:
    spec:
      fetch:
      - imgpkgBundle:
          image: projects.packages.broadcom.com/vsphere-labs/argocd-operator@sha256:850cc1f1253f2f898c61d8305b6004373c2f9422faaf562afb26456b790a3155
...
```
For additional information and examples, please refer to [Steps 1, 2 and 3 of the official documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/vsphere-supervisor-services-and-workloads-8-0/deploying-supervisor-services-from-a-private-container-image-registry/relocate-supervisor-services-to-a-private-registry.html). While the official documentation refers to the `imgpkg` binary to perform the download function, the Tanzu CLI's `imgpkg plugin` performs the identical function. 

####, E.g., Download Contour Supervisor Service Binaries and associated YAML files.
When writing this document, the latest version of the Contour Supervisor Service is “v1.28.2”. Please refer to ​​the vSphere Supervisor Services page to check for the updated versions. Download the `contour.yml` and `contour-data-values.yml` files required for the Contour Supervisor Service. These files can be accessed through the vSphere Supervisor Services page. The `contour.yml` file can be obtained by selecting the appropriate Contour version, and the `contour-data-values.yml` can be downloaded from the "Contour Sample values.yaml" section in the same vSphere Supervisor Services page. Locate the value of the image from the `contour.yml` file as described above. 

You can execute the following command from the Bastion host to download the image bundle as a tarball.

```bash
## Download Contour Binaries
tanzu imgpkg copy -b projects.packages.broadcom.com/tkg/packages/standard/contour:v1.28.2_vmware.1-tkg.1 --to-tar contour-v1.28.2.tar --cosign-signatures
```
Perform similar steps for other Supervisor Services that must be installed in the air-gapped environment. 

The table below provides the sample list of Supervisor Services that can be downloaded and installed on the platform -

|Service Name|Type|Version|Configuration file|Data Values yaml|
|------------|----|-------|------------------|----------------|
|TKG Service|Core|v3.2.0|[tkgs.yaml](https://packages.broadcom.com/artifactory/vsphere-distro/vsphere/iaas/kubernetes-service/3.2.0-package.yaml)|None|
|Contour|Standard|v1.28.2|[contour.yaml](https://vmwaresaas.jfrog.io/ui/api/v1/download?repoKey=supervisor-services&path=contour/v1.28.2/contour.yml)|[values.yaml](https://vmwaresaas.jfrog.io/ui/api/v1/download?repoKey=supervisor-services&path=contour/v1.24.4/contour-data-values.yml)|
|Harbor|Standard|v2.9.1|[harbor.yaml](https://vmwaresaas.jfrog.io/ui/api/v1/download?repoKey=supervisor-services&path=harbor/v2.9.1/harbor.yml)|[Sample values.yaml](https://vmwaresaas.jfrog.io/ui/api/v1/download?repoKey=supervisor-services&path=harbor/v2.9.1/harbor-data-values.yml)|
|Consumption Interface|Standard|v1.0.2|[lci.yaml](https://vmwaresaas.jfrog.io/artifactory/supervisor-services/cci-supervisor-service/v1.0.2/cci-supervisor-service.yml)|None|
|ExternalDNS|Standard|v0.13.4|[externaldns.yaml](https://vmwaresaas.jfrog.io/ui/api/v1/download?repoKey=supervisor-services&path=external-dns/v0.13.4/external-dns.yml)|[See doc]()|
|ArgoCD Operator|Experimental|v0.12.0|[argocd.yaml](https://raw.githubusercontent.com/vsphere-tmm/Supervisor-Services/refs/heads/main/supervisor-services-labs/argocd-operator/v0.12.0/argocd-operator.yaml)|[Sample values.yaml](https://github.com/vsphere-tmm/Supervisor-Services/blob/main/supervisor-services-labs/argocd-operator/v0.12.0/values.yaml)|

### Summary
The following files, binaries, and packages have been successfully downloaded in this section and **must be transferred to the Admin host**. 
* Kubernetes Release OVA files. 
* Tanzu CLI (tanzu-cli-linux-amd64.tar.gz).
* Tanzu CLI vSphere Plugin (tanzu-cli-plugins.tar.gz).
* Tanzu Packages (tanzu-packages.tar). This file can be gzipped if needed. 
* Supervisor Service packages and Yaml Files.

## 2. Enable the Supervisor
Using the steps and directions in the official [documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/installing-and-configuring-vsphere-supervisor.html), configure the required networking, storage policies, and profiles and enable the Supervisor on `vcenter.env1.lab.test`.

## 3. Create a Kubernetes release content library and upload Kubernetes release images
The Kubernetes release OVAs downloaded on the Bastion host and copied to the Admin host must be uploaded to a Content library within the vCenter. Before proceeding, a local content library must be created. The "Create a Local Content Library (for Air-Gapped Cluster Provisioning)" [documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/administering-kubernetes-releases-for-tkg-service-clusters/create-a-local-content-library-for-air-gapped-cluster-provisioning.html) provides instructions on creating and importing Kubernetes Release (Kr) images into the content library. **Step #12** provides details on files (downloaded previously in step 1a) that need to be uploaded to the local content library. 

## 4. Create vSphere Namespace(s) for VKS Clusters(s)
If not already created, a vSphere namespace should be created. Refer to "[Configure a vSphere Namespace for Hosting TKG Service Clusters](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/configuring-vsphere-namespaces-for-hosting-tkg-service-clusters.html)" to configure the vSphere Namespace.

## 5. Configure Admin host
The Admin host (**admin.env1.lab.test**) is essential to the next deployment stage. It performs crucial tasks such as uploading binaries and image bundles to the repository, deploying VKS Clusters, and deploying Tanzu packages on the VKS Clusters. As the control center, this machine enables effective deployment management and repository coordination. This document uses an `Ubuntu 24.04.4` machine with *Docker* installed on the Admin Machine. If Docker isn't installed, please review the [official documentation](https://docs.docker.com/engine/install/ubuntu/) for installation guidance. The instructions may have to be modified for an airgapped installation. The recommended system configuration is as follows:
* CPU: 2 vCPUs
* Memory: 4 GB
* Storage: 150–200 GB of free space

Note: Before moving forward, verify that all the files mentioned in the Summary section in Step 1 have been successfully copied to the Admin host.

### 5a. Download and Install kubectl and kubectl-vsphere CLI
`kubectl` is the command-line tool used to interact with Kubernetes clusters. It allows users to manage and inspect resources within a Kubernetes environment. `kubectl-vsphere` is a VMware-specific plugin for `kubectl` that enables administrators and developers to interact with the Supervisor and manage VKS clusters running on vSphere. It integrates with the Supervisor and extends `kubectl` with commands specific to VMware's TKG Service.

You can download and install the `kubectl` and `kubectl-vsphere` plugins to your Admin machine by accessing the Supervisor Cluster Kube-API server endpoint UI or using the command below.

```bash
wget https://<Supervisor-KubeAPI-Endpoint>/wcp/plugin/linux-amd64/vsphere-plugin.zip --no-check-certificate

## Sample Command:
wget https://supervisor0.env1.lab.test/wcp/plugin/linux-amd64/vsphere-plugin.zip --no-check-certificate

## After downloading the vsphere-plugin.zip file, use the following commands to unzip it and add the binaries to the executable path.
unzip ./vsphere-plugin.zip
cd ./bin
sudo install kubectl /usr/local/bin/kubectl
sudo install kubectl-vsphere /usr/local/bin/kubectl-vsphere

## Verify the version by executing the below commands
kubectl version
kubectl vsphere version
```

### 5b. Login to Supervisor
If not logged in, log in to the Supervisor using the `kubectl-vsphere` plugin. 

```bash
# Connect to vSphere Namespace using kubectl vsphere CLI
kubectl vsphere login --vsphere-username <sso_username> --server=https://<SupervisorAPIEndpoint> --insecure-skip-tls-verify --tanzu-kubernetes-cluster-namespace <vsphere-namespace>

## Sample Command
kubectl vsphere login --vsphere-username administrator@vsphere.local --server=https://supervisor0.env1.lab.test --insecure-skip-tls-verify
```

### 5c. Add the Enterprise Registry certificate to the Admin host Trust Store (Optional)
To ensure the Admin machine trusts the registry, we must add the Enterprise registry certificate (E.g., the contents saved in file `registry1.crt`) to the Trust Store. This step is optional and required only when using a certificate not signed by a trusted Certificate authority (e.g., a self-signed certificate).

```bash
## Ubuntu specific example
sudo cp registry1.crt /usr/local/share/ca-certificates 
sudo update-ca-certificates
```

Once the certificate has been added to the trust store, log in to the registry endpoint using Docker.

```bash
## Restart Docker Service
sudo systemctl reload docker
sudo systemctl restart docker

## Command to Login to Harbor Supervisor Service Endpoint
docker login registry1.env1.lab.test

## Enter the username and password when prompted. The expected output is shown below:
=====
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credential-stores

Login Succeeded
=====
```
### 5d. Install the Tanzu CLI

```bash
cd /<path-to-files-copied-from-bastion>/

tar -xzvf ./tanzu-cli-linux-amd64.tar.gz

## Move to Tanzu CLI to Executable Path
sudo install ./v1.1.0/tanzu-cli-linux_amd64 /usr/local/bin/tanzu

## Verify the installation using the below command
tanzu version

## Upload plugins to enterprise repository
tanzu plugin upload-bundle –tar tanzu-cli-plugins.tar.gz –to-repo registry1.env1.lab.test/tanzu-cli/plugins # Note the path.
 
## Update repository location
tanzu plugin source update default --uri registry1.env1.lab.test/tanzu_cli/plugins/plugin-inventory:latest
 
## Install the Plugins
tanzu plugin install –group vmware-vsphere/default:v8.0.3
 
## Validate the existace of plugin files in the folder
ls -al ~/.local/share/tanzu-cli/
tanzu plugin list
```

## 6. Add the Enterprise Registry certificate to the Supervisor (Optional)
The Supervisor must trust the Enterprise registry certificate. This step is optional and required only when using a certificate not signed by a trusted Certificate authority (e.g., a self-signed certificate). To perform this step, navigate to Workload Management -> Supervisor -> Configure -> Container Registries. Click on Add Registry. 

![image](/airgapped/add-cert0.png)

Input the Registry host URL, TLS Certificate of the registry (the content of `registry1.crt`), Username, and Password. Note that while the UI states that the Username and Password are optional, they are currently mandatory.

![image](/airgapped/add-cert1.png)

## 7. Upload Packages to the Enterprise registry
Create two projects/folders, with public access, within the registry to upload -
* All the supervisor services (e.g., `sup-services`) must be installed on the supervisor. The steps may vary depending on the registry vendor and version.
* The Tanzu Packages will be installed on the VKS cluster. (e.g. `tanzu-packages`). The steps may vary depending on the registry vendor and version.

### 7a. Upload Tanzu Packages to the Enterprise Registry
The Tanzu Package bundle, downloaded in Step 1c, must be uploaded to the Enterprise registry using the Tanzu CLI. 

```bash
## Sample Command
tanzu imgpkg copy --tar tanzu-packages.tar --to-repo registry1.env1.lab.test/tanzu-packages/packages/standard/repo --cosign-signatures --registry-response-header-timeout 600s
```

### 7b. Upload Supervisor Services to the Enterprise Registry
All the Supervisor Services image bundle binaries downloaded in Step 1d must be uploaded to the Enterprise registry. Follow [steps 4 and 5 from the official documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/vsphere-supervisor-services-and-workloads-8-0/deploying-supervisor-services-from-a-private-container-image-registry/relocate-supervisor-services-to-a-private-registry.html) to complete this critical step. While the official documentation refers to the `imgpkg` binary to perform the download function, the Tanzu CLI's `imgpkg plugin` performs the identical function. 

```bash
## Sample Command
tanzu imgpkg copy --tar contour-v1.28.2.tar --to-repo registry1.env1.lab.test/sup-services/contour --cosign-signatures
```

Additionally, the corresponding Supervisor Service YAML needs to be updated with the new Enterprise registry valid location -

```yaml
# Contour.yaml
...
template:
  spec:
    fetch:
    - imgpkgBundle:
        image: registry1.env1.lab.test/sup-services/contour:v1.24.4_vmware.1-tkg.1
...
```
Perform these steps for all the Supervisor Services that were previously downloaded. Once completed, add the required Supervisor Services to the Supervisor using the [steps provided in the documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/vsphere-supervisor-services-and-workloads-8-0/deploying-supervisor-services-from-a-private-container-image-registry/install-and-use-the-supervisor-service.html).

## 8. Deploy VKS Cluster(s)

### 8a. Update TKG Service to the latest version
When writing the documentation, the latest TKG Service was v3.2. Since each TKG Service introduces additional features and fixes, we must apply these updates to make the new features and fixes available. This document will update the core TKG Service from 3.1.1 to 3.2. Follow the relevant sections in the [documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/installing-and-upgrading-the-tkg-service.html) to complete the upgrade. 

### 8b. Create a Workload Cluster certificate (Optional)
The VKS cluster nodes and `kapp` controllers running on the VKS cluster must trust the Enterprise registry's SSL certificate. This step is optional and not required if a trusted certificate authority signs the Enterprise registry’s certificate. The following [product documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/provisioning-tkg-service-clusters/using-the-cluster-v1beta1-api/v1beta1-example-cluster-with-additional-trusted-ca-certificates-for-ssl-tls.html) link provides details on how to perform this step. 

```bash
## Sample command
 base64 -w 0 registry1.crt | base64 -w 0 
```

```yaml
## additional-ca-1.yaml
apiVersion: v1
data:
  ## The value of additional-ca-1 is the output of the above command.
  additional-ca-1: TFMwdExTMUNSGlSzZ3Jaa...VVNVWkpRMEMwdExTMHRDZz09
kind: Secret
metadata:
  name: workload-vsphere-vks1-user-trusted-ca-secret
  namespace: ns01
type: Opaque
```

### 8c. Deploy a Workload Cluster
Deploying a VKS Cluster (using an Ubuntu image). Review each section at a minimum and adjust the necessary fields as needed. Refer to the [“Using the Cluster v1beta1 API” documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/provisioning-tkg-service-clusters/using-the-cluster-v1beta1-api.html) for more configuration options.
* Control Plane Size:  
** Nodes: 1 or 3 for high availability 
** vCPUs: 4, Memory: 16GB, Storage: 50GB
* Worker Node Size: 
** Nodes: 3 for a standard-size stack 
** vCPUs: 4, Memory: 16GB, Storage: 50GB

While `v1beta1` provides numerous configuration options, the following additional variables may be added for a default `podSecurityStandard` and additional certificate `trust.` 
```yaml
## vksConfig.yaml
...
    variables:
      - name: podSecurityStandard
        value:
          audit: restricted
          auditVersion: latest
          enforce: privileged
          enforceVersion: latest
          warn: privileged
          warnVersion: latest
#      Uncomment the below lines if using an Enterprise registry. See above section 8b 
#      - name: trust
#        value: 
#          additionalTrustedCAs:
#          - name: additional-ca-1
...
```

```bash
## If using Enterprise registry with a private certificate. Please take a look at section 8b.
kubectl apply -f <additional-ca-1.yaml> -n ns01

## Deploy VKS cluster
kubectl create -f <vksConfig.yaml> -n ns01

## Command to check the status of Cluster Creation
Kubectl get cluster -n ns01
kubectl describe cluster workload-vsphere-vks1 -n ns01
```

### 8d. Deploy Tanzu Package(s) on Workload Cluster
Tanzu packages can be deployed from the Tanzu Standard repository using the Tanzu CLI from the Admin host. We will use `cert-manager` as an example to demonstrate the ease of deploying these packages. We must first add the package repository to the VKS Cluster to install the cert-manager.

Login to VKS Cluster using the kubectl-vsphere CLI from the Admin machine
```bash
## Login to VKS Cluster
kubectl vsphere login --vsphere-username <sso_username> --server=https://<SupervisorAPIEndpoint> --insecure-skip-tls-verify --tanzu-kubernetes-cluster-namespace <vsphere-namespace> --tanzu-kubernetes-cluster-name <tkc-name>

## Sample Command
kubectl vsphere login --vsphere-username administrator@vsphere.local --server=https://supervisor0.env1.lab.test --insecure-skip-tls-verify --tanzu-kubernetes-cluster-namespace ns01 --tanzu-kubernetes-cluster-name workload-vsphere-vks1
```

Add Package repository
```bash
## Command to add Package repository 
tanzu package repository add tanzu-standard --url <harbor-fqdn>/<project-name>/packages/standard/repo:version --namespace tkg-system

## Sample Command
tanzu package repository add tanzu-standard --url registry1.env1.lab.test/tanzu-packages/packages/standard/repo:v2024.5.16 --namespace tkg-system
```

Ensure that the package repository is reconciled successfully. 
```bash
tanzu package repository list -n tkg-system 

## Sample Output
  NAME            SOURCE                                                                  STATUS
  tanzu-standard  (imgpkg) registry1.env1.lab.test/tanzu-packages/packages/standard/repo  Reconcile succeeded

tanzu package repository list -n tkg-system

## Sample Output
  NAME                                            DISPLAY-NAME
  cert-manager.tanzu.vmware.com                   cert-manager
  cluster-autoscaler.tanzu.vmware.com             autoscaler
  contour.tanzu.vmware.com                        contour
  external-csi-snapshot-webhook.tanzu.vmware.com  external-csi-snapshot-webhook
  external-dns.tanzu.vmware.com                   external-dns
  fluent-bit.tanzu.vmware.com                     fluent-bit
  fluxcd-helm-controller.tanzu.vmware.com         Flux Helm Controller
  fluxcd-kustomize-controller.tanzu.vmware.com    Flux Kustomize Controller
  fluxcd-source-controller.tanzu.vmware.com       Flux Source Controller
  grafana.tanzu.vmware.com                        grafana
  harbor.tanzu.vmware.com                         harbor
  multus-cni.tanzu.vmware.com                     multus-cni
  prometheus.tanzu.vmware.com                     prometheus
  vsphere-pv-csi-webhook.tanzu.vmware.com         vsphere-pv-csi-webhook
  whereabouts.tanzu.vmware.com                    whereabouts
```

Install cert-manager using the below commands - 
```bash
## Command to List the versions of Cert-Manager Available
tanzu package available list cert-manager.tanzu.vmware.com -A

## Sample Output
  NAMESPACE   NAME                           VERSION                 RELEASED-AT
  tkg-system  cert-manager.tanzu.vmware.com  1.1.0+vmware.1-tkg.2    2020-11-24 18:00:00 +0000 UTC
  tkg-system  cert-manager.tanzu.vmware.com  1.1.0+vmware.2-tkg.1    2020-11-24 18:00:00 +0000 UTC
  tkg-system  cert-manager.tanzu.vmware.com  1.11.1+vmware.1-tkg.1   2023-01-11 12:00:00 +0000 UTC
  tkg-system  cert-manager.tanzu.vmware.com  1.12.10+vmware.2-tkg.2  2023-06-15 12:00:00 +0000 UTC
  tkg-system  cert-manager.tanzu.vmware.com  1.12.2+vmware.2-tkg.2   2023-06-15 12:00:00 +0000 UTC
  tkg-system  cert-manager.tanzu.vmware.com  1.5.3+vmware.2-tkg.1    2021-08-23 17:22:51 +0000 UTC
  tkg-system  cert-manager.tanzu.vmware.com  1.5.3+vmware.4-tkg.1    2021-08-23 17:22:51 +0000 UTC

## Command to install the cert-manager
tanzu package install cert-manager --package cert-manager.tanzu.vmware.com --namespace <namespaceName> --version <1.12.10+vmware.2-tkg.2>

## Sample Command
kubectl get pods -n cert-manager                                                                                                                                                                                      NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6778554f58-jhvb8              1/1     Running   0          54s
cert-manager-cainjector-575468965b-xrzf4   1/1     Running   0          54s
cert-manager-webhook-567f6945f-8m8d6       1/1     Running   0          54s
```

Note: In the sample commands provided, the cert-manager application will be deployed in the `namespaceName` namespace, with all required pods created in the `cert-manager` namespace. If a namespace named `cert-manager` already exists, the package deployment will use that existing namespace. If the package installation fails, label the `cert-manager` namespace with `pod-security.kubernetes.io/enforce=privileged` and delete all the ReplicaSets under the `cert-manager` namespace. This will prompt the deployment to recreate the ReplicaSets and the necessary pods.
