# THE CODE AND PROCEDURES IN THIS REPOSITORY ALLOWS NVIDIA-DRIVER CONTAINER TO RUN ON OKD 4.9

You can build the image this way (althought those argument values are already set in the Dockerfile and nvidia-driver script)
```
docker build --build-arg FEDORA_VERSION=34 --build-arg DRIVER_VERSION=550.54.15 -t 550.54.15-fedora .
```
In order to run this image on OKD 4.9 you will need to workaround a bug in SELinux (container-selinux package). To achieve this, we need to create a new MachineConfig to change SELinux's module.

I won't cover all the steps needed to install GPU-OPERATOR on OKD 4.9, but, in summary that is what you will need:

1) Install NFD Operator (NODE FEATURE DISCOVERY)
- It is straight-forward, just install the operator and create a new instance (dont need to change anything in the template);
- Create the COOKIE SECRET;

```
oc -n openshift-nfd create secret generic nfd-operator-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
``` 
- Once created, you will need to fix the OAUTH settings (in nfd-controller-manager DEPLOYMENT), like this (spec>template>spec>containers):

```
              - args:
                - --provider=openshift
                - --https-address=:8443
                - --http-address=
                - --email-domain=*
                - --upstream=http://localhost:8080
                - --tls-cert=/etc/secrets/tls.crt
                - --tls-key=/etc/secrets/tls.key
                - --cookie-secret-file=/etc/proxy/secrets/session_secret
                - --openshift-service-account=nfd-operator
                - --openshift-ca=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                - --skip-auth-regex=^/metrics
                image: quay.io/openshift/origin-oauth-proxy:4.9
```

- Also, you will need to edit the volumes (spec>template>spec>containers) and volumeMounts (spec>template>spec>containers>resources):

```
             volumes:
              - name: node-feature-discovery-operator-tls
                secret:
                  secretName: node-feature-discovery-operator-tls
              - name: nfd-operator-proxy
                secret:
                  secretName: nfd-operator-proxy
```

```
               volumeMounts:
                - mountPath: /etc/secrets
                  name: node-feature-discovery-operator-tls
                - mountPath: /etc/proxy/secrets
                  name: nfd-operator-proxy

```

- (OPTIONAL) Change the node-selector of the openshift-nfd NAMESPACE:

```
annotations:
    openshift.io/node-selector: ‘ ‘
```
2) Install the GPU-OPERATOR HELMCHARTREPOSITORY;

```
apiVersion: helm.openshift.io/v1beta1
kind: HelmChartRepository
metadata:
  name: nvidia-repo
spec:
  connectionConfig:
    url: https://helm.ngc.nvidia.com/nvidia
```
3) Create the GPU-OPERATOR NAMESPACE:

```
oc create ns nvidia-gpu-operator
```
4) Install the HELM CHART, changing these values:

```
plataform:
  openshift: true
…
nfd:
  enabled: false
…
operator:
  defaultRuntime: crio
…
driver:
  repository: docker.io/gvoliveira
  image: fedora-coreos-34-nvidia-driver
  version: 550.54.15
…
  manager:
     - name: ENABLE_GPU_POD_EVICTION
        value: "true"
      - name: DRAIN_DELETE_EMPTYDIR_DATA
        value: "false"
…
 upgradePolicy:
    autoUpgrade: false
```
Obs.: Had to disable ```autoUpgrade``` cause it was avoiding GPU's pods deletion which causes the nvidia-driver pod to get stuck (CrashLoopBackOff) on restarts because it won't be able to uninstall/unload the driver. Due to a known bug, the driver reinstall is triggered at every pod restart.

5) Create the MACHINEPOOLCONFIG (which we will use to apply the SELINUX fix):

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: worker-with-nvidia-gpu
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,gpu]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/gpu: ""
```
6) Create the MACHINECONFIG (which will enable IOMMU as NIVIDA needs it):

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: gpu
  name: 100-worker-iommu 
spec:
  config:
    ignition:
      version: 3.2.0
  kernelArguments:
      - intel_iommu=on
```
7) Create the MACHINESET to deploy the GPU NODES:

