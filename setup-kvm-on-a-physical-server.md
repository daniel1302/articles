# Setup KVM on a physical server

As part of my learning, I will try to explain to you how to install KVM on physical computer and server. That is the first stop on the road to install the OpenStack.

### Assumptions:

  - Do not use GUI*
  - KVM cluster
  - All VM stored on /mapper/red01(Raid5 matrix)
  - Two networks: NAT and Bridge to physical interface
  - As much automation as possible

**Note:**

  \* Usually, I work with servers I have no real access. Also, my connection to the internet is slow, so it is not possible to transfer the graphic interface.

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

### Naming conventions

I follow my convention of naming. I have a few servers called by dragon names like Farnir, Smaug, etc. That name becomes the environment for all running machines and programs. You should find your standards and follow them.
Host name contains following parts:

    - Environment name
    - Destiny / Use case
    - System version (optional)

An exception is the base image, which is base for all next virtual machines.

Examples:
- Domain: `kvm-smaug`, `kvm-farnir`
- Host names: `smaug-mysql`,


# KVM instalation
  
  1. Before you start installing the KVM soft, check if your processor has support for virtualization. To check your virtualization capabilities, you need to run the following command: 
  
    ```
    # egrep -c '(svm|vmx)' /proc/cpuinfo
    24
    ```

2. If you can see 0, that means, your computer does not support virtualization. If you can see any number larger than 0, it's valid. If your computer is not supporting virtualization, try to enable it in BIOS settings. To do it restart your computer, find the "Virtualization (VT-x/AMD-V)" option and turn it on.

3. Now you can verify if your system is correctly set up. To do it run:  the `kvm-ok` command. Correct output is:

    ```
    # kvm-ok
    INFO: /dev/kvm exists
    KVM acceleration can be used
    ```

4. Now You can install quemu and all required software. To do it run: `apt-get install qemu-kvm libvirt-bin bridge-utils virt-manager`

5. The KVM uses virtualization extension like Intel VT for Intel processors or AMD-V for AMD processors. Let's see what modules are enabled for you:

    ```
    # lsmod | grep kvm
    kvm_intel             212992  34
    kvm                   598016  1 kvm_intel
    irqbypass              16384  20 kvm
    ```

6. If your module are disabled, please run `modprobe --all kvm`

7. Verify installation:

    ```
    # virsh list --all

    Id Name                 State
    ----------------------------------
    ```

8. It's done!


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

KVM pools come against those problems. KVM pool is a mechanism, which helps us manage our drives. For example, We can create one for SSD drives and the second one for HDD drives. Users/administrators that use our infrastructure does not need to know what is under the mask. We can think about that like about a container or like about abstraction layer that isolates physical devices between software.

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

Run the `virsh pool-dumpxml` command:

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

<br>

## Deploy first VM

On the beginning, I will provide the bash command and explain a bit what it does. To create a new VM, I am going to use the virt-install bash command, which comes from virt tools.

```
virt-install \
  --virt-type=kvm \
  --name debian10-base \
  --ram 8000 \
  --disk pool=kvm-disks,sparse=true,size=20 \
  --vcpus 4 \
  --os-type linux \
  --os-variant debian9 \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial \
  --location 'http://ftp.nl.debian.org/debian/dists/buster/main/installer-amd64/' \
  --extra-args 'console=ttyS0,115200n8 serial'
```

Command has a lot of parameters. You can read more here: https://linux.die.net/man/1/virt-install.

  - `virt-type` - The hypervisor to install on. We are using KVM so, choice is clear.
  - `name` - Name of our VM
  - `ram` - Ram allocation for our VM
  - `disk` - Configure disk.
  - `vcpus` - Number of virtual CPUs allocated for VM
  - `os-type` - Type of virtualized system
  - `os-varian` - Variant of selected OS, used for futher optimizazations. For now libvirt does not support Debian 10.
  - `network` - Network settings*
  - `graphics` - Settings of graphics interface. We do not need graphics support for now, so disable it
  - `console` - Connect VMs console to pty, with type serial. Serial is configured in `extra-args`
  - `location` - Location of installation disk. For more disks see below.

You can notice, I used only one network. Moreover, this is the default network. It's a good catch, but It is aware because I want to show you **How to attach network interface and set up it in the guest OS**.

Ok, let's type above command and install your OS. My installation parameters:

#### Install system

- Language: `English`
- Location: `Poland`
- Locales: `United States - en_US.UTF-8`
- Keyboard: `American English`
- Hostname: `debian10-base`
- Domain name: `kvm-smaug`
- Username for your account: `kvm`

