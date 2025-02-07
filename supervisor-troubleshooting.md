# Introduction

This document captures the troubleshooting resources for the Supervisor control plane, which is part of our VMware Cloud Foundation (VCF) offering. It is based on the vSphere 8.0U3 and VMware Cloud Foundation 5.2 releases, and we heavily leverage our [existing troubleshooting documents](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/7-0/vsphere-with-tanzu-configuration-and-management-7-0/troubleshooting-vsphere-with-kubernetes.html) here. Checking release-specific documentation is recommended, given that some troubleshooting approaches and commands may change in future releases.  It also has a generic section with additional references.

# Supervisor Troubleshooting
This section focuses on troubleshooting Supervisor components. It specifically explains various approaches to logging in to supervisor cluster nodes, checking the status and health of components, and reviewing multiple events.

## Accessing Supervisor Cluster
There are numerous approaches to accessing the Supervisor cluster. Here are some of the most commonly used methods. 

### Accessing the Supervisor Cluster using CLI (Kubernetes API) 

To connect to the supervisor cluster, you need to follow the below steps:
* Download and install the [Kubernetes CLI tool for vSphere](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-installation-configuration/GUID-0F6E45C4-3CB1-4562-9370-686668519FCA.html)
* Configure the [secure login](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-installation-configuration/GUID-BF21BE11-3965-42E9-BBED-B6B784D97345.html)
* Connect to the supervisor cluster using the [vCenter single sign-on user](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-installation-configuration/GUID-F5114388-1838-4B3B-8A8D-4AE17F33526A.html)

```shell
kubectl vsphere login --server=<kubernetes-control-plane-ip-address>  --vsphere-username <VCENTER-SSO-USER>
```

### Accessing the Supervisor Control Plane Node

When you log in to the supervisor control plane using the `kubectl` utility, you don't automatically get the `cluster-admin` level privileges to perform the operations due to RBAC limitations.  You must log in to the Supervisor Control Plane VMs to perform the cluster-admin level task.  

**WARNING** Cluster admin privileges are elevated, so you must exercise caution when using them. Only advanced users with expertise should perform cluster-level admin tasks and access Supervisor node VMs. We highly recommend that you work with Broadcom Support on such functionalities. 

Follow the below steps to acquire the root credentials and log in to the supervisor node - 

Log in to the vCenter using root credentials to get the Supervisor Control Plane nodes' floating IP and password. 
* Establish an SSH connection to the vCenter Server Appliance.
* Login in as the root user.
* Run the command shell.
* Run the command `/usr/lib/vmware-wcp/decryptK8Pwd.py`

```shell 
$ /usr/lib/vmware-wcp/decryptK8Pwd.py
Read key from file

Connected to PSQL

Cluster: domain-c1007:c1bce180-df14-4e67-b608-ce9b37d83937
IP: 172.16.40.10
PWD: xH!i^Hew_w$C{w9B
------------------------------------------------------------

root@vcenter [ ~ ]# ssh root@172.16.40.10 ### IP DISPLAYED ABOVE

##### Use the PWD DISPLAYED ABOVE

(root@172.16.40.10) Password:
Last login: Tue Nov 12 13:20:46 2024 from 192.168.41.1
 16:13:13 up 1 day,  7:44,  0 user,  load average: 2.40, 2.86, 2.72
tdnf update info not available yet!
root@42215278add0463992c19c4a1ddd69c7 [ ~ ]#
```

### Accessing the Supervisor Agent Nodes
ESXi nodes perform the role of Kubernetes Agents (worker nodes) in a Supervisor Cluster. Please log in to the ESXi node to troubleshoot the agents. 

## Determine Cluster Health and Service Status 

### WCP Service Status
Workload Control Plane (WCP) is a service that runs within the vCenter appliance and manages the desired state and configuration of the Supervisor and related services. We can check the service status using the CLI and from the vCenter VAMI console UI. 

To check the status of the WCP service and to restart the service using the CLI, run the below command:
```shell
## Login to the vCenter Appliance
$ service-control --status wcp ## (Check the status)
$ service-control --stop   wcp ## (Stop the service)
$ service-control --start  wcp ## (Start the service)
```

To check the status of WCP service using the vCenter VAMI console, login to the vCenter VAMI console at **https://<vcenter-ip-or-fqdn>:5480**

Click on services → Select the service Workload Control Plane → Click on restart. (if needed) 

## Accessing logs

### WCP Service Logs
To check the logs of the WCP service, follow the below steps:
* Establish an SSH connection to the vCenter Server Appliance.
* Log in as the root user.
* Run the following command to tail the logs.

```shell
$ less /var/log/vmware/wcp/wcpsvc.log
```

### Supervisor Logs
Login to Supervisor Control Plane VM. See the above section (Accessing the Supervisor Agent Nodes) for logging in to the nodes. 

