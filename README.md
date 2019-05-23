# Kubernetes Windows dynamic iSCSI Proof of Concept

This is a proof of concept for a Kubernetes external ISCSI storage provisioner. This allows you to use dynamic iSCSI Persistent Volumes for your k8s workloads.

It uses a [slightly modified version](https://github.com/kubernetes-incubator/external-storage/compare/master...wk8:master) of the [targetd external provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/iscsi/targetd) together with [Microsoft's FlexVolume plugins](https://github.com/microsoft/K8s-Storage-Plugins).

## One-time cluster setup

The steps below outline the one-time process to configure your k8s cluster.

### 1. Setup a targetd iSCSI server

Please follow the [Configure Storage](https://github.com/kubernetes-incubator/external-storage/tree/master/iscsi/targetd#configure-storage) and [Configure the iSCSI server](https://github.com/kubernetes-incubator/external-storage/tree/master/iscsi/targetd#configure-the-iscsi-server) sections of the [kubernetes-incubator/external-storage](https://github.com/kubernetes-incubator/external-storage/tree/master/iscsi/targetd) repo's README.

Save your iSCSI server's IP for subsequent steps, as well as your targetd credentials.

Of course, you can skip this step if you already have an iSCSI/targetd server running.

### 2. Install the FlexVolume plugin binaries on your Windows workers

Download the latest `flexvolume-windows.zip` from the [https://github.com/microsoft/K8s-Storage-Plugins/releases](https://github.com/microsoft/K8s-Storage-Plugins/releases), and unzip it to `C:\usr\libexec\kubernetes\kubelet-plugins\volume\exec\` on all your k8s Windows workers.

### 3. Deploy the external provisioner

#### 3.1. Compile your own version of the [targetd external provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/iscsi/targetd) - optional!

This step is optional; you can just use my image `wk88/win-iscsi-controller:latest` from the Docker Hub.

If you want though, clone [https://github.com/kubernetes-incubator/external-storage](https://github.com/kubernetes-incubator/external-storage), and apply [our patch](https://github.com/wk8/k8s-win-iscsi-demo/blob/master/iscsi-exeternal-provisioner.patch) - generated starting from [commit 88aaa1c2ed58bcfef55572bbe937f2813d6d4d1d](https://github.com/kubernetes-incubator/external-storage/commit/88aaa1c2ed58bcfef55572bbe937f2813d6d4d1d); this patch simply replaces the PV source to be a FlexVolume; then compile it, build & push a Docker image with your changes.

#### 3.2. k8s Deployment

Create a k8s secret with your targetd credentials from step 1:
```bash
# substitute your own credentials
kubectl create secret generic targetd-account --from-literal=username=admin --from-literal=password=ciao
```
and then apply the following k8s manifest, taking care to replace the `TARGETD_ADDRESS` - and if you want the image name to your own from step 3.1:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: iscsi-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "create", "update", "patch"]

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-iscsi-provisioner
subjects:
  - kind: ServiceAccount
    name: iscsi-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: iscsi-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: iscsi-provisioner

---

kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: iscsi-provisioner
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: iscsi-provisioner
    spec:
      containers:
        - name: iscsi-provisioner
          imagePullPolicy: Always
          image: wk88/win-iscsi-controller:latest
          args:
            - start
          env:
            - name: PROVISIONER_NAME
              value: iscsi-targetd
            - name: LOG_LEVEL
              value: debug
            - name: TARGETD_USERNAME
              valueFrom:
                secretKeyRef:
                  name: targetd-account
                  key: username
            - name: TARGETD_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: targetd-account
                  key: password
            - name: TARGETD_ADDRESS
              value: 10.88.93.135
              
      serviceAccount: iscsi-provisioner
      nodeSelector:
        beta.kubernetes.io/os: linux
```
This creates a k8s Deployment to provision iSCSI volumes, and gives it the right permissions.

#### 3.3. Create a k8s storage class

Apply the following k8s manifest, after replacing the relevant values:
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: iscsi-targetd-vg-targetd
provisioner: iscsi-targetd
parameters:
# this IP where the iscsi server is running
  targetPortal: 10.88.93.135:3260

# file system type for Windows
  fsType: NTFS
# this is the iscsi server iqn  
  iqn: iqn.2019-04.com.wk-castle:wk-targetd
  
# this is the iscsi interface to be used, the default is default
# iscsiInterface: default

# this must be on eof the volume groups condifgured in targed.yaml, the default is vg-targetd
# volumeGroup: vg-targetd

# this is a comma separated list of initiators that will be give access to the created volumes, they must correspond to what you have configured in your nodes.
  initiators: iqn.2019-04.com.wk-castle:wk-k8s-win1,iqn.2019-04.com.wk-castle:wk-k8s-win2
  
# whether or not to use chap authentication for discovery operations  
  chapAuthDiscovery: "false"
 
# whether or not to use chap authentication for session operations  
  chapAuthSession: "false"

```

At this point, you're good to go!

## Deploy pods using iSCSI volumes

You can now deploy pods (or deployments) using dynamic iSCSI volumes! For example, one could deploy the following manifest, that deploys a single pod that does nothing, but has a 100MB iSCSI disk dynamically provisioned & mounted at `C:/my_data`:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: win-iscsi-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "iscsi-targetd-vg-targetd"
spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi

---
  
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: win-iscsi-test
  name: win-iscsi-test
spec:
  containers:
  - name: win-iscsi-test
    image: mcr.microsoft.com/windows/servercore:ltsc2019
    command: ["powershell", "'while($true) { Start-Sleep -Seconds 1 }'"]
    volumeMounts:
    - name: iscsi-vol1
      mountPath: C:/my_data
      readOnly: false
  volumes:
  - name: iscsi-vol1
    persistentVolumeClaim:
      claimName: win-iscsi-claim
  nodeSelector:
    beta.kubernetes.io/os: windows
```
And you can do the same with `Deployment`s, etc...

## Troubleshooting

### MountVolume.SetUp failed for volume "XXX" : invalid character 'X' looking for beginning of value

If your pod gets stuck in `ContainerCreating` state, and you get the above error message when you `describe` it, it means that the FlexVolume plugin PowerShell scripts errored out (and didn't spit out a nice JSON for the kubelet's consumption). You can get the plugin log's by running
```PowerShell
Get-EventLog -LogName Application -Source Kube* -Newest 50
```
on the worker node the pod got scheduled to, and/or add debugging information and debug the PowerShell scripts from `C:\usr\libexec\kubernetes\kubelet-plugins\volume\exec\microsoft.com~iscsi.cmd`

## Special thanks

To [Deep Debroy](https://github.com/ddebroy)!
