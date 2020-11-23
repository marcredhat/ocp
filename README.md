# Automated deployment of OpenShift (tested with 4.5 and 4.6) and OpenShift Container Storage  on Bare Metal

## Table of Contents

- [OpenShift and OpenShift Container Storage  on Bare Metal](#baremetal)
  - [Table of Contents](#table-of-contents)
  - [Authors](#authors)
  - [Description](#description)
  - [Prerequisites](#prerequisites)
  - [Install OpenShift](#Install-OpenShift)
  - [Install OpenShift Container Storage](#Install-OpenShift-Container-Storage)
  - [OpenShift troubleshooting](#OpenShift-troubleshooting)
  - [Other automation opportunities](#Other-automation-opportunities)
  - [Automated OpenShift Container Storage deployments using Ansible](#Automated-OpenShift-Container-Storage-deployments-using-Ansible)
  - [CDP Private Cloud Base in a container](#CDP-Private-Cloud-Base-in-a-container)
 

## Authors

- Marc Chisinevski ([@marcredhat](https://github.com/marcredhat))

## Description

This guide describes the steps to install and configure OpenShift and OpenShift Container
Storage using the bare metal UPI installation method.

Follow the OCS/OpenShift Container Platform interoperability matrix at https://access.redhat.com/articles/4731161.


## Prerequisites

One baremetal server with min 128GB RAM and 32 CPUs


```bash
ssh -i ./.ssh/id_rsa <user>@<baremetal server>
```

```bash
cat /etc/centos-release
```

```text
CentOS Linux release 7.9.2009 (Core)
```

```bash
sudo yum install util-linux
```

```bash
lsmem | grep "Total online"
```

```text
Total online memory:    127.8G
```

```bash
lscpu
```

```text
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                40
On-line CPU(s) list:   0-39
Thread(s) per core:    2
Core(s) per socket:    10
Socket(s):             2
```

## Install OpenShift


```
sudo su
```

Append below lines in /etc/sysctl.conf:

```text
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
vm.swappiness=0
```

```bash
sysctl -p
```

Add the AddressFamily line to sshd_config :

```bash
# vi /etc/ssh/sshd_config

```

```text
....
AddressFamily inet
....
```

Restart sshd for changes to get get effect :

```bash
# systemctl restart sshd
```

copy the pull secret from https://cloud.redhat.com/openshift/install/pull-secret to /root/pull-secret


```bash

yum -y update
yum  -y remove  libvirt*
yum -y install git zip podman buildah skopeo libvirt*  bind-utils wget tar gcc python3-devel python3  xauth virt-install virt-viewer virt-manager libguestfs-tools-c libguestfs-tools tmux httpd-tools git x3270-x11 nc net-tools libvirt
systemctl start libvirtd.service
systemctl enable libvirtd
systemctl status libvirtd
```

Clean up any previous dynamic leases:
```bash
rm -rf /var/lib/libvirt/dnsmasq/virbr0.*
```

As root, setup a separate dnsmasq server on the host:

```bash
fuser -k 53/tcp
#systemctl disable NetworkManager
#systemctl stop NetworkManager
iptables -A OUTPUT -p udp --sport 1024:65535 --dport 67 -j ACCEPT
iptables -A OUTPUT -p udp --sport 68 --dport 67 -j ACCEPT
iptables -A INPUT -p udp --sport 67 --dport 68 -j ACCEPT
iptables -A INPUT -p udp --sport 1024:65535 --dport 68 -j ACCEPT
yum  -y install dnsmasq
#for x in $(virsh net-list --name); do virsh net-info $x | awk '/Bridge:/{print "except-interface="$2}'; done > #/etc/dnsmasq.d/except-interfaces.conf
>/etc/dnsmasq.d/except-interfaces.conf
sed -i '/^nameserver/i nameserver 127.0.0.1' /etc/resolv.conf
#chattr +i /etc/resolv.conf
```

As we are using dnsmasq for the local system only, lock it down by adding the line
listen-address=127.0.0.1
to the dnsmasq configuration file (/etc/dnsmasq.conf).

```bash
systemctl restart dnsmasq
systemctl enable dnsmasq
systemctl status dnsmasq
```

Reboot the server

We'll be using Khizer Naeem's automation (see also https://kxr.me/2019/08/17/openshift-4-upi-install-libvirt-kvm/)

```bash
git clone  https://github.com/kxr/ocp4_setup_upi_kvm
cd ocp4_setup_upi_kvm/
```


Let's have a look at the script that we'll be using

```bash
vi ./ocp4_setup_upi_kvm.sh
```

We are using a separate dnsmasq installed on the host.
So, under "# Default Values", we need to use:

```text
DNS_DIR=“/etc/dnsmasq.d”
```

```bash
echo 'export DNS_DIR="/etc/dnsmasq.d"' >> ~/.bashrc
source ~/.bashrc
```


(If you were using NetworkManager’s embedded dnsmasq, we'd keep the default value DNS_DIR="/etc/NetworkManager/dnsmasq.d")

Under "# Default Values", set the numbers of workers, masters, CPU and memory requirements etc
according to your requirements.

Edit disk, currently hardcoded at 50GB (search for "size=")
Example to change disk from 50GB to 80GB for all masters and workers:

```bash
sed 's/size=50/size=80/g' ./ocp4_setup_upi_kvm.sh > ./NEW_ocp4_setup_upi_kvm.sh
```

I'm using:
```bash
export CLUSTER_NAME=ocp4
export N_MAST=3
export N_WORK=3
export MAS_CPU=4
export MAS_MEM=16000
export WOR_CPU=6
export WOR_MEM=16000
export BTS_CPU=4
export BTS_MEM=16000
export LB_CPU=1
export LB_MEM=1024
```

Ensure  that dnsmasq is running  and that nothing else listen on port 53 (sudo ss -tulpn | grep 53)

```bash
systemctl status  dnsmasq
```

Ensure that libvirtd is running

```bash
systemctl status libvirtd.service
```

I had an issue where libvirtd was not starting ("cannot parse MAC address" error).

Solved by `virsh net-edit default` and removing the entire line with the MAC address.



Note: 
During the install, you may see multiple "unknown error has occurred: MultipleErr" messages. 
Please be patient and ignore those.


You can specify which version of OpenShift you want to deploy.

In the example below, we are deploying 4.5.stable.

```bash
chmod +x ./NEW_ocp4_setup_upi_kvm.sh
./NEW_ocp4_setup_upi_kvm.sh --ocp-version 4.5.stable
```

Press enter to continue.

This we'll deploy everything. Note that the DNS checks are automated.

Extract:
```text
====> Creating Loadbalancer VM: ok
====> Starting Loadbalancer VM ok
====> Waiting for Loadbalancer VM to obtain IP address: 192.168.122.206
====> Adding DHCP reservation for LB IP/MAC: ok
====> Adding /etc/hosts entry for LB IP: ok
====> Waiting for SSH access on LB VM: ok

##################
#### DNS CHECK ###
##################

====> Adding test records in /etc/hosts: ok
====> Testing DNS forward record from LB: ok
====> Testing DNS reverse record from LB: ok
====> Adding test SRV record in dnsmasq: ok
====> Testing SRV record from LB: ok
====> Cleaning up: ok

############################################
#### CREATE BOOTSTRAPING RHCOS/OCP NODES ###
############################################

====> Creating Boostrap VM: ok
====> Creating Master-1 VM: ok
====> Creating Master-2 VM: ok
====> Creating Master-3 VM: ok
====> Creating Worker-1 VM: ok
====> Creating Worker-2 VM: ok
```

**Note:** You should see VMs being created and getting IP addresses as follows:

```text
virsh list
 Id    Name                           State
----------------------------------------------------
 2     ocp4-lb                        running
 3     ocp4-bootstrap                 running
 7     ocp4-worker-1                  running
 8     ocp4-worker-2                  running
 9     ocp4-worker-3                  running

virsh domifaddr ocp4-worker-1
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet5      52:54:00:e9:84:05    ipv4         192.168.122.73/24
```

If the VMs do not get IP addresses from DHCP:

```bash
systemctl restart dnsmasq
systemctl restart libvirtd

```

## Clean up operators you do not need

Example:
```bash
oc scale --replicas=0 deployment --all -n openshift-monitoring
oc scale --replicas=0 deployment --all -n cluster-samples-operator
oc scale --replicas=0 deployment --all -n  cluster-autoscaler-operator
oc scale --replicas=0 deployment --all -n cluster-node-tuning-operator
oc scale --replicas=0 deployment --all -n cluster-samples-operator
```


## Adding worker nodes

```bash
cd /root/ocp4_setup_ocp4/
./add_node.sh --name worker-3
```

Approve Certificate Signing Requests
```bash
for i in `oc get csr --no-headers | grep -i pending |  awk '{ print $1 }'`; do oc adm certificate approve $i; done
```

## Adding disks to nodes

```bash
for i in 'c' 'd' 'e' 'f' 'g' 'h' 'i' 'j' 'k'
do
qemu-img create -f raw /opt/iso/disk${i}.img 2G
done


virsh attach-disk ocp4-worker-1 /opt/iso/diskc.img  sdc
virsh attach-disk ocp4-worker-1 /opt/iso/diskd.img  sdd
virsh attach-disk ocp4-worker-1 /opt/iso/diske.img  sde
virsh attach-disk ocp4-worker-2 /opt/iso/diskf.img  sdf
virsh attach-disk ocp4-worker-2 /opt/iso/diskg.img  sdg
virsh attach-disk ocp4-worker-2 /opt/iso/diskh.img  sdh
virsh attach-disk ocp4-worker-3 /opt/iso/diski.img  sdi
virsh attach-disk ocp4-worker-3 /opt/iso/diskj.img  sdj
virsh attach-disk ocp4-worker-3 /opt/iso/diskk.img  sdk
```

# Check that the disks were added to the worker nodes

```text
oc debug node/worker-1.ocp4.local
...
sh-4.4# chroot /host
sh-4.4# lsblk
NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                            8:0    0    2G  0 disk
sdb                            8:16   0    2G  0 disk
sdc                            8:32   0    2G  0 disk
sdd                            8:48   0    2G  0 disk
vda                          252:0    0   80G  0 disk
|-vda1                       252:1    0  384M  0 part /boot
|-vda2                       252:2    0  127M  0 part /boot/efi
|-vda3                       252:3    0    1M  0 part
`-vda4                       252:4    0 79.5G  0 part
  `-coreos-luks-root-nocrypt 253:0    0 79.5G  0 dm   /sysroot
```

## Connect to OpenShift 4 console

```bash
cat /root/ocp4_setup_ocp4/install_dir/auth/kubeadmin-password
```

On your laptop, change /etc/hosts so that
console-openshift-console.apps.ocp4.local and
oauth-openshift.apps.ocp4.local
point to 127.0.0.1

```bash
sudo ssh -i ./.ssh/id_rsa mchisinevski@vd1319.halxg.cloudera.com -L  443:console-openshift-console.apps.ocp4.local:443
```

Browse to https://console-openshift-console.apps.ocp4.local/

Connect as kubeadmin/<password from /root/ocp4_setup_ocp4/install_dir/auth/kubeadmin-password>

## ssh to the OpenShift 4 nodes

Example: 

```bash
cp /root/ocp4_setup_ocp4/oc /usr/bin
cp /root/ocp4_setup_ocp4/install_dir/auth/kubeconfig ~/.kube/config
```

```bash
oc get nodes
ssh -i /root/ocp4_setup_ocp4/sshkey core@master-1.ocp4.local
```

## Create user / authentication using htpasswd
Follow the steps at
https://github.com/marcredhat/workshop/blob/master/userauth_htpasswd.adoc


## Configure AlertManager
Follow the steps at
https://blog.openshift.com/openshift-4-3-alertmanager-configuration/


## Clean up OpenShift 4 install

```bash
./NEW_ocp4_setup_upi_kvm.sh --ocp-version 4.5.stable --destroy -y
```
