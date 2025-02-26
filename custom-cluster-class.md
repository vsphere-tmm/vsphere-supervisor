# Utilizing the Custom Cluster Class to personalize VKS cluster deployment
A custom ClusterClass allows you to tailor cluster configurations to fit specific needs while leveraging standardization's benefits. Here’s why it matters:
1. Organization-Specific Standardization:
A default ClusterClass might not fit your infrastructure, security, or networking requirements. A custom ClusterClass ensures that clusters adhere to your organization’s policies and best practices.
2. Flexibility While Maintaining Consistency: 
Instead of using a generic template, a custom ClusterClass lets you predefine control plane and node pool customizations and networking setups while allowing minor variations.
3. Preconfigured Security & Compliance:
You can embed security best practices directly into the ClusterClass.

The purpose of this document is to guide you through ways you can leverage Custom ClusterClass. Please note that Custom ClusterClass is **not supported** by Broadcom, and improper configurations/use may lead to service disruptions and/or outages. 

This document is specific to versioned ClusterClass (TKG Service 3.1.x and higher). For TKG Service 3.0 and lower, please review the official documentation.

## Basic operations

### Clone an existing ClusterClass 
In this example, we will clone an existing `builtin-generic-v3.1.0` ClusterClass in the `demo1` vSphere Namespace and customize it to deploy a VKS cluster.

```
$ kubectl get clusterclass -n demo1                                        
NAME                     AGE
builtin-generic-v3.1.0   4d16h
builtin-generic-v3.2.0   4d16h
builtin-generic-v3.3.0   4d16h
tanzukubernetescluster   4d17h
```

```
$ kubectl get clusterclass builtin-generic-v3.1.0 -o yaml -n demo1 > /tmp/ccc.yaml
```

Edit the output and modify the `metadata` section to 
- remove `metadata.creationTimestamp`, `metadata.labels`, `metadata.resourceVersion` and `metadata.uid`
- modify the `metadata.name` to provide a new name
- **do not modify** the `run.tanzu.vmware.com/resolve-tkr: ""` annotation.

The metadata section should look something similar to the one below -

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  annotations:
    run.tanzu.vmware.com/resolve-tkr: ""
  generation: 1
  name: ccc-v3.1.0
  namespace: demo1
spec:
  controlPlane:
    machineHealthCheck:
      maxUnhealthy: 100%
      nodeStartupTimeout: 2h0m0s
...
```
Now, you are ready to modify the `spec` section for various customizations of the nodes and cluster settings. Some of these settings/customizations will be discussed in the following sections. Once the desired new customizations have been made to the file, save the file (`/tmp/ccc.yaml` in this example). 

### Create and Consume the new Custom ClusterClass 
To create a new Custom ClusterClass in the `demo1` namespace - 
```
$ kubectl apply -f /tmp/ccc.yaml -n demo1
```

To create a new VKS cluster using this newly designed custom ClusterClass, within the Cluster definition manifest, reference the name of the freshly created ClusterClass - 

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: workload-vsphere-tkg2
spec:
...
 topology:
    class: ccc-v3.1.0
    version: v1.31.4---vmware.1-fips-vkr.3
...
```

## Control plane node customization
### Adding custom files

```yaml
spec:
  ...
  patches:
  ...
  ### Add a new or update an existing patch
  - name: my-awesome-patch
    definitions:
    - selector:
        apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
        kind: KubeadmControlPlaneTemplate
        matchResources:
          controlPlane: true
      jsonPatches:
        - op: add
          path: /spec/template/spec/kubeadmConfigSpec/files
          valueFrom:
            template: |
              - content: |
                  Lorem ipsum dolor sit amet, consectetur adipiscing elit...                
                owner: root:root
                path: /path/to/custom/script
                permissions: "0644"
...
```
### Executing custom commands

```yaml
spec:
  ...
  patches:
  ...
  ### Add a new or update an existing patch
  - name: my-awesome-patch
    definitions:
    - selector:
        apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
        kind: KubeadmControlPlaneTemplate
        matchResources:
          controlPlane: true
      jsonPatches:
        - op: add
          path: /spec/template/spec/kubeadmConfigSpec/postKubeadmCommands/-
          value: "sh /path/to/custom/script"
...
```
## Worker node customization
### Adding custom files

```yaml
spec:
  ...
  patches:
  ...
  ### Add a new or update an existing patch
  - name: my-awesome-patch
    definitions:
    - selector:
        apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
        kind: KubeadmConfigTemplate
        matchResources:
          machineDeploymentClass:
            names:
              - node-pool
      jsonPatches:
        - op: add
          path: /spec/template/spec/files
          valueFrom:
            template: |
              - content: |
                  Lorem ipsum dolor sit amet, consectetur adipiscing elit...                
                owner: root:root
                path: /path/to/custom/script
                permissions: "0644"
...
```
### Executing custom commands

```yaml
spec:
  ...
  patches:
  ...
  ### Add a new or update an existing patch
  - name: my-awesome-patch
    definitions:
    - selector:
        apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
        kind: KubeadmConfigTemplate
        matchResources:
          machineDeploymentClass:
            names:
              - node-pool
      jsonPatches:
        - op: add
          path: /spec/template/spec/postKubeadmCommands/-
          value: "sh /path/to/custom/script"
...
```