```
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  annotations:
    machine.openshift.io/memoryMb: "16384"
    machine.openshift.io/vCPU: "8"
  name: okd4-13v1c-worker-stage-gpu
  namespace: openshift-machine-api
spec:
  deletePolicy: Newest
  replicas: 1
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: okd4-13v1c
      machine.openshift.io/cluster-api-machineset: okd4-13v1c-worker-stage-gpu
  template:
    metadata:
      labels:
        nvidia.com/device-plugin.config: Tesla-T4
        machine.openshift.io/cluster-api-cluster: okd4-13v1c
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: okd4-13v1c-worker-stage-gpu
        node-role.kubernetes.io/stage: "true" 
        node-role.kubernetes.io/gpu: ""
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      metadata:
        labels:
          node-role.kubernetes.io/gpu: ""
          node-role.kubernetes.io/stage: "true"
      providerSpec:
        value:
          apiVersion: vsphereprovider.openshift.io/v1beta1
          credentialsSecret:
            name: vsphere-cloud-credentials
          diskGiB: 120
          kind: VSphereMachineProviderSpec
          memoryMiB: 16384
          network:
            devices:
            - networkName: NETWORK-OKD-VLAN13
          numCPUs: 8
          numCoresPerSocket: 1
          template: okd4-13v1c-rhcos
          userDataSecret:
            name: worker-user-data
          workspace:
            datacenter: DATACENTER
            datastore: OKD-DATASTORE
            folder: /DATACENTER/vm/DATACENTER/OKD4
            resourcePool: /DATACENTER/host/CLUSTER/Resources/OKD
            server: vcenter-okd.example.com
```

Obs.: ResourcePool definition must be removed in most implementations

8) Once the new node is setup, turn the VIRTUALMACHINE off and edit its settings (the settings will vary according to the GPU - you will need to check it):

```
VM Settings > VM options > Advanced > Configuration Parameters > Edit Configuration
pciPassthru.use64bitMMIO TRUE
pciPassthru.64bitMMIOSizeGB 512
```

9) Attach the GPU/PCI DEVICE (in our case: DYNAMIC DIRECTPATH I/O) to the VIRTUALMACHINE;
* Obs.: You may need to update the HARDWARE VERSION of your VIRTUALMACHINE.

10) Build the NVIDIA DRIVER IMAGE for OKD 4.9:

- git clone this repo;
- docker build --build-arg FEDORA_VERSION=34 --build-arg DRIVER_VERSION=550.54.15 -t 550.54.15-fedora .
- docker tag 550.54.15-fedora your_docker_repo/fedora-coreos-34-nvidia-driver:550.54.15-fedora34
- docker push your_docker_repo/fedora-coreos-34-nvidia-driver:550.54.15-fedora34

11) Create a butane config to fix the SELinux bug:

- Download the butane binary:

```
curl https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane --output butane
chmod +x butane
mv butane /usr/local/bin/
```

- Create the butane config file (selinux_fix_nvidia.bu) that will generate a MACHINECONFIG (MC):

```
variant: openshift
version: 4.9.0
metadata:
  name: 81-nvidia-driver-fix-modprobe-spc
  labels:
    machineconfiguration.openshift.io/role: gpu
storage:
  files:
  - path: /etc/kvc/spc_lockdown_integrity.cil
    contents:
      inline: |
        (allow spc_t spc_t (lockdown (integrity)))
systemd:
  units:
  - name: fix-modprobe-spc.service
    enabled: true
    contents: |
      [Unit]
      Description=Load spc_lockdown_integrity SELinux policy module
      Before=kmods-via-containers@.service
      ConditionPathIsDirectory=!/etc/selinux/targeted/active/modules/400/spc_lockdown_integrity

      [Service]
      Type=oneshot
      ExecStart=/usr/sbin/semodule -i /etc/kvc/spc_lockdown_integrity.cil

      [Install]
      WantedBy=default.target
```

