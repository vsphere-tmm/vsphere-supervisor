# Introduction

This document captures the troubleshooting resources for the VKS cluster (workload cluster formally known as TKGs cluster), which is part of our VMware Cloud Foundation (VCF) offering. It is based on the vSphere 8.0U3 and VMware Cloud Foundation 5.2 releases, and we heavily leverage our [existing troubleshooting documents](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/7-0/vsphere-with-tanzu-configuration-and-management-7-0/troubleshooting-vsphere-with-kubernetes.html) here. Checking release-specific documentation is recommended, given that some troubleshooting approaches and commands may change in future releases. 

# Workload cluster Troubleshooting

## Accessing Workload clusters

There are multiple approaches to accessing the workload clusters. Here are some of the most commonly used ones. 

### Accessing the VKS cluster using the `kubectl vsphere` plugin
You can connect to a VKS cluster using the vSphere Plugin for kubectl and authenticate with your vCenter Single Sign-On credentials.

```shell
$ kubectl vsphere login --server=<SUPERVISOR-CLUSTER-CONTROL-PLANE-IP> --tanzu-kubernetes-cluster-name <TANZU-KUBERNETES-CLUSTER-NAME> --tanzu-kubernetes-cluster-namespace <SUPERVISOR-NAMESPACE-CLUSTER-IS-DEPLOYED> --vsphere-username <VCENTER-SSO-USER-NAME>
```
Once logged in, you can access the cluster using the `kubectl` CLI. 

### Accessing the VKS cluster using KUBECONFIG
This method is used to access the VKS cluster if the SSO login is not working and users cannot access the cluster using the `kubectl` CLI. The user creates the `kubeconfig` file using the process below.

Log in to the Supervisor using the `kubectl vsphere` plugin.
```shell
kubectl vsphere login --server=<SUPERVISOR-CLUSTER-CONTROL-PLANE-IP> --vsphere-username <VCENTER-SSO-USER-NAME>
```

Execute the below commands to generate the kubeconfig commands.   
```shell

### Listing the secret in the vsphere workload namespace
$ kubectl get secret -n <namespace> | grep -i kubeconfig
indus-wl01-kubeconfig  cluster.x-k8s.io/secret    1  	22h

### Generating the kubeconfig file 
$ kubectl get secret indus-wl01-kubeconfig -n <namespace> -o json | jq -r '.data.value' | base64 -d >> /tmp/kubeconfig.conf
# exporting the kubeconfig
export KUBECONFIG=/tmp/kubeconfig.conf
```
You can now access the cluster using the `kubectl` CLI. 

### Accessing the VKS cluster using `vmware-system-user` user. 
This method is used to access the VKS cluster by directly logging into the VKS node. This may be required for troubleshooting the node or getting running kubectl commands directly from the node. 

**WARNING** This method provides elevated admin privileges, so you must exercise caution when using them. Only advanced users with expertise should perform cluster-level admin tasks and access node VMs. We highly recommend that you work with Broadcom Support on such functionalities.

Log in to the Supervisor using the kubectl vsphere plugin.
```shell
kubectl vsphere login --server=<SUPERVISOR-CLUSTER-CONTROL-PLANE-IP> --vsphere-username <VCENTER-SSO-USER-NAME>
```

Once logged in, you need to capture the 
* ssh-key for te VKS nodes and 
* The IP address of the nodes
  
```shell
Generate a SSH key file in the temporary directory:
$ kubectl get secret <guest-cluster-name>-ssh -o jsonpath='{.data.ssh-privatekey}' -n <namespace> | base64 -d > /tmp/<guest-cluster-name>-ssh-key
 
#Change the permissions to read only for the ssh-key file generated:
$ chmod 400 /tmp/<guest-cluster-name>-ssh-key
 
#Find the IP ADDR of the nodes in the guest cluster:
$ kubectl get vm -o wide -n <namespace>
```

Using the IP ADDRESS and the ssh private key, log in to the node.

```shell 
#Connect as vmware-system-user to the desired node's IP address retrieved above using the ssh-key file
$ ssh vmware-system-user@<IP ADDRESS> -i /tmp/<guest-cluster-name>-ssh-key
 
#Root privileges as necessary
$ sudo su
 #Enable kubectl commands against kube-apiserver
$ export KUBECONFIG=/etc/kubernetes/admin.conf
```

More detailed steps for accessing the Workload Control Plane Nodes [as a system user using a password](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/7-0/vsphere-with-tanzu-configuration-and-management-7-0/connecting-to-vsphere-with-tanzu-clusters/ssh-to-tanzu-kubernetes-cluster-nodes-as-the-system-user-using-a-password.html)

## Accessing the log files

TKG services run on the supervisor cluster and are crucial in the VKS cluster LCM process. This document contains the log locations for the below TKG services: 

* CAPI logs
* CAPV logs
* VM Operator logs
* TKG Controller Manager logs

 
# TKG Service Logs on supervisor node
$ /var/log/tkg-svs 

# API connectivity issues
$ less /var/log/cloud-init-output.log 

## General Troubleshooting commands
Once successfully logged into the VKS control plane node(s), as explained in the previous section, you can use the following generic commands for troubleshooting. Basic Linux commands such as df, ls, tail, top, and ps can also be used during troubleshooting. In addition to these commands, container-specific commands are available to help address any issues that may arise.