**WARNING** Cluster admin/root privileges are elevated, so you must exercise caution when using them. Only advanced users with expertise should perform cluster-level admin tasks and access Supervisor node VMs. We highly recommend that you work with Broadcom Support on such functionalities. 

```shell
# API connectivity issues
$ less /var/log/cloud-init-output.log  

# kube-apiserver is running, but kubectl commands are still failing
$ less /var/log/pods/

# Troubleshooting stuck/failed provisioning of a CP VM
$ less /var/log/vmware-imc

# Certificate location on the Supervisor
$ cd /etc/vmware/tls/
```

### Spherelet Log Location & Status
In vSphere, a Spherelet is an ESXi UserWorld agent that allows an ESXi hypervisor to act as a Kubernetes worker node. This is our implementation of Kubelet in vSphere Kubernetes Service. Spherelet agents are installed as Virtual Infrastructure Bundles (VIBs) on the ESXi nodes to support the overall objective of the Workload Control Plane (WCP), which is to run PODs as virtual machines (VMs) on these nodes. If you encounter issues while running the vSphere PODs, you can examine the Spherelet logs using the following method:

```shell
### SSH into the ESXi node that needs attention
ssh root@<ip-address-esxi-node>

### To view the logs, view the following logs
$ tail /var/log/spherelet.log

### To check the status of the Spherelet on the ESXi node, run the command below.
$ esxcli software vib list | grep spherelet
$ /etc/init.d/spherelet status
```

## General Troubleshooting commands
Once successfully logged into the Supervisor control plane node(s), as explained in the previous section, you can use the following generic commands for troubleshooting. Basic Linux commands such as `df,` `ls,` `tail,` `top,` and `ps` can also be used during troubleshooting. In addition to these commands, container-specific commands are available to help address any issues that may arise.

**WARNING** Cluster admin/root privileges are elevated, so you must exercise caution when using them. Only advanced users with expertise should perform cluster-level admin tasks and access Supervisor node VMs. We highly recommend that you work with Broadcom Support on such functionalities. 

### `crictl` CLI
`Crictl` is a command-line interface for container runtimes compatible with the Container Runtime Interface (CRI). It can inspect and debug container runtimes and applications on the supervisor node. The `Crictl` CLI is installed on the Supervisor node as part of the initial installation. Generic `Crictl` commands that are helpful during troubleshooting are:
```shell
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
The `Kubectl` tool is a command-line interface for interacting with a Kubernetes cluster. It simplifies tasks such as creating resources and managing cluster traffic. The `Kubectl` CLI is pre-installed on the supervisor node during installation. Generic `Kubectl` commands that are helpful during troubleshooting are: 
```shell
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

# Check CPU & Memeory by Node
$ kubectl top nodes
```

### `etcdctl` CLI
`Etcdctl` is a command-line tool for interacting with the `etcd` server. The API version used by `etcdctl` to speak to `etcd` may be set to version 2 or 3 via the `ETCDCTL_API` environment variable. By default, etcdctl on master(3.4) uses the v3 API, and earlier versions (3.3 and earlier) default to the v2 API. The `etcdctl` CLI is pre-installed on the supervisor node during installation.

```shell
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
`ctr` is a command-line client shipped as part of the containerd project. `ctr` is a tool for managing containers directly through the containerd runtime. The `ctr` CLI is pre-installed on the supervisor node during installation.

```shell
# Listing local images
$ ctr image ls

# Image pull from the registry
$ ctr image pull quay.io/quay/busybox:latest

# inspect the image 
$ ctr image export /tmp/nginx.tar docker.io/library/nginx:latest
```

## Supervisor Metrics Collection
VMware uses Telegraf for metrics collection. Telegraf comes preinstalled on the Supervisor as a daemonset in the `vmware-system-monitoring` namespace. Telegraf is a server-based agent that collects and sends metrics from different systems, databases, and IoT. Each Supervisor component exposes an endpoint to which Telegraf connects. Telegraf then sends the collected metrics to an observability platform of the choice.

```shell
$ kubectl get pods -A | grep -i telegraf
vmware-system-monitoring    telegraf-9ct6r   2/2  Running  0  21h
vmware-system-monitoring    telegraf-bzqbd   2/2  Running  0  21h
vmware-system-monitoring    telegraf-dtqr6   2/2  Running  0  21h                                              
```

The `default-telegraf-config` ConfigMap holds the default Telegraf configuration and is **read-only**. We can use it as a fallback option to restore the configuration in `telegraf-config` if the file is corrupt or you want to restore it to the defaults. The only ConfigMap that you can edit is telegraf-config.

```shell
$ kubectl get cm -n vmware-system-monitoring
NAME                  	DATA   AGE
default-telegraf-config 1  	21h
kube-rbac-proxy-config	1  	21h
kube-root-ca.crt      	1  	21h
telegraf-config       	1  	21h
```

