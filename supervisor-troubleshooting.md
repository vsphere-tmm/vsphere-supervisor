# Introduction

This document captures the troubleshooting resources for the Supervisor control plane, which is part of our VMware Cloud Foundation (VCF) offering. It is based on the vSphere 8.0U3 and VMware Cloud Foundation 5.2 releases, and we heavily leverage our [existing troubleshooting documents](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/7-0/vsphere-with-tanzu-configuration-and-management-7-0/troubleshooting-vsphere-with-kubernetes.html) here. Checking release-specific documentation is recommended, given that some troubleshooting approaches and commands may change in future releases.  It also has a generic section with additional references.

## Supervisor Troubleshooting
This section focuses on troubleshooting Supervisor components. It specifically explains various approaches to logging in to supervisor cluster nodes, checking the status and health of components, and reviewing multiple events.

### Accessing Supervisor Cluster
There are numerous approaches to accessing the Supervisor cluster. Here are some of the most commonly used methods. 

### Accessing the Supervisor Cluster using CLI (Kubernetes API) 

To connect to the supervisor cluster, you need to follow the below steps:
* Download and install the [Kubernetes CLI tool for vSphere](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-installation-configuration/GUID-0F6E45C4-3CB1-4562-9370-686668519FCA.html)
* Configure the [secure login](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-installation-configuration/GUID-BF21BE11-3965-42E9-BBED-B6B784D97345.html)
* Connect to the supervisor cluster using the [vCenter single sign-on user](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-installation-configuration/GUID-F5114388-1838-4B3B-8A8D-4AE17F33526A.html)

```
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