Disk partitions:

- ext4, size: 10GB, mount point: /
- ext4, size: 10GB, mount point: /home

Software to install

- SSH Server
- standard system utilities

Install GRUB in /dev/vda.

#### Verify installation

1. Log in as root (or kvm).
2. Type `ip a` to see your IP. In my example, I can see my IP address is `192.168.122.100`. That IP belongs to default networks. Make a note this address.

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:f9:5f:ec brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.100/24 brd 192.168.122.255 scope global dynamic ens2
       valid_lft 3209sec preferred_lft 3209sec
    inet6 fe80::5054:ff:fef9:5fec/64 scope link
       valid_lft forever preferred_lft forever
```

3. Verify if your SSH server is running. Type `systemctl status sshd`

```
ssh.service - OpenBSD Secure Shell server
   Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enab
   Active: active (running) since Tue 2019-08-13 14:49:52 CEST; 8min ago
     Docs: man:sshd(8)
           man:sshd_config(5)
  Process: 387 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
 Main PID: 395 (sshd)
    Tasks: 1 (limit: 4915)
   Memory: 3.7M
           └─395 /usr/sbin/sshd -D

Aug 13 14:49:52 debian10-base systemd[1]: Starting OpenBSD Secure Shell server..
Aug 13 14:49:52 debian10-base sshd[395]: Server listening on 0.0.0.0 port 22.
Aug 13 14:49:52 debian10-base sshd[395]: Server listening on :: port 22.
Aug 13 14:49:52 debian10-base systemd[1]: Started OpenBSD Secure Shell server.
```

4. Reboot VM to see if everything is working as expected. To do that run `systemctl reboot`.

5. Detach console by typing: `CRTL + ^`. It did not work for me, so I closed the terminal.

6. Try to connect to your VM from the host computer.

```
ssh kvm@192.168.122.100
```
7. If everything is working, you did a great job!


#### Installation images:

| Distro name  | OS variant    | URL                                                                    |
| ------------ | ------------- | -----------------------------------------------------------------------|
| Debian 10    | debian9       | http://ftp.nl.debian.org/debian/dists/buster/main/installer-amd64/     |
| Debian 9     | debian9       | http://ftp.nl.debian.org/debian/dists/stretch/main/installer-amd64/    |
| Debian 8     | debian8       | http://ftp.nl.debian.org/debian/dists/jessie/main/installer-amd64/     |
| Debian 7     | debian7       | http://ftp.nl.debian.org/debian/dists/wheezy/main/installer-amd64/     |
| Debian 6     | debian6       | http://ftp.nl.debian.org/debian/dists/squeeze/main/installer-amd64/    |
| CentOS 7     | centos7       | http://mirror.i3d.net/pub/centos/7/os/x86_64/                          |
| CentOS 6     | centos6       | http://mirror.i3d.net/pub/centos/6/os/x86_64/                          |
| CentOS 5     | centos5       | http://mirror.i3d.net/pub/centos/5/os/x86_64/                          |
| Ubuntu 18.04 | linux         | http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/    |
| Ubuntu 16.04 | linux         | http://archive.ubuntu.com/ubuntu/dists/xenial/main/installer-amd64/    |
| Ubuntu 14.04 | linux         | http://archive.ubuntu.com/ubuntu/dists/trusty/main/installer-amd64/    |
| Ubuntu 12.04 | linux         | http://archive.ubuntu.com/ubuntu/dists/precise/main/installer-amd64/   |
| Ubuntu 10.04 | linux         | http://archive.ubuntu.com/ubuntu/dists/lucid/main/installer-amd64/     |
| OpenSUSE 13  | linux         | http://download.opensuse.org/distribution/13.2/repo/oss/               |
| OpenSUSE 12  | linux         | http://download.opensuse.org/distribution/12.3/repo/oss/               |
| OpenSUSE 11  | linux         | http://download.opensuse.org/distribution/11.4/repo/oss/               |

<br>

## Attach network

On the beginning, let's list all our networks on the KVM node. Execute the following command:  `virsh net-list`.

```
virsh net-list

 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
