# CrashPlan on FreeNAS 9.10+ using bhyve Virtualized CentOS Guest 

FreeNAS 9.10.1 includes [iohyve](https://github.com/pr1ntf/iohyve), a command line utility for creating, managing, and launching [bhyve](https://wiki.freebsd.org/bhyve) guests. This new virtualization option provides a simple way of hosting Crashplan on a supported operating system.

# Other Approaches

## CrashPlan Plugin
The original method for running CrashPlan was to leverage a [plugin](https://doc.freenas.org/9.10/freenas_plugins.html) which would install within a [jail](https://doc.freenas.org/9.10/freenas_jails.html). This option has several drawbacks. The CrashPlan application is running using [Linux Binary Compatibility](https://www.freebsd.org/doc/handbook/linuxemu.html). Modern versions of CrashPlan require newer versions of the Linux Kerenl than are supported by FreeBSD. For more details refer to this [CrashPlan Troubleshooting Article](https://support.code42.com/CrashPlan/4/Troubleshooting/Linux_CrashPlan_App_Version_4.5_Cannot_Back_Up).

    CrashPlan app version 4.5.x or later: Linux Kernel 2.6.27 or newer and glibc 2.9+
    CrashPlan app version 4.4.1 or earlier: Linux Kernel 2.6.13 or newer and glibc 2.4+

## VirtualBox
FreeNAS has long supported VirtrualBox, an alternate virtualzation option to bhyve. VirtualBox runs as a plugin within a jail and can host operating systems that are fully compatible with CrashPlan. 


# Hardware Requirements
[bhyve](https://wiki.freebsd.org/bhyve) is supported on most Intel and AMD processors that report the "POPCNT" (POPulation Count) processor feature in dmesg(8). iohyve will throw an error In order to test if your FreeNAS server is compatible run the following command:

```bash
grep POPCNT /var/run/dmesg.boot
```

If no results are found your machine likely does not support bhyve virtualization. Your machine should support [bhyve](https://wiki.freebsd.org/bhyve) if this command returns something like:
```bash
  Features2=0x29ee3ff<SSE3,PCLMULQDQ,DTES64,MON,DS_CPL,VMX,SMX,EST,TM2,SSSE3,CX16,xTPR,PDCM,PCID,DCA,SSE4.1,SSE4.2,POPCNT,AESNI
```

# bhyve and iohyve
bhyve is a FreeBSD virtualization technology. iohyve is a command line utility that will help us manage the bhyve CrashPlan virtual machine.

# Setting Up bhyve Virtualization Support
We will be using iohyve to setup bhyve virtualization support.

iohyve is included in FreeNAS 9.10.1+. 

## Identify Network Adapter to be Used
Your CrashPlan virtual machine will need to be connected to the internet in order to be useful. bhyve will need to know which adapter bhyve will use. In the examples below em0 is the name of the network adapter. This will be used when we setup iohyve. 

**Command Line**
```
[root@freenas ~]# ifconfig                                                      
em0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500       
        options=9b<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,VLAN_HWCSUM>           
        ether 00:0c:29:c4:17:99                                                 
        inet 192.168.1.2 netmask 0xffffff00 broadcast 192.168.1.255           
        nd6 options=9<PERFORMNUD,IFDISABLED>                                    
        media: Ethernet autoselect (1000baseT <full-duplex>)                    
        status: active                                                          
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384               
        options=600003<RXCSUM,TXCSUM,RXCSUM_IPV6,TXCSUM_IPV6>                   
        inet6 ::1 prefixlen 128                                                 
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2                              
        inet 127.0.0.1 netmask 0xff000000                                       
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL> 
```

**Web Interface**
![identifying-network-adapter](./images/identifying-network-adapter.png?raw=true)

## Identify Storage Volume Name
bhyve requires storage in order to save ISO images for installation, and for storing the actual Virtual Machines. You will need to choose a storage pool for this purpose.

**Command Line**
```
[root@freenas ~]# zpool list                                                    
NAME           SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROO
T                                                                               
freenas-boot  7.94G   624M  7.33G         -      -     7%  1.00x  ONLINE  -     
volume1       17.9G   660K  17.9G         -     0%     0%  1.00x  ONLINE  /mnt  
[root@freenas ~]#
````

**Web Interface**
![identfying-volume-name](./images/identifying-volume-name.png?raw=true)

## Configure Tunables
By defualt, the kernal modules requried for bhyve virtualization are not turned on. iohyve will enable the kernel modules for you with, but if you want virtualization support to survive a reboot you will need to configure some [tunables](https://doc.freenas.org/9.10/freenas_system.html#tunables) in order enable support automatically on reboot.

The two tunables we will add are (where em0 is replaced with your actual network adapter name):

* `iohyve_enable="YES"`
* `iohyve_flags="kmod=1 net=em0"`

The `iohyve_enable` tells the FreeNAS server to run `/usr/local/etc/rc.d/iohyve` on boot. The `iohyve_flags` tells iohyve to load the required Kernel Modules and also the adapter to use for virtual machine network bridging.

![adding-tunables](./images/adding-tunables.png?raw=true)


## Start iohyve
We can now use the iohyve setup command on the FreeNAS shell.
```
[root@freenas ~]# iohyve setup pool=volume1 kmod=1 net=em0                      
Setting up iohyve pool...                                                       
On FreeNAS installation.                                                        
Checking for symbolic link to /iohyve from /mnt/iohyve...                       
Symbolic link to /iohyve from /mnt/iohyve successfully created.                 
Loading kernel modules...                                                       
Seting up bridge0 on em0...                                                     
net.link.tap.up_on_open: 0 -> 1 
```

## Create a CentOS VM

Download a CentOS 7 IO image from your mirror of choice
```
iohyve fetchiso http://mirror.its.dal.ca/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-1511.iso
```

Create a virtual machine with a 8GB disk. 
```
iohyve create centosguest 8G
```

Set the boot loader as GRUB
```
iohyve set centosguest loader=grub-bhyve
```

Identify the Operating System of the VM
```
iohyve set centosguest os=centos7
```

Configure the VM with 8GB of Memory. How much memory you allocate will depend on your own requirements, [CrashPlan System Requirements](https://support.code42.com/CrashPlan/4/Getting_Started/Code42_CrashPlan_System_Requirements) are one 1G. The default for a iohyve created machines is 256M which won't get you far with CrashPlan.
```
iohyve set centosguest ram=2G
```

Set the VM to start automatically.
```
iohyve set centosguest boot=1
```

Start the VM to install CentOS
```
[root@freenas] ~# iohyve install centosguest CentOS-7-x86_64-Everything-1511.iso
Installing centosguest...
```

Launch a Console Session on the Virtual Machine so you can perform the install
```
iohyve console centosguest
```

## CentOS Installation
At this point you should be connected to the console of your new VM. The VM should boot and then start the CentOS text based install wizard.

Setting a static IP during the install will make it easier to connect to the VM in the future.

![text-mode-centos-install](./images/text-mode-centos-install.png?raw=true)

When the installation is complete, I needed to run the following commands on the FreeNAS machine to reboot the VM
```
iohyve destroy centosguest
iohyve start centosguest
iohyve console centosguest
```


## CrashPlan Installation
```
yum install wget
wget https://download.code42.com/installs/linux/install/CrashPlan/CrashPlan_4.7.0_Linux.tgz
tar xvfz CrashPlan_4.7.0_Linux.tgz
cd crashplan-install
./install.sh
```

Select yes for all the defaults in the CrashPlan installer.

## Allow Remote Connections to CrashPlan and Connect via GUI
Ensure the CrashPlan server procses is stopped on the CentOS VM.
```
/usr/local/crashplan/bin/CrashPlanEngine stop
```

Edit `/usr/local/crashplan/conf/my.service.xml` `serviceHost` to be `0.0.0.0` so that it will listen on all IPs and not just accept local connections.

Then copy contents of `/var/lib/crashplan/.ui_info` into the local GUI .ui_info file (for Windows this is often at `C:\ProgramData\CrashPlan\.ui_info`) with the last IP the IP address of the centosguest instead of 0.0.0.0.
```
[root@localhost ~]# cat /var/lib/crashplan/.ui_info
4243,a40fe588-2116-443f-b1ba-09cedeb9bc4e,0.0.0.0
```

Open up the CentOS Firewall to allow traffic on 4200 and 4243
```
firewall-cmd --zone=public --add-port=4200/tcp --permanent
firewall-cmd --zone=public --add-port=4243/tcp --permanent
firewall-cmd --zone=public --add-port=4200/udp --permanent
firewall-cmd --zone=public --add-port=4243/udp --permanent
firewall-cmd --reload
```

Start the CrashPlan server procses on the CentOS VM.
```
/usr/local/crashplan/bin/CrashPlanEngine start
```
Run the CrashPlan client on your local client. At this point you should be ab

## Mount NFS Shares from FreeNAS
At this point you should have a working CrashPlan, but with nothing to back up as CrashPlan only has access to the new CentOS VM and not all the files on your FreeNAS server. In order to provide access to those files we are going to share them using NFS and mount them on the CentOS VM.

On the CentOS VM
```
yum install nfs-utils
yum install nfs4-acl-tools
```
Test proper mounting of NFS shares (where 192.168.1.2 should be replaced with the IP of the FreeNAS Server):
```
mkdir /mnt/freenas-volume1-Documents
mount -t nfs 192.168.1.2:/mnt/volume1/Documents /mnt/freenas-volume1-Documents
```

If this worked you should now be able to see the contents of your NFS share in your CentOS VM.

Edit `/etc/fstab` to include perm mount

192.168.1.2:/mnt/volume1/Documents /mnt/freenas-volume1-Documents nfs rsize=8192,wsize=8192,timeo=14,intr

Per https://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-nfs-client-config.html 




