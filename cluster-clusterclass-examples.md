
You can use the example below as a guideline to build your VKS cluster manifest. The cluster spec could be broken down into two parts - 
* The general spec and the topology describing the control plane and worker node pools.
* The clusterclass variables spec that defines a list of key-value pairs that can be used to add additional configuration to the cluster. This section depends on the version of clusterclass that has been defined in the `spec.topology.class` section. Depending on the version being used, you can refer to the example provided in the relevant section.

Where possible, we have provided an exhaustive list of available variables. We have provided some default variables and commented out optional sections. To build a cluster manifest for your environment - 
* Copy the first yaml in this document (section 1),
* Modify the values as per your environment, 
* Depending on the cluster class version that will be used, copy the relevant clusterclass from section 2 and append it to the end of the first yaml.

Please refer to the official documentation for additional details. This has been tested with hvSphere 8.0U3+/VKS 3.4

## Section 1 

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: CLUSTER-NAME
  namespace: VSPHERE-NAMESPACE
spec:
  clusterNetwork:
    apiServerPort: 
    pods:
      cidrBlocks: ["240.0.0.0/20"] 
    serviceDomain: cluster.local
    services:
      cidrBlocks: ["240.1.0.0/20"]
#  paused: false
  topology:
    class: CLUSTER-CLASS                        # kubectl get clusterclass -n VSPHERE-NAMESPACE. See below for more info. 
#    classNamespace: vmware-system-vks-public 
    controlPlane:
#      machineHealthCheck:
#        enable: true
#        maxUnhealthy: 100%
#        nodeStartupTimeout: 4h0m0s
#        unhealthyConditions:
#        - status: Unknown
#          timeout: 5m0s
#          type: Ready
#        - status: False
#          timeout: 12m0s
#          type: Ready
#      metadata:
#        annotations:
#          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu
#          or
#          run.tanzu.vmware.com/resolve-os-image: "os-name=ubuntu,os-version=24.04" # If using Kubernetes 1.33
#        labels:
#          my-custom-label-key: my-custom-label-value
#      nodeDeletionTimeout: 10s
#      nodeDrainTimeout: 0
#      nodeVolumeDetachTimeout: 0
      replicas: 3                               # allowed values 1,3
#      variables:
#        overrides:                             # See Clusterclass variables for more details
#        - name: vmClass
#          value: best-effort-large
    version: VKR-VERSION                        # kubectl get kr
    workers:
      machineDeployments:
      - class: node-pool
#        failureDomain: <string>
#        machineHealthCheck:
#          enable: true
#          maxUnhealthy: 100%
#          nodeStartupTimeout: 4h0m0s
#          unhealthyConditions:
#          - status: Unknown
#            timeout: 5m0s
#            type: Ready
#          unhealthyRange: "[3-5]"
#        metadata:
#          annotations:
#            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "3"         # Use this for ClusterAutoscaler. 
#            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "0"         # Use this for ClusterAutoscaler. 
#            run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu 
#            or
#            run.tanzu.vmware.com/resolve-os-image: "os-name=ubuntu,os-version=24.04" # If using Kubernetes 1.33
#          labels:
#            my-custom-label-key: my-custom-label-value
#        minReadySeconds: 0
        name: node-pool-1
#        nodeDeletionTimeout: 10s
#        nodeDrainTimeout: 0
#        nodeVolumeDetachTimeout: 0
        replicas: 3
#        strategy:
#          remediation:
#            maxInFlight: 20%                   # See details for more info
#          rollingUpdate:
#            deletePolicy: Random               # Allowed values - "Random, "Newest", "Oldest"
#            maxSurge: 1
#            maxUnavailable: 0
#          type: RollingUpdate                  # Allowed values - RollingUpdate and OnDelete
#        variables:
#          overrides:
#          - name: vmClass
#            value: best-effort-large
    variables:                                  # Append the relevant Clusterclass from Section 2
