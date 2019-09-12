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

### Scenario 1: Expand the attached volume

### Scenario 2: Attach new disk and map to specific location (/var/lib/mysql)

### Scenario 3: Attach new disk and expand root (/)


... TODO ...
