# MicroShift on Fedora IoT 38

This guide will show you how to run the newly GA'ed bits of MicroShift on a Raspberry Pi 4 using 
Fedora IoT 38 or 39. It can announce routes via mDNS so hosting applications in an mDNS aware LAN is a breeze.

**THIS IS COMPLETELY UNSUPPORTED. DON'T EVEN THINK OF RED HAT SUPPORT WHEN YOU FOLLOW THESE INSTRUCTIONS.**

## Motivation

My [home automation setup](https://github.com/wrichter/homeautomation) is based on an [older install of MicroShift on a Raspberry Pi 4](https://www.opensourcerers.org/2022/01/17/openshift-on-raspberry-pi-4/). This hardware is close to ideal because it combines a small form factor which makes it fit my breaker box with GPIO ports that allow me to integrate additional sensors.

Recently I managed to fry the SDCard through repeated short circuits (yes, I was testing something). This is where the idea of a declarative configuration shone: after setting up MicroShift again based on the old instructions, I was able to deploy all applications & their configurations from my repository. I was also able to mount the old SDCard filesystem in a read-only fashion and copy all data over, effectively re-creating the complete state of my setup.

At the same time it felt silly to continue to run old versions when a GA'ed version of MicroShift is now available. This guide shows how to set up this more modern version on Fedora IoT 38. The same approach also works on Fedora IoT 39.

## Prerequisites

You have a Raspberry Pi 4 with [>2GB of RAM and an SDCard with >10 GiB for the root partition](https://access.redhat.com/documentation/en-us/red_hat_build_of_microshift/4.14/html/installing/microshift-install-rpm#microshift-install-system-requirements_microshift-install-rpm) and additional space as storage for PVs (this guide uses a 128GB SDCard). 

Note that the SDCard Interface on a Raspberry Pi is comparably slow, if you are interested in faster IO speeds, use a storage device attached via USB or similar. For this use case, I valued simplicity over performance and have everything running from the SDCard slot.

You have an access.redhat.com account, an OpenShift or Red Hat Device Edge Subscription and can access your [OpenShift pull secret](https://console.redhat.com/openshift/install/pull-secret).

### Provision Fedora IOT 38

[Create an SDCard with a bootable Fedora IOT 38 image](https://docs.fedoraproject.org/en-US/iot/physical-device-setup/#_create_a_bootable_sd_card). There are multiple ways to do this - on MacOS, I used
the Raspberry Pi Imager application to write the image to the SDCard, and configured the SSH key(s) on first boot using the [zezere service](https://docs.fedoraproject.org/en-US/iot/ignition/) provided by the Fedora project.

Once the device is fully booted, you should be able to log in as root via ssh.

### Download Microshift packages
Download the following RPMs from https://access.redhat.com/downloads/content/package-browser . You can do this
on the Raspberry Pi by copying the download links and using `curl -O`.
* cri-o
* cri-tools
* crun
* jansson
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

The `microshift-*`, `cri*`, `crun`, `openvswitch*` packages make up MicroShift. They pull in `NetworkManager*` as dependency, which in turn requires the `jansson` library. `openshift-clients` provides the CLI tools to interface with MicroShift. Make sure you download the RHEL9 version of all packages (with *el9* in the name) - some are available with a higher version number for RHEL8 which then fail to install.


## Prepare Fedora IoT and install MicroShift rpms

We will need to make some adjustments to be able to install MicroShift, such as preparing the filesystem layout, configuring hostname and time zone and updating the system.

### Grow the root partition

In order to be able to host the additional rpms and have some room to breathe, 
we'll extend the root partition to 32 GiB.

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

### Prepare RHEL volume group

MicroShift assumes an LVM volume group named "rhel" to host persistent volumes using the topolvm storage provider.
This volume group will be hosted on an extended partition.

```console
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

### Update and Configure System & install mDNS

rpm-ostree is the image-based deployment mechanism used by Fedora IoT. It requires you to reboot the system to make package installations visible. Two ostrees are maintained by default at the same time. This allows you to roll back your device to a previously known good configuration in case a new setup is broken.

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

### Enable mDNS 

After reboot, log back in and activate the mMDNS service.

```console
# systemctl enable --now avahi-daemon.service
```


### Install MicroShift packages

We will install the MicroShift packages in two steps - first we overlay existing packages from the Fedora IoT ostree with the ones downloaded earlier. Then we will install the remaining packages.

**At this point it is a good idea to mention again that the resulting setup will be completely unsupported.**

```console
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

Once the system comes back up, log in again and verify the ostree:

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

## Configure and start MicroShift

At this point, we pretty much just have to follow the [MicroShift installation instructions](https://access.redhat.com/documentation/en-us/red_hat_build_of_microshift/4.14/html/installing/microshift-install-rpm#installing-microshift-from-rpm-package_microshift-install-rpm) from step 3 onwards. There's a small caveat as we have to add a compatibility fix for Fedora's systemd-resolved.

### Download OpenShift Pull Secret & configure cri-o

Download your pull secret from [https://console.redhat.com/openshift/install/pull-secret] and store it as `~/openshift-pull-secret`

```console
# ls ~/openshift-pull-secret
/root/openshift-pull-secret

# cp $HOME/openshift-pull-secret /etc/crio/openshift-pull-secret

# chown root:root /etc/crio/openshift-pull-secret

# chmod 600 /etc/crio/openshift-pull-secret
```

### Configure firewall

Open the mandatory firewall ports.

```console
# firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
success

# firewall-cmd --permanent --zone=trusted --add-source=169.254.169.1
success

# firewall-cmd --reload
success
```

### Enable MicroShift
Enable (but not yet start) MicroShift. This will also cause the greenboot health checks to run at system startup. In case the microshift service fails, disable MicroShift before rebooting - otherwise the greenboot health checks will identify a problem on the next reboot, and will roll back to the previously known good configuration.

```console
# systemctl enable microshift.service
``` 

### Add and enable systemd-resolved-fix

On start, MicroShift copies the resolv.conf DNS configuration file location into the CoreDNS configmap. 
On a system like Fedora IoT 38 this causes the wrong resolv.conf location to be present 
in the CoreDNS config map, causing the pod to crashloop. To work around this, we will create 
systemd unit that patches the configmap after microshift has started.

```console
# cat >/etc/systemd/system/microshift-on-fedora.service <<EOF
[Unit]
Description=MicroShift 4.14 with systemd-resolved-fix
After=microshift.service
BindsTo=microshift.service

[Service]
ExecStart=/bin/bash -c "kubectl --kubeconfig /var/lib/microshift/resources/kubeadmin/kubeconfig get cm dns-default -n openshift-dns -o yaml | sed 's_/run/systemd/resolve/resolv.conf_/etc/resolv.conf_' | kubectl --kubeconfig /var/lib/microshift/resources/kubeadmin/kubeconfig apply -f -"
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# systemctl enable microshift-on-fedora.service
```

### Start MicroShift

Now you can start MicroShift (with the fix). Please use the microshift-on-fedora unit to start and stop microshift, it will in turn start and stop the underlying microshift service. You can  query the status of microshift using the regular microshift.service unit.
Give it some time to pull all required pods & start up properly.

```console
# systemctl start microshift-on-fedora

# systemctl status microshift
● microshift.service - MicroShift
     Loaded: loaded (/usr/lib/systemd/system/microshift.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Thu 2023-11-02 22:22:40 CET; 10h ago
   Main PID: 1525 (microshift)
      Tasks: 16 (limit: 4160)
     Memory: 580.1M
        CPU: 1h 54min 41.851s
     CGroup: /system.slice/microshift.service
             └─1525 microshift run

Nov 03 09:12:28 microshift-new.local microshift[1525]: kubelet I1103 09:12:28.694006    1525 kubelet.go:2457] "SyncLoop (PLEG): event for pod" pod="openshift-dns/dns-default-qcvpv" event={"ID":"e86bfc5a-7fb9-425e-ab9b-1ecef66f6996","Type":"ContainerDied","Data":"6514c6574172b44b036011f8a97a37d932066def9d98075a1b30d6b8c8e7977b"}
Nov 03 09:12:28 microshift-new.local microshift[1525]: kubelet I1103 09:12:28.694223    1525 scope.go:115] "RemoveContainer" containerID="080aa0f9ff8c69165d7d464abb13afd7a6236bd52aedaf3eef61cc5e3b6ef158"
[...]
```

### Copy access credentials

Upon start, microshift creates a kubeconfig configuration file that allows administrative access. Copy it into your home directory to use via the CLI.

```console
# mkdir -p ~/.kube/
# cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
# chmod go-r ~/.kube/config
```

### Test client access

`oc status` and `oc project` will throw an error, since MicroShift does not know the project API objects. This is expected. You can test the CLI access as follows:

```console
# oc get all -A
[...]

# kubectl get all -A
[...]
```

## Test the Setup using a sample Application

We will use [Node Red](https://nodered.org/) as a sample application to test the MicroShift platform. It starts on an unprivileged port (doesn't have to run as root) and can use a persisten volume to store its flows.

### Deploy sample application

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
```

Once the application is deployed, you should be able to access it from the CLI or your mDNS aware browser.

```console
# curl http://nodered.local/
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
[...]
```

### Review storage artifacts

The topolvm storage provider creates LVM logical volumes for each Kubernetes PV that is created.

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

### Remove sample application

To remove the sample application, simply delete the namespace. This will also clean up the LVM logical volumes.

```console
# kubectl delete namespace node-red
namespace "node-red" deleted

# lvs
#
```

## Summary

This guide showed how to set up MicroShift 4.14 (and beyond?) on a Raspberry Pi 4 using Fedora IoT 38 – a totally unsupported setup. This is mostly achieved by replacing some Fedora packages with those coming from Red Hat Enterprise Linux and adding the remaining packages to the image-based Fedora deployment and following the MicroShift installation instructions.


Fedora 38 uses systemd-resolved while Red Hat Enterprise Linux doesn’t – this triggers a code path in MicroShift which causes a wrong configuration to be stored for the CoreDNS pod. This is why we need a workaround systemd unit that reverses the configuration after MicroShift startup.


## Fix after 1 year - the certificate expires

systemctl stop microshift

cd /var/lib/containers/storage/volumes/microshift-data/_data 

mv certs certs.old
mv resources resources.old

systemctl start microshift

(certs + resources werden wieder angelegt)

mv ~/.kube/config ~/.kube/config.old

cp ./resources/kubeadmin/kubeconfig ~/.kube/config

export NEWCA=$(base64 -w 0 certs/ca-bundle/ca-bundle.crt)
for API in `oc get apiservices | grep default/openshift | cut -f1 -d" "`; do oc patch apiservices $API -p "{\"spec\":{\"caBundle\":\"${NEWCA}\"}}"; done