```

We can notice only the default network is available. Let's create a new one then.

cat /var/lib/libvirt/networks/host-bridge.xml

1. Create a new network file in the `/var/lib/libvirt/networks/host-bridge.xml`
File content: 

    ```
    <network>
    <name>host-bridge</name>
    <forward mode="bridge"/>
    <bridge name="br0"/>
    </network>
    ```

2. Create a network based on the XML file. Execute the following commands: 
    ```
    # virsh net-define /var/lib/libvirt/networks/host-bridge.xml
    # virsh net-start host-bridge
    # virsh net-autostart host-bridge
    ```

    Example: 

    ```
    # virsh net-define /var/lib/libvirt/networks/host-bridge.xml
    Network host-bridge-1 defined from /var/lib/libvirt/networks/host-bridge.xml

    # virsh net-list --all
    Name                 State      Autostart     Persistent
    ----------------------------------------------------------
    default              active     yes           yes
    host-bridge          inactive   no            yes

    # virsh net-start host-bridge
    Network host-bridge started

    # virsh net-list
    Name                 State      Autostart     Persistent
    ----------------------------------------------------------
    default              active     yes           yes
    host-bridge          active     no            yes

    # virsh net-autostart host-bridge
    Network host-bridge marked as autostarted

    # virsh net-list
    Name                 State      Autostart     Persistent
    ----------------------------------------------------------
    default              active     yes           yes
    host-bridge          active     yes           yes
    ```

3. I assume you have got VM called `debian10-base`. Let's find it.

    ```
    # virsh list
    Id    Name                           State
    ----------------------------------------------------
    41    debian10-base                  running
    ```

4. SSH into VM:

    ```
    virsh console debian10-base
    ```

5. See current network settings.
    ```
    # ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
    2: ens2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 52:54:00:f9:5f:ec brd ff:ff:ff:ff:ff:ff
        inet 192.168.122.100/24 brd 192.168.122.255 scope global dynamic ens2
        valid_lft 3583sec preferred_lft 3583sec
        inet6 fe80::5054:ff:fef9:5fec/64 scope link
        valid_lft forever preferred_lft forever
    ```

6. Type `CTRL + ]` to exit from the attached console.

7. Generate random MAC address.

    ```
    # openssl rand -hex 6 | sed 's/\(..\)/\1:/g; s/:$//'
    52:54:00:4b:73:5f
    ```

8. Attach interface with the following command:

    ```
    # virsh attach-interface    \
        --domain debian10-base  \
        --type network          \
        --source host-bridge    \
        --model virtio          \
        --mac 52:54:00:4b:73:5f \
        --config                \
        --live
    ```

9. Disable netfilter on the bridge. This is required to disable filtering packages by our KVM host for bridged interfaces.

    ```
    # cat >> /etc/sysctl.conf <<EOF
    net.bridge.bridge-nf-call-ip6tables = 0
    net.bridge.bridge-nf-call-iptables = 0
    net.bridge.bridge-nf-call-arptables = 0
    EOF
    # sysctl -p /etc/sysctl.conf
    ```

10. It's good time to restart our VM

    ```
    # virsh reboot debian10-base
    Domain debian10-base is being rebooted
    ```

11. SSH into VM with `virsh console debian10-base`. Then see available networks:

    ```
    # ip a

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
    2: ens2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 52:54:00:f9:5f:ec brd ff:ff:ff:ff:ff:ff
        inet 192.168.122.100/24 brd 192.168.122.255 scope global dynamic ens2
        valid_lft 3583sec preferred_lft 3583sec
        inet6 fe80::5054:ff:fef9:5fec/64 scope link
        valid_lft forever preferred_lft forever
    3: ens9: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
        link/ether 52:54:00:4b:73:5f brd ff:ff:ff:ff:ff:ff
    ```

12. We can see our MAC addres: `52:54:00:4b:73:5f`. Now lets UP the interface

    ```
    # ip link set dev ens9 up
    ```

13. See if the interface is up

    ```
    # ip a

    3: ens9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4b:73:5f brd ff:ff:ff:ff:ff:ff
    inet6 fe80::5054:ff:fe4b:735f/64 scope link
       valid_lft forever preferred_lft forever

    ```

14. Obtain new IP from DHCP server with `/usr/sbin/dhclient <interface name>`

    ```
    # /usr/sbin/dhclient ens9
    ```

15. That's it! You should be able to see what is your IP:

    ```
    # ip a

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
    2: ens2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 52:54:00:f9:5f:ec brd ff:ff:ff:ff:ff:ff
        inet 192.168.122.100/24 brd 192.168.122.255 scope global dynamic ens2
        valid_lft 2480sec preferred_lft 2480sec
        inet6 fe80::5054:ff:fef9:5fec/64 scope link
        valid_lft forever preferred_lft forever
    3: ens9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 52:54:00:4b:73:5f brd ff:ff:ff:ff:ff:ff
        inet 192.168.0.71/24 brd 192.168.0.255 scope global dynamic ens9
        valid_lft 692448sec preferred_lft 692448sec
        inet6 fe80::5054:ff:fe4b:735f/64 scope link
        valid_lft forever preferred_lft forever

    ```

<br>

## Expand HDD

#### Quick about the volume format:
Different people prefer to use a different type of volumes.

*Raw* format is the most universal and portable. You can export raw disks into another Virtualization format. Another advantage of that format is high performance.

The *qcow* format provides several like features: snapshots, compressions, backups, etc.

At the moment, it's not important what kind of format you choose.

### Scenario 1: Attach new disk and map to specific location (/var/lib/mysql)

Like You remember We use the "kvm-pool" pool as a volumes storage.

1. Let's check our volumes pool.

    ```
    # virsh pool-dumpxml kvm-disks
    <pool type='dir'>
        <name>kvm-disks</name>
        <uuid>4ab0ab6e-5006-4dcf-9fdf-d769946783b5</uuid>
        <capacity unit='bytes'>7936293720064</capacity>
        <allocation unit='bytes'>348723073024</allocation>
        <available unit='bytes'>7587570647040</available>
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