## Supervisor Log Collection
Fluent Bit is an open-source log shipper multi-platform for log processing and distribution. A log shipper is a tool that gathers logs from various sources, like containers and servers, and directs them to a central location for analysis. Fluent Bit pods come preinstalled on the Supervisor as a daemonset in the `vmware-system-logging` namespace. 

```shell
$ kubectl get pods -n vmware-system-logging
NAME          	READY   STATUS	RESTARTS   AGE
fluentbit-c5cmq   1/1 	Running   0      	31h
fluentbit-krpq6   1/1 	Running   0      	31h
fluentbit-zxckz   1/1 	Running   0      	31h
```

vSphere administrator can use Fluent Bit to:
* Forward Supervisor control plane logs and system journal logs to major external log monitoring platforms such as Loki, Elastic Search, Grafana, and other platforms supported by Fluent Bit
* Update or reset the log forwarding configuration for the Supervisor control plane via the Kubernetes API.

Fluent bit stores its configuration in the `fluentbit-config-custom` ConfigMap under the `vmware-system-logging` namespace that vSphere administrators can edit to configure log forwarding to external platforms by defining log servers.

```shell
$ kubectl get cm -n vmware-system-logging
NAME                  	DATA   AGE
fluentbit-config      	  1  	 32h
fluentbit-config-custom   4  	 32h
fluentbit-config-system   4  	 32h
kube-root-ca.crt      	  1  	 32h
```

Using Fluent Bit, you can configure the forwarding of Supervisor control plane logs to external monitoring systems, such as Grafana Loki or Elastic Search. 

You can also perform Streaming Logs of the Supervisor Control Plane to a Remote rsyslog. The remote rsyslog server is responsible for streaming logs from the Supervisor control plane virtual machines (VMs) to a remote rsyslog receiver. This setup helps ensure that valuable logging data is not lost. On the other hand, Fluent Bit serves as a log shipper for both Kubernetes service logging and application logging, which runs as pods within Kubernetes clusters.

## Appendix

### Collecting a Support Bundle for Supervisor
You can export the supervisor logs to troubleshoot Supervisor and VKS cluster errors. Typically, such logs are reviewed in consultation with VMware Support.
* Log in to your vSphere IaaS control plane environment using the vSphere Client.
* Select Menu > Workload Management.
* Select the Supervisor tab.
* Select the target Supervisor instance.
* Click `Export Logs`.

Refer [Gathering Logs for vSphere with Tanzu](https://kb.vmware.com/s/article/96617) for more information.


### Namespace details on Supervisor

Below is the list of namespaces created on the Supervisor node after the clusters are successfully installed. These are the default namespaces on Kubernetes version 1.29. 

|Namespace Name|NSX-T / VDS|Usage|Where is the component running|
| :----------- | :-------: | :-- | :--------------------------: |
|svc-tkg-domain-cxxxxx|NSX / VDS|TKG Supervisor Svc for VKS cluster LCM|Controlplane|
|svc-tmc-cxxxxx|NSX / VDS|TMC agent installer and other TMC agents|Controlplane|
|svc-velero-domain-cxxxxx|NSX / VDS|Velero Operator Supervisor Svc for VKS Velero LCM|Controlplane|
|vmware-system-ako|VDS|AVI AKO operator for Supervisor load balancing|Controlplane|
|vmware-system-appplatform-operator-system|NSX / VDS|App Platform Controllers for Supervisor Service LCM|Controlplane|
|vmware-system-cert-manager|NSX / VDS|Cert-Manager for Supervisor Certificate management|Controlplane|
|vmware-system-csi|NSX / VDS|vSphere CSI controller|Controlplane|
|vmware-system-imageregistry|NSX / VDS|Image registry for Supervisor container images|Controlplane|
|vmware-system-kubeimage|NSX / VDS|Image controller for PodVM image LCM|Controlplane|
|vmware-system-license-operator|NSX / VDS||Controlplane|
|vmware-system-logging|NSX / VDS|Fluentbit daemonset for log processing and forwarding|Controlplane|
|vmware-system-monitoring|NSX / VDS|Telegraf daemonset for metrics aggregation and forwarding|Controlplane|
|vmware-system-netop|NSX / VDS|Netop Controller for IP pools, LB, and other network object management|Controlplane|
|vmware-system-nsop|NSX / VDS|vSphere Namespace Controller|Controlplane|
|vmware-system-pinniped|NSX / VDS|Pinniped for authentication|Controlplane|
|vmware-system-vmop|NSX / VDS|VM Operator|Controlplane|
|svc-[GENERIC SUPERVISOR SERVICE]-domain-cxxxx|NSX / VDS|Any generic supervisor service|Agent|