```

## Section 2

### `class: builtin-generic-v3.4.0`

```yaml
      - name: kubernetes                                     # OPTIONAL, cluster-wide config
        value: 
          certificateRotation:                               # OPTIONAL
            enabled: true                                    # OPTIONAL, default: true
            renewalDaysBeforeExpiry: 90                      # OPTIONAL, default: 90, min: 7
          endpointFQDNs: ["demo.fqdn.com", "test.fqdn.com"]  # OPTIONAL, string, e.g., "k8s.prod.example.com"
          security:                                          # OPTIONAL
            podSecurityStandard:                             # OPTIONAL
              audit: "privilaged"                            # OPTIONAL, enum: "", privileged, baseline, restricted
              auditVersion: "latest"                         # OPTIONAL, e.g., "v1.31"
              enforce: "privilaged"                          # OPTIONAL, enum: "", privileged, baseline, restricted
              enforceVersion: "latest"                       # OPTIONAL, e.g., "v1.31"
              warn: "privilaged"                             # OPTIONAL, enum: "", privileged, baseline, restricted
              warnVersion: "latest"                          # OPTIONAL, e.g., "v1.31"
              deactivated: false                             # OPTIONAL, default: false
              exemptions:
                namespaces: []                               # OPTIONAL, string, namespace to exempt
      - name: node                                           # OPTIONAL, per-node config
        value: 
          labels:                                            # OPTIONAL, key: value map
            tenant: tenant-foo
            organization: engineering
            managed: ""
          taints:                                            # OPTIONAL, list of taints
            - key: key1                                      # REQUIRED for each taint
              value: value1                                  # REQUIRED for each taint
              effect: NoSchedule                             # REQUIRED for each taint, enum: NoSchedule, PreferNoSchedule, NoExecute
      - name: osConfiguration                                # OPTIONAL
        value: 
          directoryJoin:                                     # OPTIONAL, Windows only
            credentialSecretRef: ""                          # REQUIRED if directoryJoin is set
            domain: "my-ad.domain.com"                       # REQUIRED if directoryJoin is set
            gmsaControlSecurityGroupDN: ""                   # OPTIONAL
            organizationalUnitDN: ""                         # OPTIONAL
          fips:                                              # OPTIONAL
            enabled: false                                   # OPTIONAL, default: false
          ntp:                                               # OPTIONAL
            servers: ["ntp1.vmware.com", "ntp2.vmware.com"]  # REQUIRED if ntp is set IP/FQDN
          sshd:                                              # OPTIONAL
            banner: "SECURITY WANRNING...."                  # OPTIONAL, string, login banner
          systemProxy:                                       # OPTIONAL
            http: "http://1.2.3.4:2139"                      # REQUIRED if systemProxy is set, string
            https: "http://1.2.3.4:2139"                     # REQUIRED if systemProxy is set, string
            noProxy: ["no.proxy.test1", "no.proxy.test2"]    # REQUIRED if systemProxy is set, string
          trust:                                             # OPTIONAL
            additionalTrustedCAs:                            # REQUIRED if trust is set
              - caCert:
                  secretRef:                                 # OPTIONAL alternative
                    name: "trust-ca-secret-1"                # REQUIRED if secretRef used
                    key: "trust-ca-test1"                    # REQUIRED if secretRef used
              - caCert:
                  content: |-                                # OPTIONAL, inline PEM content
                    ------BEGIN CERTIFICATE-----
                    MII.....
                    EXJHDJKSDsd
                    MII.....
                    ------END CERTIFICATE-----
          ubuntuPro:
            tokenSecretRef: ""                               # REQUIRED if ubuntuPro is set
            services: []                                     # OPTIONAL, service names
            settings:
              - key: ""                                      # REQUIRED for each setting
                value: ""                                    # REQUIRED for each setting
          user:
            user: "vmware-system-user"                       # REQUIRED, string, e.g., "vmware-system-user"
            sshAuthorizedKey: "sshAuthorizedKeyTest..."      # OPTIONAL, must have at least sshAuthorizedKey or passwordSecret if user is set
            password:
              renewalDaysBeforeExpiry: 7                     # OPTIONAL, default 7
            passwordSecret:
              name: "user-secret-name"                       # REQUIRED if passwordSecret used
              key: "user-secret-key"                         # REQUIRED if passwordSecret used
      - name: resourceConfiguration                          # OPTIONAL
        value: 
          systemReserved:                                    # OPTIONAL
            automatic: false                                 # OPTIONAL, boolean, default: true
            cpu: "1"                                         # OPTIONAL, string, e.g., "1"
            memory: "1024Mi"                                 # OPTIONAL, string, e.g., "4096Mi"
      - name: storageClass                                   # REQUIRED, root volume StorageClass, e.g., "fast-ssd"
        value: "vsan-default-storage-policy"
      - name: vmClass                                        # REQUIRED, VMClass name for VM sizing, e.g., "best-effort-medium"
        value: "best-effort-medium"
      - name: volumes                                        # OPTIONAL, extra attached disks
        value: 
          - name: "containerd"                               # REQUIRED for each volume
            mountPath: "/var/lib/containerd"                 # REQUIRED
            storageClass: "vsan-default-storage-policy"      # REQUIRED
            capacity: 15Gi"                                  # REQUIRED, e.g., "20Gi"
          - name: "kubelet"                                  # REQUIRED for each volume
            mountPath: "/var/lib/kubelet"                    # REQUIRED
            storageClass: "vsan-default-storage-policy"      # REQUIRED
            capacity: 15Gi"                                  # REQUIRED, e.g., "20Gi"
      - name: vsphereOptions                                 # OPTIONAL, vSphere-specific overrides
        value: 
          persistentVolumes:
            availableStorageClasses: ["vsan-default-storage-policy", "vsan-default-storage-policy-1"]  # OPTIONAL
            availableVolumeSnapshotClasses: ["vol-snapclass-foo", "vol-snapclass-bar"]                 # OPTIONAL
            customizableStorageClassAnnotations: []                                                    # OPTIONAL
            customizableStorageClassLabels: []                                                         # OPTIONAL
            defaultStorageClass: "vsan-default-storage-policy"                                         # OPTIONAL, sets default
            defaultVolumeSnapshotClass: "vol-snapclass-foo"                                            # OPTIONAL
