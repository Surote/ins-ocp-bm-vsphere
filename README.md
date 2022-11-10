# OCP-Baremetal-Install-IPI-sim on Vsphere

Created: November 6, 2022 8:15 PM
Last Edited Time: November 10, 2022 11:12 AM
Type: OCP
tag: baremetal, install

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled.png)

## Pre-requitsite / Environment

- OCP installer 4.10
- dhcp/dns server
- baremetal network with Internet access
- Installer/Helper node - Red Hat Enterprise Linux release 8.5 (Ootpa)
- Mainly follow this document below

[https://docs.openshift.com/container-platform/4.10/installing/installing_bare_metal_ipi/ipi-install-installation-workflow.html](https://docs.openshift.com/container-platform/4.10/installing/installing_bare_metal_ipi/ipi-install-installation-workflow.html)

- needed DNS record + PTR

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%201.png)

- reserve IP with DHCP and MAC address of baremetal network nic

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%202.png)

## Prepare empty VMs for boot over PXE (master/worker)

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%203.png)

- add Configuration Parameters on every VM `disk.EnableUUID: TRUE` if not set ironic-python-agent on each node will failed  symptom is  PXE boot loop.
- either BIOS or EFI are fine to use.
- Disabled `secure boot` - PXE will not works if enable.
- (optional) make sure using Thin provisioning for optimize disk space usage.

## Prepare VM for Provisioning

### Enable H/W virtualization on Provisioner node

```bash
ERROR Error: Error defining libvirt domain: virError(Code=8, Domain=10, Message='invalid argument: could not get preferred machine for /usr/libexec/qemu-kvm type=kvm')
```

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%204.png)

- install rhel 8.5
- create user kni
- install/download needed tools virt etc.
- subscription-register
- bridge networks for bm, provisioning networks
- pull-secret for download ocp software

## Prepare VM for IPMI-GW

### Install vsbmc to be an IPMI proxy

follow the link below

[vbmc4vsphere](https://pypi.org/project/vbmc4vsphere/)

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%205.png)

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%206.png)

- master x 3
NIC: baremetal, provisioning
- worker x 2
NIC: baremetal, provisioning
- IPMI-GW x 1
NIC: baremetal
- helper-node/provisioner-node x 1
NIC: baremetal, provisioning

## Install-config.yaml

```bash

apiVersion: v1
baseDomain: surote.local
metadata:
  name: bm
networking:
  machineNetwork:
  - cidr: 192.168.0.0/22
  networkType: OVNKubernetes
compute:
- name: worker
  replicas: 2
controlPlane:
  name: master
  replicas: 3
  platform:
    baremetal: {}
platform:
  baremetal:
    apiVIP: 192.168.3.130
    ingressVIP: 192.168.3.140
    provisioningNetworkCIDR: 172.22.0.0/24
    hosts:
      - name: openshift-master-0
        role: master
        bmc:
          address: ipmi://192.168.3.102:6231
          username: admin
          password: password
        bootMACAddress: 00:50:56:84:d4:5e
      - name: openshift-master-1
        role: master
        bmc:
          address: ipmi://192.168.3.102:6232
          username: admin
          password: password
        bootMACAddress: 00:50:56:84:5c:1d
      - name: openshift-master-2
        role: master
        bmc:
          address: ipmi://192.168.3.102:6233
          username: admin
          password: password
        bootMACAddress: 00:50:56:84:2b:e2
      - name: openshift-worker-0
        role: worker
        bmc:
          address: ipmi://192.168.3.102:6234
          username: admin
          password: password
        bootMACAddress: 00:50:56:84:0c:8d
      - name: openshift-worker-1
        role: worker
        bmc:
          address: ipmi://192.168.3.102:6235
          username: admin
          password: password
        bootMACAddress: 00:50:56:84:01:0a
pullSecret: |
  <REDACTED>
sshKey: |
  ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILToaXlZLwENv3o2HAm7Rb38JDNRbOADfs1amvrE5pFZ kni@localhost.localdomain
```

## Install and pray

```bash
# create cluster command
./openshift-baremetal-install --dir ~/clusterconfigs --log-level debug create cluster

# clear previous virsh bootstrap
for i in $(sudo virsh list | tail -n +3 | grep bootstrap | awk {'print $2'}); do   sudo virsh destroy $i;   sudo virsh undefine $i;   sudo virsh vol-delete $i --pool $i;   sudo virsh vol-delete $i.ign --pool $i;   sudo virsh pool-destroy $i;   sudo virsh pool-undefine $i; done
```

## Appendix/ Remark

```bash
#LAB
--- baremetal NIC ----
00:50:56:84:f0:0f master 0
00:50:56:84:27:cc master 1
00:50:56:84:2e:fd master 2

00:50:56:84:bd:6e worker 0
00:50:56:84:ca:c2 worker 1

--- provisioning NIC ---
00:50:56:84:d4:5e  master 0
00:50:56:84:5c:1d  master 1
00:50:56:84:2b:e2  master 2

00:50:56:84:0c:8d  worker 0
00:50:56:84:01:0a  worker 1
```

- helper node / installer not works on rhel 9 due to some error about virsh

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%207.png)

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%208.png)

## Result

[success-log-ipi-baremetal-sim-vsphere.log](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/success-log-ipi-baremetal-sim-vsphere.log)

[result-cluster.log](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/result-cluster.log)

![Untitled](OCP-Baremetal-Install-IPI-sim%20on%20Vsphere%2001be04674e4e41da8624d8e0066d954a/Untitled%209.png)