2. Go to the folder where pool is defined: `/mapper/red01/kvm/disks`

3. Let's create new `10G` disk:

    ```
    # cd /mapper/red01/kvm/disks
    # qemu-img create -f raw test-disk-10G 10G
    Formatting 'test-disk-10G', fmt=raw size=10737418240
    ```

    You can also use dd command:

    ```
    # dd if=/dev/zero of=test-disk-2-10G bs=1M count=10240 status=progress
    10078912512 bytes (10 GB, 9.4 GiB) copied, 9 s, 1.1 GB/s
    10240+0 records in
    10240+0 records out
    10737418240 bytes (11 GB, 10 GiB) copied, 9.54978 s, 1.1 GB/s
    ```

    ```
    # sudo ls -lh
    total 15G
    -rw------- 1 root root  21G Sep 24 12:30 debian10-base.qcow2
    -rw------- 1 root root 8.1G Aug 13 09:55 debian9.qcow2
    -rw-r--r-- 1 root root  10G Sep 24 12:27 test-disk-10G
    -rw-r--r-- 1 root root  10G Sep 24 12:30 test-disk-2-10G
    ```

4. I am going to attach this image to the `debian10-base` VM. Let's log into that instance.

    ```
    # virsh console debian10-base
    Connected to domain debian10-base
    Escape character is ^]

    root@debian10-base:~#
    ```

5. Find currently attached disks

    ```
    root@debian10-base:~# ls -als /dev | grep -e vd.$
    0 brw-rw----  1 root disk    254,   0 Sep 10 12:39 vda
    ```
    Here You can see that currently attached is disk /dev/vda

6. We can attach our disk as vdb device. Let's do this! Log out from the guest VM, then attach created volume.

    ```
    # virsh attach-disk debian10-base \
        --source /mapper/red01/kvm/disks/test-disk-10G \
        --target vdb \
        --persistent

    Disk attached successfully
    ```

7. Login to the guest VM and verify.

    ```
    root@debian10-base:~# fdisk -l | grep '^Disk /dev/vd[a-z]'
    Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
    Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
    ```

8. Create a new partition

    ```
    root@debian10-base:~# fdisk /dev/vdb

    Welcome to fdisk (util-linux 2.33.1).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.

    Device does not contain a recognized partition table.
    Created a new DOS disklabel with disk identifier 0xa0b8bd13.

    Command (m for help): n
    Partition type
    p   primary (0 primary, 0 extended, 4 free)
    e   extended (container for logical partitions)
    Select (default p): p
    Partition number (1-4, default 1):
    First sector (2048-20971519, default 2048):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

    Created a new partition 1 of type 'Linux' and of size 10 GiB.

    Command (m for help): w
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```

9. Verify partitions

    ```
    root@debian10-base:~# lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    vda    254:0    0   20G  0 disk
    ├─vda1 254:1    0  9.3G  0 part /
    └─vda2 254:2    0 10.7G  0 part /home
    vdb    254:16   0   10G  0 disk
    └─vdb1 254:17   0   10G  0 part
    ```

10. Format created partition

    ```
    root@debian10-base:~# mkfs.ext4 /dev/vdb1
    mke2fs 1.44.5 (15-Dec-2018)
    Creating filesystem with 2621184 4k blocks and 655360 inodes
    Filesystem UUID: 659eed8b-c913-45ad-a2a6-79c499e338a2
    Superblock backups stored on blocks:
            32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

    Allocating group tables: done
    Writing inode tables: done
    Creating journal (16384 blocks): done
    Writing superblocks and filesystem accounting information: done

    ```