```

### `class: builtin-generic-v3.3.0`
```yaml
      - name: kubernetes                                     # OPTIONAL, cluster-wide config
        value: 
          certificateRotation:                               # OPTIONAL
            enabled: true                                    # OPTIONAL, default: true
            renewalDaysBeforeExpiry: 90                      # OPTIONAL, default: 90, min: 7
          endpointFQDNs: ["demo.fqdn.com", "test.fqdn.com"]  # OPTIONAL, string, e.g., "k8s.prod.example.com"
          security:                                          # OPTIONAL
            podSecurityStandard:                             # OPTIONAL
              audit: "privilaged"                            # OPTIONAL, enum: "", privileged, baseline, restricted
              auditVersion: "latest"                         # OPTIONAL, e.g., "v1.31"
              enforce: "privilaged"                          # OPTIONAL, enum: "", privileged, baseline, restricted
              enforceVersion: "latest"                       # OPTIONAL, e.g., "v1.31"
              warn: "privilaged"                             # OPTIONAL, enum: "", privileged, baseline, restricted
              warnVersion: "latest"                          # OPTIONAL, e.g., "v1.31"
              deactivated: false                             # OPTIONAL, default: false
              exemptions:
                namespaces: []                               # OPTIONAL, string, namespace to exempt
      - name: node                                           # OPTIONAL
        value: 
          labels:                                            # OPTIONAL, key: value map
            tenant: tenant-foo
            organization: engineering
            managed: ""
          taints:                                            # OPTIONAL, list of taints
            - key: key1                                      # REQUIRED for each taint
              value: value1                                  # REQUIRED for each taint
              effect: NoSchedule                             # REQUIRED for each taint, enum: NoSchedule, PreferNoSchedule, NoExecute
      - name: osConfiguration                                # OPTIONAL
        value: 
          directoryJoin:                                     # OPTIONAL, Windows only
            credentialSecretRef: ""                          # REQUIRED if directoryJoin is set
            domain: "my-ad.domain.com"                       # REQUIRED if directoryJoin is set
            gmsaControlSecurityGroupDN: ""                   # OPTIONAL
            organizationalUnitDN: ""                         # OPTIONAL
          fips:                                              # OPTIONAL
            enabled: false                                   # OPTIONAL, default: false
          ntp:                                               # OPTIONAL
            servers: ["ntp1.vmware.com", "ntp2.vmware.com"]  # REQUIRED if ntp is set IP/FQDN
          sshd:                                              # OPTIONAL
            banner: "SECURITY WANRNING...."                  # OPTIONAL, string, login banner
          systemProxy:                                       # OPTIONAL
            http: "http://1.2.3.4:2139"                      # REQUIRED if systemProxy is set, string
            https: "http://1.2.3.4:2139"                     # REQUIRED if systemProxy is set, string
            noProxy: ["no.proxy.test1", "no.proxy.test2"]    # REQUIRED if systemProxy is set, string
          trust:                                             # OPTIONAL
            additionalTrustedCAs:                            # REQUIRED if trust is set
              - caCert:
                  secretRef:                                 # OPTIONAL alternative
                    name: "trust-ca-secret-1"                # REQUIRED if secretRef used
                    key: "trust-ca-test1"                    # REQUIRED if secretRef used
              - caCert:
                  content: |-                                # OPTIONAL, inline PEM content
                    ------BEGIN CERTIFICATE-----
                    MII.....
                    EXJHDJKSDsd
                    MII.....
                    ------END CERTIFICATE-----
          ubuntuPro:                                         # OPTIONAL, Ubuntu nodes only
            tokenSecretRef: ""                               # REQUIRED if ubuntuPro is set
            services: []                                     # OPTIONAL, service names
            settings:
              - key: ""                                      # REQUIRED for each setting
                value: ""                                    # REQUIRED for each setting
          user:                                              # OPTIONAL
            user: "vmware-system-user"                       # REQUIRED, string, e.g., "vmware-system-user"
            sshAuthorizedKey: "sshAuthorizedKeyTest..."      # OPTIONAL, must have at least sshAuthorizedKey or passwordSecret if user is set
            passwordSecret:
              name: "user-secret-name"                       # REQUIRED if passwordSecret used
              key: "user-secret-key"                         # REQUIRED if passwordSecret used
      - name: resourceConfiguration                          # OPTIONAL
        value: 
          systemReserved:                                    # OPTIONAL
            automatic: false                                 # OPTIONAL, boolean, default: true
            cpu: "1"                                         # OPTIONAL, string, e.g., "1"
            memory: "1024Mi"                                 # OPTIONAL, string, e.g., "4096Mi"
      - name: storageClass                                   # REQUIRED, string, e.g., "fast-ssd"
        value: "vsan-default-storage-policy"
      - name: vmClass                                        # REQUIRED, string, e.g., "best-effort-xsmall"
        value: "best-effort-medium"
      - name: volumes                                        # OPTIONAL
        value: 
          - name: "containerd"                               # REQUIRED for each volume
            mountPath: "/var/lib/containerd"                 # REQUIRED
            storageClass: "vsan-default-storage-policy"      # REQUIRED
            capacity: 15Gi"                                  # REQUIRED, e.g., "20Gi"
          - name: "kubelet"                                  # REQUIRED for each volume
            mountPath: "/var/lib/kubelet"                    # REQUIRED
            storageClass: "vsan-default-storage-policy"      # REQUIRED
            capacity: 15Gi"                                  # REQUIRED, e.g., "20Gi"
      - name: vsphereOptions                                 # OPTIONAL
        value: 
          persistentVolumes:                                 # OPTIONAL
            availableStorageClasses: ["vsan-default-storage-policy", "vsan-default-storage-policy-1"]  # OPTIONAL, string
            availableVolumeSnapshotClasses: ["vol-snapclass-foo", "vol-snapclass-bar"]                 # OPTIONAL, string
            defaultStorageClass: "vsan-default-storage-policy"                                         # OPTIONAL, string
            defaultVolumeSnapshotClass: "vol-snapclass-foo"                                            # OPTIONAL, string
