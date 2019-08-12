# Setup KVM on a physical server

As part of my learning, I will try to explain to you how to install KVM on physical computer and server. That is the first stop on the road to install the OpenStack.

### Assumptions:

  - KVM cluster
  - All VM stored on /mapper/red01(Raid5 matrix)
  - Two networks: NAT and Bridge to physical interface
  - As much automation as possible
  
### Triggered topics:

  - Ansible
  - Linux networking
  - libvirt
  
 ### Machine I run cluster on:
 
  - CPU: 2x Intel(R) Xeon(R) CPU E5645 @ 2.40GHz
  - RAM: 128 GB
  - HDD:
      - SSD 128GB
      - HDD RAID1 1TB
      - HDD RAID5 8TB
      - HDD RAID5 8TB
      
      
# KVM instalation
  ... TODO ...
  
  
# KVM configuration

### Infrastructure, you need to modify.

```
    /etc/libvirt/
    └── qemu.conf

    /mapper/red01/kvm
    ├── disks/
    └── images/

    /
    ├── kvm-disks -> /mapper/red01/kvm/disks/
    └── kvm-images -> /mapper/red01/kvm/images/

    /var/lib/libvirt/
    └── networks
        └── host-bridge.xml
```

<br>

## Qemu config
Open the file: /etc/libvirt/qemu.conf and change the following values:

```
user = "root"
group = "root"

security_driver = "none"
```

First two lines set the qemu user and group to root. The last line disables all security for our cluster. I plan to change it, but for learning purposes, I will leave it as it is now.

Then restart the `libvirtd` service

```
# service libvirt restart
```

<br>

## Host bridge network
I use `Ubuntu Server 18.04.3 LTS`, and I am going to prepare creating bridge network for this system.

#### Enable forwarding. 
Open the /etc/sysctl.conf file and uncomment/add following line:

```
net.ipv4.ip_forward=1
```

Then apply changes with the `sudo sysctl -p /etc/sysctl.conf` command
Then reload procps with the `sudo /etc/init.d/procps restart`

#### Install missing software
Type following command: `sudo apt-get install bridge-utils`

#### Create bridge interface
My physical output interface is `eno2`. I am going to create a bridge for this interface. Make sure if your name is correct. If you do not have any backup connect and you make a mistake, you can lose the connection to your server.

*Notes:*
 - Network: 192.168.0.0/24
 - Net mask: 255.255.255.0
 - Gateway: 192.168.0.1
 - DHCP on the router: enabled

1. Open the /etc/netplan/50-cloud-init.yaml and update your physical interface config to similar:

```
eno2:
    addresses: []
    dhcp4: false

```

2. Add bridge configuration:
```
    bridges:
        br0:
            interfaces: [eno2]
            dhcp4: false
            addresses: [192.168.0.100/24]
            gateway4: 192.168.0.1
            nameservers:
                addresses:
                  - 192.168.0.1
                  - 8.8.8.8
```

What is this configuration mean?

- `dhcp4: false` - With this directive, We disable DHCP for our interfaces. We don't want any IP for the physical interface. We can disable DHCP also for br0 because We assigned the static IP(192.168.0.100).
- `interfaces: [eno2]` - Determine which interface do you want to bridge.
- Rest of the fields are straightforward

*Note:* Remember to replace config with your parameters.

Finally my file looks like below:

```
# This file is generated from information provided by
# the datasource.  Changes to it will not persist across an instance.
# To disable cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
        eno1:
            addresses: []
            dhcp4: true
        eno2:
            addresses: []
            dhcp4: false
        enp4s0f0:
            addresses: []
            dhcp4: true
            optional: true
        enp4s0f1:
            addresses: []
            dhcp4: true
            optional: true
        enp5s0f0:
            addresses: []
            dhcp4: true
            optional: true
        enp5s0f1:
            addresses: []
            dhcp4: true
            optional: true

    bridges:
        br0:
            interfaces: [eno2]
            dhcp4: false
            addresses: [192.168.0.100/24]
            gateway4: 192.168.0.1
            nameservers:
                addresses:
                  - 192.168.0.1
                  - 8.8.8.8
```

3. Now test and apply your configuration

```
netplan try
netplan apply
```

4. Verify if all works good. You should see 3 interfaces:
Physical interface, should be up without any IP

```
3: eno2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether 78:2b:cb:33:29:3d brd ff:ff:ff:ff:ff:ff
```

Bridge interface shoudl be up with static IP: 192.168.0.100:

```
94: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 82:43:74:4e:8d:c4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.100/24 brd 192.168.0.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::8043:74ff:fe4e:8dc4/64 scope link
       valid_lft forever preferred_lft forever
```

And you should be able to see a KVM NAT interface, with some DHCP IP:

```
111: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:43:8d:69 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
```


<br>

## Create storage

#### What are KVM pools? 

Today, everybody uses different disks, settings, partitions, file systems, etc.

All storages have pros and cons. Ones provide better control others are faster.  For example, an application which is doing some computations and save data on HDD requires a faster drive than DB, which keeps user DB.

KVM pools come against those problems. KVM pool is a mechanism, which helps us manage our drives. For example, We can create one for SSD drives and the second one for HDD drives. Users/administrators that use our infrastructure does not need to know what is under the mask. 

We have a few types of pool types:

- dir - uses direcectory as a storage for volumes.
- disk - uses physical hard disks as a storage for volumes.
- fs - uses preformated partition to store volumes.
- lvm - uses LVM volume as a storage driver.
- netfs - uses Use network file system(like NFS) to store volumes.


#### Let's start...

1. Create pool on /mapper/red01/kvm/disks . Let's call it 

```
virsh pool-define-as   \
  --name kvm-disks     \
  --type dir          \
  --target /mapper/red01/kvm/disks

virsh pool-build kvm-pool
virsh pool-start kvm-pool
virsh pool-autostart kvm-pool
```

Above commands define pool with the type of dir, called kvm-disks, which points to the /mapper/red01/kvm/disks directory. The next CLI command builds the pool. For the dir type, it is not necessary, but for another(like a disk), it is required to prepare filesystem. Then we start the pool and enable autostart.


2. Verify, your configuration

Run the `virsh pool-list` command:

```
 Name                 State      Autostart
-------------------------------------------
 default              active     yes
 kvm-disks            active     yes
```

Run the `virsh pool-dumpxml command: 

```
<pool type='dir'>
  <name>kvm-disks</name>
  <uuid>4ab0ab6e-5006-4dcf-9fdf-d769946783b5</uuid>
  <capacity unit='bytes'>7936293720064</capacity>
  <allocation unit='bytes'>346643271680</allocation>
  <available unit='bytes'>7589650448384</available>
  <source>
  </source>
  <target>
    <path>/mapper/red01/kvm/disks</path>
    <permissions>
      <mode>0755</mode>
      <owner>0</owner>
      <group>0</group>
    </permissions>
  </target>
</pool>
```

Now we can verify if everything is correct. If not, you can run the `virsh pool-edit kvm-disks` command.