11. Mount partition to the `/var/lib/mysql`

    ```
    root@debian10-base:~#mount /dev/vdb1 /var/lib/mysql

    root@debian10-base:~# mount | grep vdb1
    /dev/vdb1 on /var/lib/mysql type ext4 (rw,relatime)
    ```

12. Add a fstab entry

    ```
    root@debian10-base:~# echo '/dev/vdb1    /var/lib/mysql    ext4    defaults    0  0' > /etc/fstab
    root@debian10-base:~# tail -n 1 /etc/fstab
    /dev/vdb1    /var/lib/mysql    ext4    defaults    0  0
    ```

### Scenario 2: Expand the attached volume

1. Before We start let's check current file system size for particular guest

    ```
    root@debian10-base:~# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    udev            3.8G     0  3.8G   0% /dev
    tmpfs           779M   77M  703M  10% /run
    /dev/vda1       9.2G  1.1G  7.7G  12% /
    tmpfs           3.9G     0  3.9G   0% /dev/shm
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/vda2        11G   41M  9.9G   1% /home
    tmpfs           779M     0  779M   0% /run/user/0
    /dev/vdb1       9.8G   37M  9.3G   1% /var/lib/mysql
    ```

2. Shutdown the guest VM

    ```
    root@debian10-base:~# shutdown -P now
    ```

3. On the host machine resize the disk file

    ```
    # sudo qemu-img resize -f raw ./test-disk-10G +25G
    Image resized.
    ```

4. Start guest VM

    ```
    # virsh start  debian10-base
    Domain debian10-base started
    ```

5. Login into guest VM, and verify filesystem

    ```
    # virsh console debian10-base

    root@debian10-base:~# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    udev            3.8G     0  3.8G   0% /dev
    tmpfs           779M  8.4M  771M   2% /run
    /dev/vda1       9.2G  1.1G  7.7G  12% /
    tmpfs           3.9G     0  3.9G   0% /dev/shm
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/vdb1       9.8G   37M  9.3G   1% /var/lib/mysql
    tmpfs           779M     0  779M   0% /run/user/0

    root@debian10-base:~# lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    vda    254:0    0   20G  0 disk
    ├─vda1 254:1    0  9.3G  0 part /
    └─vda2 254:2    0 10.7G  0 part
    vdb    254:16   0   35G  0 disk
    └─vdb1 254:17   0   10G  0 part /var/lib/mysql
    ```

    As You can see disk size is 35G, but used is only 10. Let's change it.

6. Modify partition

    ```
    root@debian10-base:~# fdisk /dev/vdb1
    Command (m for help): p
    Disk /dev/vdb: 35 GiB, 37580963840 bytes, 73400320 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0xa0b8bd13

    Device     Boot Start      End  Sectors Size Id Type
    /dev/vdb1        2048 20971519 20969472  10G 83 Linux

    Command (m for help): d
    Selected partition 1
    Partition 1 has been deleted.

    Command (m for help): p
    Disk /dev/vdb: 35 GiB, 37580963840 bytes, 73400320 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0xa0b8bd13

    Command (m for help): n
    Partition type
    p   primary (0 primary, 0 extended, 4 free)
    e   extended (container for logical partitions)
    Select (default p): p
    Partition number (1-4, default 1):
    First sector (2048-73400319, default 2048):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-73400319, default 73400319):

    Created a new partition 1 of type 'Linux' and of size 35 GiB.
    Partition #1 contains a ext4 signature.

    Do you want to remove the signature? [Y]es/[N]o: N

    Command (m for help): w

    The partition table has been altered.
    Syncing disks.
    ```

    Commands:
    * p - Print partition table
    * d - Delete partition
    * n - Create new partition
    * w - Write changes

7. Resize current mount point

    ```
    root@debian10-base:~# resize2fs /dev/vdb1
    ```

8. Reboot your guest VM, then verify results

    ```
    root@debian10-base:~# reboot

    root@debian10-base:~# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    udev            3.8G     0  3.8G   0% /dev
    tmpfs           779M  8.4M  771M   2% /run
    /dev/vda1       9.2G  1.1G  7.7G  12% /
    tmpfs           3.9G     0  3.9G   0% /dev/shm
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/vdb1        35G   48M   33G   1% /var/lib/mysql
    tmpfs           779M     0  779M   0% /run/user/0
    ```