```

### `class: builtin-generic-v3.2.0`

```yaml
      - name: kubernetes                                     # OPTIONAL
        value: 
          certificateRotation:                               # OPTIONAL
            enabled: true                                    # OPTIONAL, boolean, default: true
            renewalDaysBeforeExpiry: 90                      # OPTIONAL, int, default: 90, min: 7
          endpointFQDNs: ["demo.fqdn.com", "test.fqdn.com"]  # OPTIONAL, string, FQDN alias for control plane
          security:                                          # OPTIONAL
            podSecurityStandard:                             # OPTIONAL
              audit: privileged                              # OPTIONAL, enum: "", privileged, baseline, restricted
              auditVersion: latest                           # OPTIONAL, string, e.g., "v1.31"
              enforce: privileged                            # OPTIONAL, enum: "", privileged, baseline, restricted
              enforceVersion: latest                         # OPTIONAL, string
              warn: privileged                               # OPTIONAL, enum: "", privileged, baseline, restricted
              warnVersion: latest                            # OPTIONAL, string
              deactivated: false                             # OPTIONAL, boolean, default: false
              exemptions:                                    # OPTIONAL
                namespaces: []                               # OPTIONAL, string, namespace to exempt
      - name: node                                           # OPTIONAL
        value: 
          labels:                                            # OPTIONAL, map of key: value
            tenant: tenant-foo
            organization: engineering
            managed: ""
          taints:                                            # OPTIONAL
            - key: key1                                      # REQUIRED for each taint
              value: value1                                  # REQUIRED for each taint
              effect: NoSchedule                             # REQUIRED for each taint, enum: NoSchedule, PreferNoSchedule, NoExecute
      - name: osConfiguration                                # OPTIONAL
        value: 
          ntp:                                               # OPTIONAL
            servers: ["ntp1.vmware.com", "ntp2.vmware.com"]  # REQUIRED if ntp is set
          systemProxy:                                       # OPTIONAL
            http: "http://1.2.3.4:2139"                      # REQUIRED if systemProxy is set, string
            https: "http://1.2.3.4:2139"                     # REQUIRED if systemProxy is set, string
            noProxy: ["no.proxy.test1", "no.proxy.test2"]    # REQUIRED if systemProxy is set, string
          trust:                                             # OPTIONAL
            additionalTrustedCAs:                            # REQUIRED if trust is set
              - caCert:
                  secretRef:                                 # OPTIONAL alternative
                    name: "trust-ca-secret-1"                # REQUIRED if secretRef used
                    key: "trust-ca-test1"                    # REQUIRED if secretRef used. Value needs to be double base64 encoded.
              - caCert:
                  content: |-                                # OPTIONAL, inline PEM content
                    ------BEGIN CERTIFICATE-----
                    MII.....
                    EXJHDJKSDsd
                    MII.....
                    ------END CERTIFICATE-----
          user:                                              # OPTIONAL
            user: "vmware-system-user"                       # REQUIRED, string, e.g., "vmware-system-user"
            sshAuthorizedKey: "sshAuthorizedKeyTest..."      # OPTIONAL, must have at least sshAuthorizedKey or passwordSecret if user is set
            passwordSecret:
              name: "user-secret-name"                       # REQUIRED if passwordSecret used
              key: "user-secret-key"                         # REQUIRED if passwordSecret used
      - name: resourceConfiguration                          # OPTIONAL
        value: 
          systemReserved:                                    # OPTIONAL
            automatic: false                                 # OPTIONAL, boolean, default: true
            cpu: "1"                                         # OPTIONAL, string, e.g., "1"
            memory: "1024Mi"                                 # OPTIONAL, string, e.g., "4096Mi"
      - name: storageClass                                   # REQUIRED, string, root disk storage class
        value: "vsan-default-storage-policy"
      - name: vmClass                                        # REQUIRED, string, VM class for compute shape
        value: "best-effort-medium"
      - name: volumes                                        # OPTIONAL
        value: 
          - name: "containerd"                               # REQUIRED for each volume
            mountPath: "/var/lib/containerd"                 # REQUIRED
            storageClass: "vsan-default-storage-policy"      # REQUIRED
            capacity: 15Gi"                                  # REQUIRED, e.g., "20Gi"
          - name: "kubelet"                                  # REQUIRED for each volume
            mountPath: "/var/lib/kubelet"                    # REQUIRED
            storageClass: "vsan-default-storage-policy"      # REQUIRED
            capacity: 15Gi"                                  # REQUIRED, e.g., "20Gi"
      - name: vsphereOptions                                 # OPTIONAL
        value: 
          persistentVolumes:                                                                           # OPTIONAL
            availableStorageClasses: ["vsan-default-storage-policy","vsan-default-storage-policy-1" ]  # OPTIONAL, string
            availableVolumeSnapshotClasses: ["vol-snapclass-foo", "vol-snapclass-bar"]                 # OPTIONAL, string
            defaultStorageClass: "vsan-default-storage-policy"                                         # OPTIONAL, string
            defaultVolumeSnapshotClass: "vol-snapclass-foo"                                            # OPTIONAL, string
