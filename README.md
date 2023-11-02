# THIS IS COMPLETELY UNSUPPORTED. DON'T EVEN THINK OF RED HAT SUPPORT WHEN YOU FOLLOW THESE INSTRUCTIONS.


# Prerequisites

You have a Raspberry Pi 4 and an SDCard with 32 GiB for the root partition and another 64 GiB as storage for PVs.
You have an access.redhat.com account, an OpenShift or Red Hat Device Edge Subscription and can access your pull secret.

# Provision Fedora IOT

Write the Fedora IOT image to the SD Card and configure your SSH key on first boot using the zezere service:

https://docs.fedoraproject.org/en-US/iot/ignition/


# Prepare MicroShift
## Grow root partition

```console
# parted /dev/mmcblk0 resizepart 3 32GiB
Warning: Partition /dev/mmcblk0p3 is being used. Are you sure you want to continue?
Yes/No? yes
Information: You may need to update /etc/fstab.

# mount /sysroot -o remount,rw
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.

# resize2fs /dev/mmcblk0p3
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mmcblk0p3 is mounted on /sysroot; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 4
The filesystem on /dev/mmcblk0p3 is now 7997952 (4k) blocks long.

``` 

## Prepare RHEL VG

```
# parted /dev/mmcblk0 mkpart extended 32GiB 128GB
Information: You may need to update /etc/fstab.

# parted /dev/mmcblk0 mkpart logical 33GiB 97GiB
Information: You may need to update /etc/fstab.

# parted /dev/mmcblk0 set 5 lvm on
Information: You may need to update /etc/fstab.

# pvcreate /dev/mmcblk0p5
  Physical volume "/dev/mmcblk0p5" successfully created.
  Creating devices file /etc/lvm/devices/system.devices

# vgcreate rhel /dev/mmcblk0p5
  Volume group "rhel" successfully created
```

## Update and Configure System & install mDNS
```console
# rpm-ostree update
⠈ Receiving objects; 99% (11112/11125) 5.3 MB/s 465.1 MB                                                                           1925 metadata, 9252 content objects fetched; 457581 KiB transferred in 93 seconds; 1.2 GB content written
Receiving objects; 99% (11112/11125) 5.3 MB/s 465.1 MB... done
Staging deployment... done
Freed: 205.8 MB (pkgcache branches: 1)
Upgraded:
[...]

# rpm-ostree install nss-mdns avahi
[...]

# timedatectl set-timezone Europe/Berlin

# hostnamectl set-hostname microshift-new.local

# systemctl reboot
``` 

## Enable mDNS 

```console
# systemctl enable --now avahi-daemon.service
```

## Download Microshift packages
Download the following RPMs from https://access.redhat.com/downloads/content/package-browser
* cri-o
* cri-tools
* crun
* microshift
* microshift-greenboot
* microshift-networking
* microshift-selinux
* openshift-clients
* openvswitch-selinux-extra-policy
* openvswitch3.1
* NetworkManager
* NetworkManager-libnm
* NetworkManager-ovs
* NetworkManager-wwan
* NetworkManager-wifi

## install MicroShift packages
``` console
# ls
NetworkManager-1.44.0-3.el9.aarch64.rpm
NetworkManager-libnm-1.44.0-3.el9.aarch64.rpm
NetworkManager-ovs-1.44.0-3.el9.aarch64.rpm
NetworkManager-wifi-1.44.0-3.el9.aarch64.rpm
NetworkManager-wwan-1.44.0-3.el9.aarch64.rpm
cri-o-1.27.1-11.1.rhaos4.14.git9b9c375.el9.aarch64.rpm
cri-tools-1.27.0-2.1.el9.aarch64.rpm
crun-1.11-1.rhaos4.14.el9.aarch64.rpm
jansson-2.14-1.el9.aarch64.rpm
microshift-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.aarch64.rpm
microshift-greenboot-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.noarch.rpm
microshift-networking-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.aarch64.rpm
microshift-selinux-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.noarch.rpm
openshift-clients-4.14.0-202310191146.p0.g0c63f9d.assembly.stream.el9.aarch64.rpm
openvswitch-selinux-extra-policy-1.0-34.el9fdp.noarch.rpm
openvswitch3.1-3.1.0-40.el9fdp.aarch64.rpm

# rpm-ostree override replace NetworkManager-{,libnm-,wwan-,wifi-}1* crun* jansson*
[...]

# rpm-ostree install ./cri-* ./openvswitch* ./microshift* ./openshift-clients* ./NetworkManager-ovs*
[...]

# systemctl reboot 
```