- Generate the MC YAML:

```
butane selinux_fix_nvidia.bu --files-dir . -o 81-selinux-fix-gpu.yaml
```

- Apply the MC:

```
oc create -f 81-selinux-fix-gpu.yaml
```

12) Configure the GPU Time-Slicing to share a GPU resource between multiple pods

- Update the "gpu-operator" helm and change the "devicePlugin" section to this:

```
devicePlugin:
  args: []
  config:
    create: true
    data:
      Tesla-T4:
      version: v1
      sharing:
        timeSlicing:
          failRequestsGreaterThanOne: true
          renameByDefault: false
          resources:
            - name: nvidia.com/gpu
              replicas: 8
    default: "Tesla-T4"
    name: device-plugin-config
  enabled: true
```
Obs.: In our case we have Tesla T4 GPU, you will have to change it according to your GPU. Also, it doesn't support MIG (Multi-instance GPU), if your GPU supports it you rather use it instead of "time-slicing" method.

INFO: The argument "failRequestsGreaterThanOne=true" won't allow one pod to request more than 1 GPU.

13) Enable NVIDIA Dashboard

- Install the HELM CHART on OKD:

```
apiVersion: helm.openshift.io/v1beta1
kind: HelmChartRepository
metadata:
  name: console-plugin-nvidia-gpu
spec:
  connectionConfig:
    url: https://rh-ecosystem-edge.github.io/console-plugin-nvidia-gpu
```

- Include "tag" and "version" to the HELM values and install it:

```
  version: 0.0.1
```
- Enable the plugin by running the following command:

```oc patch clusterpolicies.nvidia.com cluster-policy --patch '{ "spec": { "dcgmExporter": { "config": { "name": "console-plugin-nvidia-gpu" } } } }' --type=merge```

- Verify the plugins field is specified:

```oc get consoles.operator.openshift.io cluster --output=jsonpath="{.spec.plugins}"``` 

If it is not specified, then run the following to enable the plugin:

```oc patch consoles.operator.openshift.io cluster --patch '{ "spec": { "plugins": ["console-plugin-nvidia-gpu"] } }' --type=merge```

If it is specified, then run the following to enable the plugin:

```oc patch consoles.operator.openshift.io cluster --patch '[{"op": "add", "path": "/spec/plugins/-", "value": "console-plugin-nvidia-gpu" }]' --type=json```

- Download the latest NVIDIA DCGM Exporter Dashboard from the DCGM Exporter repository on GitHub:

```curl -LfO https://github.com/NVIDIA/dcgm-exporter/raw/main/grafana/dcgm-exporter-dashboard.json``` 

- Create a config map from the downloaded file in the openshift-config-managed namespace:

```oc create configmap nvidia-dcgm-exporter-dashboard -n openshift-config-managed --from-file=dcgm-exporter-dashboard.json```

- Label the config map to expose the dashboard in the Administrator perspective of the web console:

```oc label configmap nvidia-dcgm-exporter-dashboard -n openshift-config-managed "console.openshift.io/dashboard=true"```

- Optional: Label the config map to expose the dashboard in the Developer perspecitive of the web console:

```oc label configmap nvidia-dcgm-exporter-dashboard -n openshift-config-managed "console.openshift.io/odc-dashboard=true"```

- View the created resource and verify the labels:

```oc -n openshift-config-managed get cm nvidia-dcgm-exporter-dashboard --show-labels```

# THINGS THAT STILL NEED TO BE REVIEWED
1) Create a new template for the GPU NODES so we don't need to manually change the VMs after it being created;
2) The template must have UPDATED HW VERSION;
3) We didn't change the BOOT OPTIONS to EFI as suggested in NVIDIA Docs;
4) NVIDIA DASHBOARD doesn't work properly on 4.9;
5) GPU's NODES autoscaling.