WARNING: Cluster admin/root privileges are elevated, so you must exercise caution when using them. Only advanced users with expertise should perform cluster-level admin tasks and access Supervisor node VMs. We highly recommend that you work with Broadcom Support on such functionalities.

### `crictl` CLI
Crictl is a command-line interface for container runtimes compatible with the Container Runtime Interface (CRI). It can inspect and debug container runtimes and applications on the supervisor node. The Crictl CLI is installed on the Supervisor node as part of the initial installation. Generic Crictl commands that are helpful during troubleshooting are:

```
# To check the images on the node 
$ crictl images

# To list the Pods on the node 
$ crictl pods 

# To list container(s) resource usage statistics
$ crictl stats

# To list the option available with crictl command line
$ crictl --help 
```

### `kubectl` CLI
The Kubectl tool is a command-line interface for interacting with a Kubernetes cluster. It simplifies tasks such as creating resources and managing cluster traffic. The Kubectl CLI is pre-installed on the supervisor node during installation. Generic Kubectl commands that are helpful during troubleshooting are:

```
# To list the non-running pods only
$ kubectl get pods -A | grep -v "Running"

# To list the events of the entire cluster 
$ kubectl get events
             
# To list events of a specific namespace
$ kubectl get events -n <namespace-name>

# To check the status of the nodes in a cluster
$ kubectl get nodes

# To check the error logs on the POD
$ kubectl logs <pod-name> -n <namespace>

# Pod Logs Bundle of a Namespace
$ kubectl cluster-info dump -n <namespace> --output directory=/PATH/NEWDIRECTORYNAME

# Check CPU & Memory by Node
$ kubectl top nodes
```

### etcdctl CLI
Etcdctl is a command-line tool for interacting with the etcd server. The API version used by etcdctl to speak to etcd may be set to version 2 or 3 via the ETCDCTL_API environment variable. By default, etcdctl on master(3.4) uses the v3 API, and earlier versions (3.3 and earlier) default to the v2 API. The etcdctl CLI is pre-installed on the supervisor node during installation.

```
# Setting the etcd environmental variable
$ export ETCDCTL_API=3

# Getting the etcd member list
$ etcdctl member list --write-out=table

# Health of the etcd cluster
$ etcdctl endpoint status --cluster -w table

# Get the etcd version
$ etcdctl --version

# Get the etcd help
$ etcdctl --help
```

### `ctr` CLI
ctr is a command-line client shipped as part of the containerd project. ctr is a tool for managing containers directly through the containerd runtime. The ctr CLI is pre-installed on the supervisor node during installation.

```
# Listing local images
$ ctr image ls

# Image pull from the registry
$ ctr image pull quay.io/quay/busybox:latest

# inspect the image 
$ ctr image export /tmp/nginx.tar docker.io/library/nginx:latest
```
## Metrics 
Prometheus can be used to collect metrics from VKS clusters. Prometheus is a system and service monitoring tool that collects metrics from configured targets at specified intervals. It evaluates rule expressions and displays the results. You can deploy Prometheus on your VKS cluster as a Tanzu package. Follow the link below to install [Prometheus Package](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/installing-standard-packages-on-tkg-service-clusters/installing-standard-packages-on-tkg-cluster-using-tkr-for-vsphere-8-x/install-prometheus-with-alertmanager.html)

The VKS clusters are CNCF-conformant Kubernetes clusters. They support the installation of standard observability tools such as Datadog, ELK, Telegraf and others.


## Logging
Fluent Bit can be used as a log forwarder for logs from a VKS cluster. You need to have a logging management server deployed to store and analyze the logs. To configure logging on the workload cluster, follow the link [Fluent Bit Package Reference](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/installing-standard-packages-on-tkg-service-clusters/installing-standard-packages-on-tkg-cluster-using-tkr-for-vsphere-8-x/install-fluent-bit.html).

The VKS clusters are CNCF-conformant Kubernetes clusters. They support log integration with standard solutions such as Syslog, HTTP, Elastic Search, Kafka, and Splunk.

## Appendix

### Collecting a Support Bundle for the VKS Cluster
You can use the TKC Support Bundler utility to collect TKG cluster log files and troubleshoot problems. To obtain and use the TKC Support Bundler utility, please take a look at the article [Gathering Logs for vSphere with Tanzu](https://kb.vmware.com/s/article/96617) at the VMware Support Knowledge Base.
To collect logs from Windows nodes, provision the Windows nodes with a built-in administrative account when building Windows node images. For information on how to build the Windows image with a custom answer file, see the [Provision Administrative Account for Log Collection](https://github.com/vmware-tanzu/vsphere-tanzu-kubernetes-grid-image-builder/blob/main/docs/windows.md#provision-administrative-account-for-log-collection) documentation.

### Namespace details on VKS Cluster
|Namespace Name|Usage|
|--------------|-----|
|secretgen-controller|Controller for secrets management|
|tkg-system|kapp controller for package managment|
|vmware-system-<antrea /calico>|Kubernetes CNI|
|vmware-system-auth|Guest Cluster Authentication|
|vmware-system-cloud-provider|Guest Cluster cloud provider|
|vmware-system-csi|vSphere CSI Controller and daemonset|