Once the system comes back up, verify the ostree:
```console
# rpm-ostree status
State: idle
Deployments:
● fedora-iot:fedora/stable/aarch64/iot
                  Version: 38.20231101.0 (2023-11-01T14:02:01Z)
               BaseCommit: e1b38ff646cab74c4d58d535d4ba70736e52ae5cfc95b88374d9cca93c4cc3c7
             GPGSignature: Valid signature by 6A51BBABBA3D5467B6171221809A8D7CEB10B464
           LocalOverrides: crun 1.11-1.fc38 -> 1.11-1.rhaos4.14.el9 jansson 2.13.1-6.fc38 -> 2.14-1.el9 NetworkManager-wwan NetworkManager-libnm NetworkManager NetworkManager-wifi 1:1.42.8-1.fc38 -> 1:1.44.0-3.el9
          LayeredPackages: avahi nss-mdns
            LocalPackages: cri-o-1.27.1-11.1.rhaos4.14.git9b9c375.el9.aarch64 cri-tools-1.27.0-2.1.el9.aarch64 microshift-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.aarch64
                           microshift-greenboot-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.noarch microshift-networking-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.aarch64
                           microshift-selinux-4.14.1-202310271350.p0.g1586504.assembly.4.14.1.el9.noarch NetworkManager-ovs-1:1.44.0-3.el9.aarch64 openshift-clients-4.14.0-202310191146.p0.g0c63f9d.assembly.stream.el9.aarch64
                           openvswitch-selinux-extra-policy-1.0-34.el9fdp.noarch openvswitch3.1-3.1.0-40.el9fdp.aarch64
[...]
```

## Download OpenShift Pull Secret & configure cri-o

Download your pull secret from https://console.redhat.com/openshift/install/pull-secret and store it as `~/openshift-pull-secret`

```console
# ls ~/openshift-pull-secret
/root/openshift-pull-secret

# cp $HOME/openshift-pull-secret /etc/crio/openshift-pull-secret

# chown root:root /etc/crio/openshift-pull-secret

# chmod 600 /etc/crio/openshift-pull-secret
```

## configure firewall

```console
# firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
success

# firewall-cmd --permanent --zone=trusted --add-source=169.254.169.1
success

# firewall-cmd --reload
success
```

# Use MicroShift

## enable & start microshift
```console
# systemctl enable --now microshift
```

## copy access credentials
```console
# mkdir -p ~/.kube/
# cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
# chmod go-r ~/.kube/config
```

## test client access

`oc status` and `oc project` will throw an error, since MicroShift does not know the project API objects. This is expected. You can test the CLI access as follows:

```console
# oc get all -A
[...]

# kubectl get all -A
[...]
```

# Test Application

We will use Node Red to test the MicroShift platform

## deploy sample application (Node-Red)

This sample application deploys Node-Red and creates a route that is advertised via mDNS. 

```console
# kubectl create namespace node-red
namespace/node-red created

# kubectl -n node-red apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: node-red-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  name: node-red
spec: 
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: node-red
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: node-red
    spec:
      containers:
      - image: nodered/node-red
        imagePullPolicy: Always
        name: node-red
        ports:
        - containerPort: 1880
          name: ui
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - name: data
          mountPath: /data
        livenessProbe:
          httpGet:
            path: /
            port: ui
        readinessProbe:
          httpGet:
            path: /
            port: ui
        startupProbe:
          httpGet:
            path: /
            port: ui
          failureThreshold: 30
          periodSeconds: 10
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: node-red-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: node-red
spec:
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 1880
    protocol: TCP
    targetPort: ui
    name: ui
  selector:
    app: node-red
  sessionAffinity: None
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: node-red
spec:
  host: nodered.local
  port:
    targetPort: ui
  to:
    kind: Service
    name: node-red
    weight: 100
  wildcardPolicy: None
EOF
persistentvolumeclaim/node-red-pv-claim created
deployment.apps/node-red created
service/node-red created
route.route.openshift.io/node-red created

# curl http://nodered.local/
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
[...]
```

## inspect sample application artifacts
```console
# kubectl describe pvc node-red-pv-claim
Name:          node-red-pv-claim
Namespace:     node-red
StorageClass:  topolvm-provisioner
Status:        Bound
Volume:        pvc-745b7227-76b0-4018-8474-f77238609f41
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: topolvm.io
               volume.kubernetes.io/selected-node: microshift-new.local
               volume.kubernetes.io/storage-provisioner: topolvm.io
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      3Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       node-red-6b87898896-hv5mv
Events:
  Type    Reason                 Age   From                                                                                Message
  ----    ------                 ----  ----                                                                                -------
  Normal  WaitForFirstConsumer   10m   persistentvolume-controller                                                         waiting for first consumer to be created before binding
  Normal  Provisioning           10m   topolvm.io_topolvm-controller-9b4ff4fb6-98jbf_6c22d071-31a6-47a4-9423-f0fbe9f177d2  External provisioner is provisioning volume for claim "node-red/node-red-pv-claim"
  Normal  ExternalProvisioning   10m   persistentvolume-controller                                                         waiting for a volume to be created, either by external provisioner "topolvm.io" or manually created by system administrator
  Normal  ProvisioningSucceeded  10m   topolvm.io_topolvm-controller-9b4ff4fb6-98jbf_6c22d071-31a6-47a4-9423-f0fbe9f177d2  Successfully provisioned volume pvc-745b7227-76b0-4018-8474-f77238609f41

# kubectl get pv -o jsonpath='{.items[*].metadata.name} {.items[*].spec.csi.volumeHandle}{"\n"}'
pvc-745b7227-76b0-4018-8474-f77238609f41 d2268877-8d1d-4fb0-b08d-bef055496bc5

# lvs
  LV                                   VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  d2268877-8d1d-4fb0-b08d-bef055496bc5 rhel -wi-ao---- 3.00g
#
```

## remove sample application
```console
# kubectl delete namespace node-red
```