```

### `class: tanzukubernetescluster` or `class: builtin-generic-v3.1.0`

```yaml
      - name: clusterEncryptionConfigYaml                    # OPTIONAL
        value: |                                             # REQUIRED, Inline YAML string
          apiVersion: apiserver.config.k8s.io/v1
          kind: EncryptionConfiguration
          resources:
            - resources:
                - secrets
              providers:
                - aescbc:
                    keys:
                      - name: key1
                        secret: QiMgJGYXudtljldVyl+AnXQQlk7r9iUXBfVKqdEfKm8=
                - identity: {}
      - name: controlPlaneCertificateRotation                # OPTIONAL
        value:
          activate: true                                     # OPTIONAL, boolean, default: true
          daysBefore: 90                                     # OPTIONAL, integer, default: 90, minimum: 7
      - name: controlPlaneVolumes                            # OPTIONAL
        value:
        - name: "containerd"                                 # OPTIONAL, string
          mountPath: "/var/lib/containerd"                   # OPTIONAL, string
          storageClass: "vsan-default-storage-policy"        # OPTIONAL, string
          capacity:
            storage: "15Gi"                                  # OPTIONAL, string
        - name: "kubelet"                                    # OPTIONAL, string
          mountPath: "/var/lib/kubelet"                      # OPTIONAL, string
          storageClass: "vsan-default-storage-policy"        # OPTIONAL, string
          capacity:
            storage: "15Gi"                                  # OPTIONAL, string
      - name: defaultRegistrySecret                          # OPTIONAL
        value: 
          name: ""                                           # OPTIONAL, string
          namespace: ""                                      # OPTIONAL, string
          data: ""                                           # OPTIONAL, string
      - name: defaultStorageClass                            # OPTIONAL, string
        value: "vsan-default-storage-policy"
      - name: defaultVolumeSnapshotClass                     # OPTIONAL, string
        value: "vol-snapclass-foo"
      - name: kubeAPIServerFQDNs                             # OPTIONAL
        value: ["demo.fqdn.com", "test.fqdn.com"]            # OPTIONAL, string
      - name: nodePoolLabels                                 # OPTIONAL
        value:
        - key: "tenant"                                      # OPTIONAL, string
          value: "tenant-foo"                                # OPTIONAL, string
        - key: "organization"                                # OPTIONAL, string
          value: "engineering"                               # OPTIONAL, string
        - key: "managed"                                     # OPTIONAL, string
          value: ""                                          # OPTIONAL, string
      - name: nodePoolTaints                                 # OPTIONAL
        value: 
        - key: "key1"                                        # OPTIONAL, string
          value: "value1"                                    # OPTIONAL, string
          effect: "NoSchedule"                               # OPTIONAL, string
          timeAdded: 0                                       # OPTIONAL, integer (epoch)
      - name: nodePoolVolumes                                # OPTIONAL
        value: 
        - name: "containerd"                                 # OPTIONAL, string
          mountPath: "/var/lib/containerd"                   # OPTIONAL, string
          storageClass: "vsan-default-storage-policy"        # OPTIONAL, string
          capacity:
            storage: "15Gi"                                  # OPTIONAL, string
        - name: "kubelet"                                    # OPTIONAL, string
          mountPath: "/var/lib/kubelet"                      # OPTIONAL, string
          storageClass: "vsan-default-storage-policy"        # OPTIONAL, string
          capacity:
            storage: "15Gi"                                  # OPTIONAL, string
      - name: ntp                                            # OPTIONAL, string
        value: "ntp.vmware.com"
      - name: podSecurityStandard                            # OPTIONAL
        value: 
          audit: "restricted"                                # OPTIONAL, enum: "", privileged, baseline, restricted
          auditVersion: "latest"                             # OPTIONAL, string
          enforce: "restricted"                              # OPTIONAL, enum: "", privileged, baseline, restricted
          enforceVersion: "latest"                           # OPTIONAL, string
          warn: "restricted"                                 # OPTIONAL, enum: "", privileged, baseline, restricted
          warnVersion: "latest"                              # OPTIONAL, string
          deactivated: false                                 # OPTIONAL, boolean
          exemptions:
            namespaces: []                                   # OPTIONAL, string
      - name: proxy                                          # OPTIONAL
        value: 
          httpProxy: "http://1.2.3.4:2139"                   # OPTIONAL, string
          httpsProxy: "http://1.2.3.4:2139"                  # OPTIONAL, string
          noProxy: ["no.proxy.test1", "no.proxy.test2"]      # OPTIONAL, string
      - name: storageClass                                   # REQUIRED, string, must be set
        value: "vsan-default-storage-policy"
      - name: storageClasses                                 # OPTIONAL
        value: ["vsan-default-storage-policy","vsan-default-storage-policy-1"]  # OPTIONAL, string
      - name: trust                                          # OPTIONAL
        value: 
          additionalTrustedCAs:
            - name: "harbor-ca-1"                            # OPTIONAL, string
      - name: user                                           # OPTIONAL
        value: 
          sshAuthorizedKey: "sshAuthorizedKeyTest..."        # OPTIONAL, string
          passwordSecret:
            name: "user-secret-name"                         # OPTIONAL, string
            key: "user-secret-key"                           # OPTIONAL, string
      - name: vmClass                                        # REQUIRED, string, must be set
        value: "best-effort-medium"
      - name: volumeSnapshotClasses                          # OPTIONAL
        value: ["vol-snapclass-foo", "vol-snapclass-bar"]    # OPTIONAL, string
```
