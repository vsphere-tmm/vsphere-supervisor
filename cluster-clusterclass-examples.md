
You can use the example below as a guideline to build your VKS cluster manifest. The cluster spec could be broken down into two parts - 
* The general spec and the topology describing the control plane and worker node pools.
* The clusterclass variables spec that defines a list of key-value pairs that can be used to add additional configuration to the cluster. This section depends on the version of clusterclass that has been defined in the `spec.topology.class` section. Depending on the version being used, you can refer to the example provided in the relevant section.

Where possible, we have provided an exhaustive list of available variables. We have provided some default variables and commented out optional sections. To build a cluster manifest for your environment - 
* Copy the first yaml in this document,
* Modify the values as per your environment, 
* Depending on the cluster class version that will be used, copy the relevant clusterclass section and append it to the end of the first yaml

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
    class: CLUSTER-CLASS                     # kubectl get clusterclass -n VSPHERE-NAMESPACE. See below for more info. 
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
#            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "3"
#            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "0"
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

`class: tanzukubernetescluster` or `class: builtin-generic-v3.1.0`

```yaml
      - name: clusterEncryptionConfigYaml
        value: |
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
      - name: controlPlaneCertificateRotation
        value:
          daysBefore: 90 
      - name: controlPlaneVolumes
        value:
        - capacity:
            storage: "15Gi"
          mountPath: "/var/lib/containerd"
          name: containerd
          storageClass: vsan-default-storage-policy
        - capacity:
            storage: "15Gi"
          mountPath: "/var/lib/kubelet"
          name: kubelet
          storageClass: vsan-default-storage-policy
      - name: defaultRegistrySecret
        value: 
      - name: defaultStorageClass
        value: vsan-default-storage-policy
      - name: defaultVolumeSnapshotClass
        value: vol-snapclass-foo
      - name: extensionCert
        value: 
      - name: kubeAPIServerFQDNs
        value: [demo.fqdn.com, test.fqdn.com]
      - name: nodePoolLabels
        value:
          - key: tenant 
            value: tenant-foo
          - key: organization
            value: engineering
          - key: managed
            value: ""
      - name: nodePoolTaints
        value: 
      - name: nodePoolVolumes
        value:
        - capacity:
            storage: "15Gi"
          mountPath: "/var/lib/containerd"
          name: containerd
          storageClass: vsan-default-storage-policy
        - capacity:
            storage: "15Gi"
          mountPath: "/var/lib/kubelet"
          name: kubelet
          storageClass: vsan-default-storage-policy
      - name: ntp
        value: ntp.vmware.com
      - name: podSecurityStandard
        value:
          audit: restricted
          auditVersion: latest
          enforce: privileged
          enforceVersion: latest
          warn: privileged
          warnVersion: latest 
      - name: proxy
        value:
          httpProxy: http://1.2.3.4:2139
          httpsProxy: http://1.2.3.4:2139
          noProxy: ["no.proxy.test1" , "no.proxy.test2"] 
      - name: storageClass
        value: vsan-default-storage-policy
      - name: storageClasses
        value: [vsan-default-storage-policy, vsan-default-storage-policy-1]
      - name: trust
        value:
          additionalTrustedCAs:
          - name: harbor-ca-1
      - name: user
        value: 
      - name: vmClass
        value: best-effort-small
      - name: volumeSnapshotClasses
        value: [vol-snapclass-foo, vol-snapclass-bar]
```

`builtin-generic-v3.2.0`

```yaml
      - name: kubernetes
        value:
          certificateRotation:
            enabled: true
            renewalDaysBeforeExpiry: 90
          endpointFQDNs: [demo.fqdn.com, test.fqdn.com]
          security: 
            podSecurityStandard:
              deactivated: false
              audit: privileged
              auditVersion: latest
              enforce: privileged
              enforceVersion: latest
              warn: privileged
              warnVersion: latest
      - name: node
        value:
          labels:
            tenant: tenant-foo
            organization: engineering
            managed: ""
          taints:
          - key: key1
            value: value1
            effect: NoSchedule
          - key: key2
            value: value2
            effect: NoExecute            
      - name: osConfiguration
        value:
          ntp:
            servers: [ntp.vmware.com]
          systemProxy:
            http: http://1.2.3.4:2139
            https: http://4.3.2.1:2139
            noProxy: ["no.proxy.test1" , "no.proxy.test2"] 
          trust:
            additionalTrustedCAs:
            - caCert:
                secretRef:
                  key: trust-ca-test1
                  name: "trust-ca-secret-1"
            - caCert:
                content: |-
                  -----BEGIN CERTIFICATE-----
                  MIIEczCCA1ugAwIBAgIBADANBgkqhkiG9w0BAQQFAD..AkGA1UEBhMCR0Ix
                  EzARBgNVBAgTClNvbWUtU3RhdGUxFDASBgNVBAoTC0..0EgTHRkMTcwNQYD
                  ...
                  -----END CERTIFICATE-----
          user:
            passwordSecret:
              key: user-secret-key-test
              name: user-secret-name-test
            sshAuthorizedKey: sshAuthorizedKeyTest...
            user: customuser 
      - name: resourceConfiguration
        value:
          systemReserved:
            cpu: 1
            memory: 4G
            automatic: false
      - name: storageClass
        value: vsan-default-storage-policy
      - name: vmClass
        value: best-effort-small
      - name: volumes
        value: 
        - name: containerd
          mountPath: /var/lib/containerd
          storageClass: vsan-default-storage-policy
          capacity: 15Gi
        - name: kubelet
          mountPath: /var/lib/kubelet
          storageClass: vsan-default-storage-policy
          capacity: 15Gi
      - name: vsphereOptions
        value:
          persistentVolumes:
            availableStorageClasses: [vsan-default-storage-policy, vsan-default-storage-policy-1 ]
            availableVolumeSnapshotClasses: [vol-snapclass-foo, vol-snapclass-bar]
            defaultStorageClass: vsan-default-storage-policy
            defaultVolumeSnapshotClass: vol-snapclass-bar
```

`builtin-generic-v3.3.0`

```yaml
to add
```
