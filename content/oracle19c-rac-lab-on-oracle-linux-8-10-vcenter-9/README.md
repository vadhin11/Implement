# Oracle 19c RAC

This guide walks through a complete Oracle RAC 19c lab setup from start to finish. All key steps are aligned with my upcoming YouTube videos, allowing you to follow along easily while building your own environment. Each section is structured to be simple, practical, and time-saving, so you can quickly understand and implement Oracle RAC in a lab setup.

## Table of Contents
- [Oracle 19c RAC](#oracle-19c-rac)
  * [Requirements](#requirements)
  * [Setup VM Network](#setup-vm-network)
    + [NAT](#nat)
    + [Hostonly](#hostonly)
  * [OS Installation](#os-installation)
    + [Network and Host](#network-and-host)
    + [Software Selection](#software-selection)
    + [Root Password](#root-password)
    + [User Creation](#user-creation)
    + [Begin Installation](#begin-installation)
    + [Post Installation](#post-installation)
  * [Format Oracle Binaries Disk](#format-oracle-binaries-disk)
    + [Create LVM](#create-lvm)
    + [Make Files system](#make-files-system)
    + [Add to fstab](#add-to-fstab)
  * [Setup Prerequisites](#setup-prerequisites)
    + [Prerequisites installation](#prerequisites-installation)
    + [Create Env Variables](#create-env-variables)
    + [Disable Unnecessary Services](#disable-Unnecessary-services)
  * [Setup Cluster Network](#setup-cluster-network)
    + [Cluster Network](#cluster-network)
    + [Host file](#host-file)
  * [Setup DNS](#setup-dns)
    + [Install BIND9](#install-bind9)
    + [Configure BIND](#configure-bind)
    + [Forward Backward Zones](#forward-backward-zones)
    + [Test DNS Configuration](#test-dns-configuration)
    + [Use DNS](#use-dns)
  * [Clone VM](#clone-vm)
    + [Clone node1 to node2 ](#clone-node1-to-node2)
    + [Edit Network](#edit-network)
  * [ASM Shared Disks](#asm-shared-disks)
    + [Create VSAN Disk](#create-vsan-disk)
    + [Add Disk on Both Nodes](#add-disk-on-both-nodes)
  * [Format Disks](#format-disks)
  * [Setup GI + ASM AFD](#setup-gi---asm-afd)
    + [Upload files](#upload-files)
    + [Unzip files](#unzip-files)
    + [Prerequisites](#prerequisites)
    + [Installation](#installation)
    + [Check Cluster Status](#check-cluster-status)
  * [Database Installation](#database-installation)
    + [Install DB Software Only](#install-db-software-only)
    + [Create Data,FRA](#create-data-fra)
    + [Create DB](#create-db)
  * [Patch DB](#patch-db)
    + [Node 1](#node-1)
      - [Check](#check)
      - [Analyze](#analyze)
      - [Apply](#apply)
    + [Node 2](#node-2)
      - [Check](#check-1)
      - [Analyze](#analyze-1)
      - [Apply](#apply-1)
    + [Data Patch](#data-patch)
    + [Check Version](#check-version)
  * [Manage Cluster Services](#manage-cluster-services)
  * [SQL commands](#sql-commands)
  * [References](#references)


## Requirements 
1.  Oracle Linux 8.10
2.  Oracle 19.3 Grid Infrastructure (GI)
4.  Oracle 19.3 Oracle Database (DB)
5.  Vcenter version 9
6.  Vcenter Network
	1. Host only subnet 10.10.10.0/24 (Private Subnet)
	2. NAT subnet 172.16.2.0/24 (Public Subnet)
8.  Virtual Machine Resource Allocation:
    1.  Node1
        1.  32 GB VM RAM
        2.  8 Cores
        3.  128 GB OS VM DISK
        4.  64 GB GRID & DB Binaries VM DISK
        5.  2 VNIC Private/Public
    2.  Node2
        1.  32 GB VM RAM
        2.  8 Cores
        3.  128 GB OS VM DISK
        4.  64 GB GRID & DB Binaries VM DISK
        5.  2 VNIC Private/Public
9. Shared Disk (VSAN)
	1. 64 GB ASM shared VM disks.
  2. 64 GB FRA shared VM disks.
  3. 64 GB DATA shared VM disks.
10. 2 NIC (Network Interface Card) , 4 NIC (Redundant) 
11. 2 Network Switches, 4 Switches (redundant) 
12. DNS server (self DNS for lab)

## Setup VM Network
### NAT 
Change NAT subnet to `172.16.2.0/24` DHCP as well

### Hostonly
Change HostOnly to `10.10.10.0/24` 

## OS Installation 

After download the Oracle 8.10, you can create a vm in Vsphere and select the iso as virutal CD/DVD then start the VM that you create and boot up the ISO media and follow the below steps:

selete a creation type VM

![](picture/selete_a_creation_type_VM.png)

![](picture/provisioning_VM.png)

Click Power on VM:

select `Install Oracle Linux 8.10.0`

![](picture/Install_Oracle_Linux_8.png)

`Welcome to Oracle Linux 8.10` installation : then press continue ([ref-link](https://docs.oracle.com/en/operating-systems/oracle-linux/8/install/))

![](picture/welcome_to_Oracle_Linux.png)

[document](https://docs.oracle.com/en/operating-systems/oracle-linux/8/install/install-InstallingOracleLinuxinGraphicsMode.html#graphics-mode)

![](picture/Installation_summary_init.png)

### Installation destination [link](https://docs.oracle.com/en/operating-systems/oracle-linux/8/install/install-DefaultDiskPartitionLayout.html)

![](picture/Installation_Destination_crop.png)

select custom and press `Done`

![](picture/Installation_Destination_1.png)

custom 

![](picture/Installation_Destination_2.png)

`Accept` 

![](picture/Installation_Destination_3.png)

### Network and Host

![](picture/Network_HostName_crop.png)

![](picture/Network_HostName_1.png)

![](picture/Network_HostName_2.png)

![](picture/Network_HostName_3.png)

![](picture/Network_HostName_4.png)

### Software Selection

![](picture/Software_Selection_crop.png)

![](picture/Software_Selection.png)


- Performance Tools
- Legacy Linux Compatibility 
- Development Tools
- Graphic Administration Tools 
- System Tools

### Root Password 

![](picture/Root_Password_crop.png)

![](picture/Root_Password.png)


### User Creation

![](picture/User_Creation_crop.png)


![](picture/User_Creation.png)


### Begin Installation

![](picture/Begin_Installation.png)

![](picture/Installation_Progress.png)

![](picture/Reboot_System.png)

![](picture/License.png)

![](picture/License2.png)

![](picture/License3.png)


### Post Installation

* Remove screen lock and sleep.
* Increase terminal font size.
* Auto connect network interfaces.
* Add static network IP.
* Install guest addition tools 

enable repos

```bash
dnf config-manager --set-enabled ol8_appstream
dnf config-manager --set-enabled ol8_codeready_builder
dnf install oracle-epel-release-el8.x86_64 -y
```

```bash
sudo dnf install kernel-uek-devel-$(uname -r) gcc binutils automake make perl bzip2 elfutils-libelf-devel htop -y
```
```bash
sudo dnf install -y bc binutils libcap libstdc++ libstdc++-devel dtrace elfutils-libelf elfutils-libelf-devel fontconfig-devel glibc glibc-devel ksh libaio libaio-devel libXrender.x86_64 libXrender-devel.x86_64 libX11 libXau libXi libXtst libgcc librdmacm-devel libstdc++ libstdc++-devel libxcb net-tools nfs-utils python3 python3-configshell python3-rtslib python3-six targetcli smartmontools sysstat gcc unixODBC libnsl libnsl.i686 libnsl2-devel.i686 libnsl2-devel.x86_64 libnsl2.x86_64 libnsl2.i686
```


## Format Oracle Binaries Disk
### Create LVM

Login as root list all the avilable disks on the OS:
```bash
lsblk
```
```log
[root@oracle61 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:2    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
[root@oracle61 ~]#
```

formate the disk using below command:

```bash
fdisk /dev/sdb
```
```log
[root@oracle61 ~]# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-134217727, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-134217727, default 134217727):

Created a new partition 1 of type 'Linux' and of size 64 GiB.

Command (m for help): p
Disk /dev/sdb: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x2cc9ccab

Device     Boot Start       End   Sectors Size Id Type
/dev/sdb1        2048 134217727 134215680  64G 83 Linux

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): p
Disk /dev/sdb: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x2cc9ccab

Device     Boot Start       End   Sectors Size Id Type
/dev/sdb1        2048 134217727 134215680  64G 8e Linux LVM

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@oracle61 ~]#
```

Create PV 
```bash
pvcreate /dev/sdb1
pvs
```
```log
[root@oracle61 ~]# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
[root@oracle61 ~]# pvs
  PV         VG Fmt  Attr PSize   PFree
  /dev/sda3  ol lvm2 a--  126.41g      0
  /dev/sdb1     lvm2 ---  <64.00g <64.00g
[root@oracle61 ~]#
```

Create VG

```bash
vgcreate db /dev/sdb1
vgs
```
```log
[root@oracle61 ~]# vgcreate db /dev/sdb1
  Volume group "db" successfully created
[root@oracle61 ~]# vgs
  VG #PV #LV #SN Attr   VSize   VFree
  db   1   0   0 wz--n- <64.00g <64.00g
  ol   1   3   0 wz--n- 126.41g      0
[root@oracle61 ~]#
```

Create LV 

```bash
lvcreate db -n u01 -l 100%FREE
lvs
```
```log
[root@oracle61 ~]# lvcreate db -n u01 -l 100%FREE
  Logical volume "u01" created.
[root@oracle61 ~]# lvs
  LV   VG Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  u01  db -wi-a----- <64.00g
  home ol -wi-ao---- <53.61g
  root ol -wi-ao----  60.00g
  swap ol -wi-ao----  12.80g
[root@oracle61 ~]#
```

### Make Files system

```bash
mkfs -t xfs /dev/mapper/db-u01
fsck -N /dev/db/u01
```
```log
[root@oracle61 ~]# mkfs -t xfs /dev/mapper/db-u01
meta-data=/dev/mapper/db-u01     isize=512    agcount=4, agsize=4194048 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=0 inobtcount=0
data     =                       bsize=4096   blocks=16776192, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=8191, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@oracle61 ~]# fsck -N /dev/db/u01
fsck from util-linux 2.32.1
[/usr/sbin/fsck.xfs (1) -- /dev/mapper/db-u01] fsck.xfs /dev/mapper/db-u01
[root@oracle61 ~]#
```

### Add to fstab

```bash
vim /etc/fstab
```
`add`
```
/dev/db/u01   /u01    xfs defaults        0 0
```

Mount the u01

```bash
umount /u01
```
```log
[root@oracle61 ~]# umount /u01
umount: /u01: no mount point specified.
```
```bash
systemctl daemon-reload
mkdir -p /u01
mount /u01
lsblk
mount | grep /u01
```
```log
[root@oracle61 ~]# mkdir -p /u01
[root@oracle61 ~]# mount /u01
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
[root@oracle61 ~]# systemctl daemon-reload
[root@oracle61 ~]# mount /u01
mount: /u01: /dev/mapper/db-u01 already mounted on /u01.
[root@oracle61 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:2    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
└─sdb1        8:17   0    64G  0 part
  └─db-u01  252:3    0    64G  0 lvm  /u01
[root@oracle61 ~]# mount | grep /u01
/dev/mapper/db-u01 on /u01 type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota)
[root@oracle61 ~]#
```

Check the list of disks 

```bash
lsblk
```
```bash
[root@oracle61 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:2    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
└─sdb1        8:17   0    64G  0 part
  └─db-u01  252:3    0    64G  0 lvm
[root@oracle61 ~]#
```

Reboot to check if everything is ok

```bash
reboot
```


## Setup Prerequisites

### Prerequisites installation
check the Internet connectivity using ping :

```bash
# check internet connectivity 
ping www.google.com
```

check cache to download the metadata from online repo:

```bash
# check the cache
dnf makecache
```

Install prerequisites and ASM required packages:

```bash
# Search for preinstall package
dnf search preinstall-19c

# Install oracle prereqisites  
dnf install oracle-database-preinstall-19c.x86_64 -y 
```

this step is optional if you want to update the system:

```bash
# Optional 
dnf check-update # list the packages that need for update  
dnf update -y
dnf clean all
```

Create OS groups for asm administration and operation:

```bash
# Create ASM groups 
groupadd -g 54327 asmdba
groupadd -g 54328 asmoper
groupadd -g 54329 asmadmin
```

Add `asmdba` as secondary group to oracle user:

```bash
# add asmdba group to oracle user asmadmin
usermod -a -G  asmdba oracle
id oracle 
```

Create Grid user:

```bash
# create grid user 
useradd -u 54331 -g oinstall -G dba,asmdba,asmadmin,asmoper,racdba grid
id grid 
```

Change the password for Oracle and Grid user:

```bash
# create grid oracle user passwords 
passwd oracle
echo password | passwd oracle --stdin
echo password | passwd grid --stdin
passwd grid
```

Create the Directories for the Oracle Grid installation 

```bash
mkdir -p /u01/19c/oracle/ora_base/db_home
mkdir -p /u01/19c/grid/grid_home
mkdir -p /u01/19c/grid/grid_base
chown -R oracle:oinstall /u01
chown -R grid:oinstall /u01/19c/grid/
chmod -R 775 /u01

```

Check the NTP service 

```bash
# chrony both servers
vim /etc/chrony.conf
```

`add`

```conf
server 0.jp.pool.ntp.org iburst
server 1.jp.pool.ntp.org iburst
server 2.jp.pool.ntp.org iburst
server 3.jp.pool.ntp.org iburst
```
command check service

```bash
systemctl enable chronyd
systemctl restart chronyd
chronyc sources
systemctl status chronyd

```

set secure linux to permissive 

```bash
# change SELINUX=enforcing to SELINUX=permissive
sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config
# Or Disabled it 
sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
cat /etc/selinux/config
# Forse the change  
setenforce Permissive
```
create limitation in security forlder for grid user 
```bash
cp /etc/security/limits.d/oracle-database-preinstall-19c.conf /etc/security/limits.d/grid-database-preinstall-19c.conf
```

rename oracle with grid in this file grid-database-preinstall-19c.conf

```bash
# use vim 
vim /etc/security/limits.d/grid-database-preinstall-19c.conf
```
`command`
```
:%s/oracle/grid/g
:x
```

allow the port 1521 port in the Linux firewall [ref-link](https://forums.oracle.com/ords/apexds/post/firewall-clearance-for-oracle-listener-port-1521-0273)

allow the traffic for the port 1521
```bash
firewall-cmd --permanent --add-port=1521/tcp
firewall-cmd --reload
firewall-cmd --list-ports
```
example expected 
```log
[root@oracle61 ~]# firewall-cmd --list-ports
1521/tcp
[root@oracle61 ~]
```
or stop the firewall and disable it.
```bash
systemctl stop firewalld
systemctl disable firewalld
```

### Create Env Variables
Switch to the `grid` user.<br>
Create a backup of the Grid user's .bash_profile.

```bash
su - grid
cp .bash_profile .bash_profile.bkp
```
Configure the ORA_SID environment variable on each node as follows:<br>
node1
```bash
export ORA_SID=+ASM1
```
node2
```bash
export ORA_SID=+ASM2
```

Create Grid Environment Configuration File

Copy and paste the following script to create a Grid environment file in the Grid user’s home directory:

```bash
cat > /home/grid/.grid_env <<EOF
host=$(hostname -s)
if [ \$host == "oracle61" ]; then
    ORA_SID=+ASM1
elif [ \$host == "oracle62" ]; then
    ORA_SID=+ASM2
else
 echo "Host not meet the option $host"
fi

# User specific environment and startup programs

ORACLE_SID=\$ORA_SID; export ORACLE_SID
ORACLE_BASE=/u01/19c/grid/grid_base; export ORACLE_BASE
ORACLE_HOME=/u01/19c/grid/grid_home; export ORACLE_HOME
ORACLE_TERM=xterm; export ORACLE_TERM
JAVA_HOME=/usr/bin/java; export JAVA_HOME
TNS_ADMIN=\$ORACLE_HOME/network/admin; export TNS_ADMIN

PATH=.:\${JAVA_HOME}/bin:\${PATH}:\$HOME/bin:\$ORACLE_HOME/bin
PATH=\${PATH}:/usr/bin:/bin:/usr/local/bin
export PATH
export CV_ASSUME_DISTID=OEL7.8
umask 022
EOF
```

Add the Grid environment file to the Grid user’s profile

```bash
echo '. ~/.grid_env' >> /home/grid/.bash_profile
```

Fix ownership (if created by root)
```bash
chown grid:oinstall /home/grid/.grid_env
```

Apply changes

```bash
source .bash_profile 
env | grep -i "tns\|oracle"
exit
```

example expected

```log
[grid@oracle61 ~]$ source .bash_profile
[grid@oracle61 ~]$ env | grep -i "tns\|oracle"
ORACLE_SID=+ASM1
ORACLE_BASE=/u01/19c/grid/grid_base
ORACLE_HOME=/u01/19c/grid/grid_home
HOSTNAME=oracle61.p2ok.site
ORACLE_TERM=xterm
TNS_ADMIN=/u01/19c/grid/grid_home/network/admin
[grid@oracle61 ~]$ exit
logout
[root@oracle61 ~]#
```
Switch to the `oracle` user.<br>
Create a backup of the Oracle user's .bash_profile.

```bash
su - oracle 
cp .bash_profile .bash_profile.bkp
```

Configure the ORA_SID environment variable on each node as follows:<br>
node1
```bash
export ORA_SID=PROD11
```
node2
```bash
export ORA_SID=PROD12
```

```bash
cat > /home/oracle/.db19_env << EOF
host=\$(hostname -s)
if [ \$host == "oracle61" ]; then 
    ORA_SID=PROD1
elif [ \$host == "oracle62" ]; then
    ORA_SID=PROD2
else 
 echo "Host not meet the option $host"
fi

ORACLE_HOSTNAME=\$HOSTNAME; export ORACLE_HOSTNAME
ORACLE_SID=\$ORA_SID; export ORACLE_SID
ORACLE_UNQNAME=prod; export ORACLE_UNQNAME
ORACLE_BASE=/u01/19c/oracle/ora_base; export ORACLE_BASE
ORACLE_HOME=/u01/19c/oracle/ora_base/db_home; export ORACLE_HOME
ORACLE_TERM=xterm; export ORACLE_TERM

JAVA_HOME=/usr/bin/java; export JAVA_HOME
NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"; export NLS_DATE_FORMAT
TNS_ADMIN=\$ORACLE_HOME/network/admin; export TNS_ADMIN
PATH=.:\${JAVA_HOME}/bin:\${PATH}:\$HOME/bin:\$ORACLE_HOME/bin
PATH=\${PATH}:/usr/bin:/bin:/usr/local/bin
export PATH

LD_LIBRARY_PATH=\$ORACLE_HOME/lib
LD_LIBRARY_PATH=\${LD_LIBRARY_PATH}:\$ORACLE_HOME/oracm/lib
LD_LIBRARY_PATH=\${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export LD_LIBRARY_PATH

CLASSPATH=\$ORACLE_HOME/JRE:\$ORACLE_HOME/jlib:\$ORACLE_HOME/rdbms/jlib:\$ORACLE_HOME/network/jlib
export CLASSPATH

TEMP=/tmp ;export TMP
TMPDIR=\$tmp ; export TMPDIR

export CV_ASSUME_DISTID=OEL7.8
EOF
```

Add the Grid environment file to the Grid user’s profile

```bash
cat  >> /home/oracle/.bash_profile <<EOF
source /home/oracle/.db19_env
alias db19_env='. /home/oracle/.db19_env'
EOF
```

Fix ownership (if created by root)

```bash  
chown oracle:oinstall /home/oracle/.bash_profile
chown oracle:oinstall /home/oracle/.db19_env
```

Apply changes from oracle user

```bash
source /home/oracle/.bash_profile
env | grep ORACLE_
exit
```
example expected

```log
[oracle@oracle61 ~]$ env | grep ORACLE_
ORACLE_UNQNAME=prod
ORACLE_SID=PROD11
ORACLE_BASE=/u01/19c/oracle/ora_base
ORACLE_HOME=/u01/19c/oracle/ora_base/db_home
ORACLE_TERM=xterm
ORACLE_HOSTNAME=oracle61.p2ok.site
[oracle@oracle61 ~]$ exit
logout
[root@oracle61 ~]#
```

### Disable Unnecessary Services
Stop Services (**Root** Privilege Required)

```bash
systemctl stop tuned.service ktune.service
systemctl stop firewalld.service
systemctl stop postfix.service
systemctl stop avahi-daemon.service
systemctl stop avahi-daemon.socket

systemctl stop atd.service
systemctl stop bluetooth.service
systemctl stop wpa_supplicant.service
systemctl stop accounts-daemon.service
systemctl stop ModemManager.service
systemctl stop debug-shell.service
systemctl stop rtkit-daemon.service
systemctl stop rpcbind.service
systemctl stop rpcbind.socket
systemctl stop rngd.service
systemctl stop upower.service
systemctl stop rhsmcertd.service
systemctl stop colord.service
systemctl stop libstoragemgmt.service
systemctl stop ksmtuned.service
systemctl stop brltty.service


systemctl disable tuned.service ktune.service
systemctl disable firewalld.service
systemctl disable postfix.service
systemctl disable avahi-daemon.socket
systemctl disable avahi-daemon.service
systemctl disable bluetooth.service
systemctl disable wpa_supplicant.service
systemctl disable accounts-daemon.service
systemctl disable atd.service cups.service
systemctl disable ModemManager.service
systemctl disable debug-shell.service
systemctl disable rpcbind.service
systemctl disable rpcbind.socket
systemctl disable rngd.service
systemctl disable upower.service
systemctl disable rhsmcertd.service
systemctl disable rtkit-daemon.service
systemctl disable mcelog.service
systemctl disable colord.service
systemctl disable libstoragemgmt.service
systemctl disable ksmtuned.service
systemctl disable brltty.service

```

## Setup Cluster Network

To configure a 2-node Oracle RAC cluster, allocate:

* 7 Public IPs
* 2 Private IPs (for interconnect)

### Cluster Network 

The Current Lab network setup will be simple as shown below

![](attachments/Pasted%20image%2020220602231907.png)


 For maximum HA and the Best Practice to have redundant network setup, you need  4 NIC 4 Switches
  
![](attachments/Pasted%20image%2020220602231751.png)

### Host file 
add below to host file `/etc/hosts`
```conf
172.16.2.61   oracle61.p2ok.site       oracle61 #DNS
172.16.2.62   oracle62.p2ok.site       oracle62 #DNS
10.10.10.61   oracle61-priv.p2ok.site   oracle61-priv
10.10.10.62   oracle62-priv.p2ok.site   oracle62-priv
172.16.2.63   oracle61-vip.p2ok.site            oracle61-vip #DNS
172.16.2.64   oracle62-vip.p2ok.site            oracle62-vip #DNS
#172.16.2.65    oracle-cluster-scan.p2ok.site   oracle-cluster-scan #DNS
#172.16.2.66    oracle-cluster-scan.p2ok.site   oracle-cluster-scan #DNS
#172.16.2.67    oracle-cluster-scan.p2ok.site   oracle-cluster-scan #DNS
```

## Setup DNS
Best practice: Use an external DNS server (not on RAC nodes)

### Install BIND9

```bash

# DNS Server
dnf makecache
dnf install bind bind-utils -y
hostnamectl
```

### Configure BIND

```bash
systemctl stop named
systemctl disable named 
```

backup the file 
```bash
cp /etc/named.conf /etc/named.conf.bkp
```
script for the named.conf based on your system

```bash
export DNS_IP="172.16.2.61"
export DNS_DOMAIN="p2ok.site"
export DNS_NETWORK="172.16.2.0/24"
export DNS_BACKWARD="2.16.172.in-addr.arpa"
export DNS_FORWARD=$DNS_DOMAIN
export DNS_BACKWARD_FILE="backward.$DNS_DOMAIN"
export DNS_FORWARD_FILE="forward.$DNS_DOMAIN"
export DNS_HOSTNAME="oracle61"
export DNS_FQDN=$DNS_HOSTNAME.$DNS_DOMAIN

cat > /etc/named.conf <<EOF
options {
	listen-on port 53 { 127.0.0.1; $DNS_IP; };
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	secroots-file	"/var/named/data/named.secroots";
	recursing-file	"/var/named/data/named.recursing";
	allow-query     { localhost; $DNS_NETWORK; };

	recursion yes;

	#dnssec-enable yes;
	dnssec-validation yes;

	managed-keys-directory "/var/named/dynamic";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";

	/* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
	include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

//forward zone
zone "$DNS_DOMAIN" IN {
     type master;
     file "$DNS_FORWARD_FILE";
     allow-update { none; };
     allow-query { any; };
};

//backward zone or reverse zone
zone "$DNS_BACKWARD" IN {
     type master;
     file "$DNS_BACKWARD_FILE";
     allow-update { none; };
     allow-query { any; };
};
EOF
```

### Backward Zone

```conf
cat  > /var/named/forward.p2ok.site <<EOF
\$TTL 86400
@ IN SOA oracle61.p2ok.site. admin.p2ok.site. (
                                                2021040300 ;Serial
                                                3600 ;Refresh
                                                1800 ;Retry
                                                604800 ;Expire
                                                86400 ;Minimum TTL
)

;Name Server Information
@ IN NS oracle61.p2ok.site.
@ IN A  172.16.2.61
@ IN A  172.16.2.62
@ IN A  172.16.2.63
@ IN A  172.16.2.64
@ IN A  172.16.2.65
@ IN A  172.16.2.66
@ IN A  172.16.2.67

;IP Address for Name Server
oracle61 IN A 172.16.2.61

;Mail Server MX (Mail exchanger) Record
;oracle61.p2ok.site IN MX 10 mail.oracle61.p2ok.site

;A Record for the following Host name
oracle61  IN   A   172.16.2.61
oracle62  IN   A   172.16.2.62
oracle61-vip  IN  A  172.16.2.63
oracle62-vip  IN  A  172.16.2.64
oracle-cluster-scan  IN  A  172.16.2.65
oracle-cluster-scan  IN  A  172.16.2.66
oracle-cluster-scan  IN  A  172.16.2.67

;CNAME Record
;ftp  IN   CNAME ftp.oracle61.p2ok.site.
EOF
```

### Backward Zone

```conf
cat > /var/named/backward.p2ok.site <<EOF
\$TTL 86400
@ IN SOA oracle61.p2ok.site. admin.p2ok.site. (
                                            2021040300 ;Serial
                                            3600 ;Refresh
                                            1800 ;Retry
                                            604800 ;Expire
                                            86400 ;Minimum TTL
)
;Name Server Information
@ IN NS oracle61.p2ok.site.
oracle61     IN      A       172.16.2.61

;Reverse lookup for Name Server
35 IN PTR oracle61.p2ok.site.

;PTR Record IP address to Hostname
61      IN      PTR     oracle61.p2ok.site.
62      IN      PTR     oracle62.p2ok.site.
63      IN      PTR     oracle61-vip.p2ok.site
64      IN      PTR     oracle62-vip.p2ok.site
65      IN      PTR     oracle-cluster-scan.p2ok.site
66      IN      PTR     oracle-cluster-scan.p2ok.site
67      IN      PTR     oracle-cluster-scan.p2ok.site
EOF
```

change the owner 

```bash
chown named:named /var/named/forward.p2ok.site
chown named:named /var/named/backward.p2ok.site
```


### Test DNS Configuration
```bash
# check the configurations 
named-checkconf
named-checkzone p2ok.site /var/named/forward.p2ok.site
named-checkzone 172.16.2.61 /var/named/forward.p2ok.site
systemctl start named
systemctl enable named
```

### Use DNS 
edit resolv.conf
```bash
cat > /etc/resolv.conf <<EOF
search p2ok.site
nameserver 172.16.2.61
EOF
```

edit the network profiles may different in your system

```bash
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```
add this line to this file 

```conf
DNS1=172.16.2.61
```

allow the dns traffic 

```bash
firewall-cmd --add-service=dns --zone=public  --permanent
firewall-cmd --reload
```
or stop don't use firewalld.service
```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

test from the client
```bash
nslookup oracle61.p2ok.site
nslookup oracle62.p2ok.site

ping oracle61
ping oracle62

```


## Clone VM

### Clone node1 to node2 

Select first node VM from vSphere Client and right click VM select `Clone` then click to `Clone to Virtual Machine`:


![](picture/Clone_to_Virtual_Machine.png)

click finish

![](picture/Clone_Existing_Virtual_Machine.png)

then press power on VM


### Edit Network
Start the second node and update the following:

#### Update Hostname

```bash
hostnamectl set-hostname oracle62.p2ok.site
hostnamectl
```
example expected

```log
[root@oracle62-p2ok-site ~]# hostnamectl set-hostname oracle62.p2ok.site
[root@oracle62-p2ok-site ~]# hostnamectl
   Static hostname: oracle62.p2ok.site
         Icon name: computer-vm
           Chassis: vm
        Machine ID: aa204701209e40cc8db809b0d45ac506
           Boot ID: e1d3e83c5bdc404cb2a56d26930bcb35
    Virtualization: vmware
  Operating System: Oracle Linux Server 8.10
       CPE OS Name: cpe:/o:oracle:linux:8:10:server
            Kernel: Linux 5.15.0-318.199.3.2.el8uek.x86_64
      Architecture: x86-64
[root@oracle62-p2ok-site ~]#
```
#### Update Network Configuration
* Change Public IP
* Change Private (Interconnect) IP

#### Disable Local DNS called 
`named.service`
```bash
systemctl stop named.service
systemctl disable named.service
``` 

## ASM Shared Disks 
### Create VSAN Disk

Create Virtual Disks
1. Go to vCenter → VM → Edit Settings
2. Click Add New Device → Hard Disk
3. Select:
    - New Hard Disk
    - Disk size as required
    - Disk Provisioning: Thick (Eager Zeroed) (recommended for RAC)
    - 3 disks
      * 64G for CRS
      * 64G for FRA
      * 64G for DATA
4. Set **Sharing** → `Multi-writer`

![](picture/VSAN_Disk.png)

### Add Disk on Both Nodes

1. Edit both Node 
2. Add existing disk:
    - Add Existing Hard Disk
    - Select the same VMDK used in both Node

![](picture/Attach_disk.png)

## Format Disks 

you should see new three disk on both node
```log
[root@oracle61 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:3    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
└─sdb1        8:17   0    64G  0 part
  └─db-u01  252:2    0    64G  0 lvm  /u01
sdc           8:32   0    64G  0 disk
sdd           8:48   0    64G  0 disk
sde           8:64   0    64G  0 disk
[root@oracle61 ~]#
```

Start First Node and using fdisk to format the three disks 
```bash
fdisk /dev/sdc
fdisk /dev/sdd
fdisk /dev/sde
```

```log
[root@oracle61 ~]# fdisk /dev/sdc

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x2311393d.

Command (m for help):n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-134217727, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-134217727, default 134217727):

Created a new partition 1 of type 'Linux' and of size 64 GiB.

Command (m for help): p
Disk /dev/sdc: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x2311393d

Device     Boot Start       End   Sectors Size Id Type
/dev/sdc1        2048 134217727 134215680  64G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@oracle61 ~]# fdisk /dev/sdd

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xfa4ad1ee.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-134217727, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-134217727, default 134217727):

Created a new partition 1 of type 'Linux' and of size 64 GiB.

Command (m for help): p
Disk /dev/sdd: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfa4ad1ee

Device     Boot Start       End   Sectors Size Id Type
/dev/sdd1        2048 134217727 134215680  64G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@oracle61 ~]# fdisk /dev/sde

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x6b6667c4.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-134217727, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-134217727, default 134217727):

Created a new partition 1 of type 'Linux' and of size 64 GiB.

Command (m for help): p
Disk /dev/sde: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x6b6667c4

Device     Boot Start       End   Sectors Size Id Type
/dev/sde1        2048 134217727 134215680  64G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@oracle61 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:3    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
└─sdb1        8:17   0    64G  0 part
  └─db-u01  252:2    0    64G  0 lvm  /u01
sdc           8:32   0    64G  0 disk
└─sdc1        8:33   0    64G  0 part
sdd           8:48   0    64G  0 disk
└─sdd1        8:49   0    64G  0 part
sde           8:64   0    64G  0 disk
└─sde1        8:65   0    64G  0 part
[root@oracle61 ~]#
```

then startup the second node the shared disks should be formated there and use command
```bash
partprobe
lsblk
```
```log
[root@oracle62 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:3    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
└─sdb1        8:17   0    64G  0 part
  └─db-u01  252:2    0    64G  0 lvm  /u01
sdc           8:32   0    64G  0 disk
sdd           8:48   0    64G  0 disk
sde           8:64   0    64G  0 disk
[root@oracle62 ~]# partprobe
```
```log
[root@oracle62 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:3    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
└─sdb1        8:17   0    64G  0 part
  └─db-u01  252:2    0    64G  0 lvm  /u01
sdc           8:32   0    64G  0 disk
└─sdc1        8:33   0    64G  0 part
sdd           8:48   0    64G  0 disk
└─sdd1        8:49   0    64G  0 part
sde           8:64   0    64G  0 disk
└─sde1        8:65   0    64G  0 part
[root@oracle62 ~]#
```

## Setup GI + ASM AFD

Use **ASM Filter Driver (AFD)** to manage ASM disks.

### Upload Files

Upload required installation files to `/u01`

#### Download from URL
[Oracle Database 19c for Linux (Intel x86-64) (64-bit)](https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html)<br>
[Patch Search](https://support.oracle.com/support/?page=sptemplate&sptemplate=cp-patches-updates-view-more)<br>
[Simple Search](https://updates.oracle.com/Orion/PatchDetails/switch_to_simple)

#### SCP (Send file to remote server)
```bash
scp p6880880_190000_Linux-x86-64.zip grid@oracle61://home/grid/19c/
scp LINUX.X64_193000_grid_home.zip grid@oracle61://home/grid/19c/
scp p33803476_190000_Linux-x86-64.zip grid@oracle61://home/grid/19c/
```


### Unzip files 
login as `grid` user
```bash
unzip /u01/LINUX.X64_193000_grid_home.zip -d $ORACLE_HOME
mv $ORACLE_HOME/OPatch/ $ORACLE_HOME/OPatch_BKP
unzip /u01/p6880880_190000_Linux-x86-64.zip -d $ORACLE_HOME
unzip p33803476_190000_Linux-x86-64.zip -d /u01
```
```log
[grid@oracle61 19c]$ ls -l .
total 5658652
-rw-r--r--. 1 grid oinstall 2889184573 Mar 25 21:15 LINUX.X64_193000_grid_home.zip
-rw-r--r--. 1 grid oinstall 2832372532 Mar 25 23:38 p33803476_190000_Linux-x86-64.zip
-rw-r--r--. 1 grid oinstall   72896144 Mar 26 00:19 p6880880_190000_Linux-x86-64.zip
[grid@oracle61 19c]$ echo $ORACLE_HOME
/u01/19c/grid/grid_home
[grid@oracle61 19c]$ unzip LINUX.X64_193000_grid_home.zip -d $ORACLE_HOME
[grid@oracle61 19c]$ $ORACLE_HOME/OPatch/opatch version
OPatch Version: 12.2.0.1.17

OPatch succeeded.
[grid@oracle61 19c]$ mv $ORACLE_HOME/OPatch/ $ORACLE_HOME/OPatch_BKP
[grid@oracle61 19c]$ unzip -q p6880880_190000_Linux-x86-64.zip -d $ORACLE_HOME
[grid@oracle61 19c]$ $ORACLE_HOME/OPatch/opatch version
OPatch Version: 12.2.0.1.49

OPatch succeeded.
[grid@oracle61 19c]$ unzip p33803476_190000_Linux-x86-64.zip -d /u01
```
### Prerequisites 

#### Generate SSH Key (Both Nodes)

```bash
ssh-keygen -t rsa
```

Copy SSH Key

From Node2 → Node1
```bash
ssh-copy-id grid@oracle61.p2ok.site
```

From Node1 → Node2
```bash
ssh-copy-id grid@oracle62.p2ok.site
```

check Passwordless SSH from both nodes

```bash
ssh grid@oracle62 date
ssh grid@oracle61 date
```

login as `root` user in both nodes <br>
verify NTP time
```bash
systemctl restart chronyd.service
chronyc sources
```
```log
[root@oracle62 ~]# systemctl restart chronyd.service
[root@oracle62 ~]# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^? 47.87.71.223                  0   6     0     -     +0ns[   +0ns] +/-    0ns
^? 202.28.33.225                 0   6     0     -     +0ns[   +0ns] +/-    0ns
^? 124.109.2.169                 0   6     0     -     +0ns[   +0ns] +/-    0ns
^? vps-tyo3.orleans.ddnss.de     0   6     0     -     +0ns[   +0ns] +/-    0ns
^? 172-233-91-137.ip.linode>     2   6     1     2  +7981us[+7981us] +/-   78ms
^? x.ns.gin.ntt.net              2   6     1     2  +7916us[+7916us] +/-  115ms
^? tok2.jp.ntp.li                4   6     3     0    -12ms[  -12ms] +/-  161ms
[root@oracle62 ~]#
```
fix the error INS-06006

```bash
cp -p /usr/bin/scp /usr/bin/scp.bkp
echo "/usr/bin/scp.bkp -T \$*" > /usr/bin/scp
```

fix the error PRVG-11550 and PRVG-10467 : The default Oracle Inventory group could not be determined.

```bash
export ORACLE_HOME=/u01/19c/grid/grid_home
rpm -ivh $ORACLE_HOME/cv/rpm/cvuqdisk*.rpm
scp $ORACLE_HOME/cv/rpm/cvuqdisk*.rpm root@oracle62://tmp
ssh oracle62
rpm -ivh /tmp/cvuqdisk*.rpm
```

```log
[root@oracle61 ~]# export ORACLE_HOME=/u01/19c/grid/grid_home
[root@oracle61 ~]# echo $ORACLE_HOME
/u01/19c/grid/grid_home
[root@oracle61 ~]# rpm -ivh $ORACLE_HOME/cv/rpm/cvuqdisk*.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Using default group oinstall to install package
Updating / installing...
   1:cvuqdisk-1.0.10-1                ################################# [100%]
[root@oracle61 ~]# scp $ORACLE_HOME/cv/rpm/cvuqdisk*.rpm root@oracle62://tmp
The authenticity of host 'oracle62 (172.16.2.62)' can't be established.
ECDSA key fingerprint is SHA256:3wrUVElAWZOUF04S9rwcsm+FSEsAv6+E8cgCW/vM/WA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'oracle62,172.16.2.62' (ECDSA) to the list of known hosts.
root@oracle62's password:
cvuqdisk-1.0.10-1.rpm                                   100%   11KB  11.9MB/s   00:00
[root@oracle61 ~]# ssh oracle62
root@oracle62's password:
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Thu Mar 26 00:08:07 2026 from 10.10.100.2
[root@oracle62 ~]# rpm -ivh /tmp/cvuqdisk*.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Using default group oinstall to install package
Updating / installing...
   1:cvuqdisk-1.0.10-1                ################################# [100%]
[root@oracle62 ~]#
```

fix the error PRVG-10048
change /etc/resolv.conf of both nodes to
```conf
search p2ok.site
nameserver      172.16.2.61
```

#### Verify Prerequisites (CVU)
login as `grid` user
```bash
$ORACLE_HOME/runcluvfy.sh stage -post crsinst -n oracle61,oracle62 -verbose
$ORACLE_HOME/oui/prov/resources/scripts/sshUserSetup.sh -user grid -hosts "oracle61 oracle62" -noPromptPassphrase -confirm -advanced
$ORACLE_HOME/runcluvfy.sh stage -pre crsinst -n oracle61,oracle62 -networks ens33:172.16.2.0:PUBLIC/ens34:10.10.10.0:cluster_interconnect -method root -verbose

```

### Installation

Keep the installer ready, then configure ASMFD (CRS voting disk) before proceeding. [link](https://docs.oracle.com/en/database/oracle/oracle-database/21/ladbi/configuring-asmfd-during-install.html#GUID-A82EE293-226B-431D-9888-4F957AE19234)

login as `root` user<br>
Label CRS Disk (ASMFD)
```bash
source /home/grid/.grid_env
export ORACLE_BASE=/tmp
asmcmd afd_label CRS1 /dev/sdc1 --init
asmcmd afd_lslbl /dev/sdc1
unset ORACLE_BASE
```

```log
[root@oracle61 ~]# source /home/grid/.grid_env
[root@oracle61 ~]# export ORACLE_BASE=/tmp
[root@oracle61 ~]# asmcmd afd_label CRS1 /dev/sdc1 --init
[root@oracle61 ~]# asmcmd afd_lslbl /dev/sdc1
--------------------------------------------------------------------------------
Label                     Duplicate  Path
================================================================================
CRS1                                  /dev/sdc1
[root@oracle61 ~]# unset ORACLE_BASE
[root@oracle61 ~]#
```

Remove Label (if wrong disk)
```bash
asmcmd afd_unlabel /dev/sdc1 --init
```

chown owner group disk do both nodes
```bash
blkid /dev/sdc1
blkid /dev/sdd1
blkid /dev/sde1
vi /etc/udev/rules.d/99-vmware-scsi-timeout.rules
udevadm control --reload-rules
udevadm trigger
ls -l /dev/sdc1
ls -l /dev/sdd1
ls -l /dev/sde1
```
expected output
```log
[root@oracle61 ~]# blkid /dev/sdd1
/dev/sdd1: PARTUUID="fa4ad1ee-01"
[root@oracle61 ~]# blkid /dev/sdc1
/dev/sdc1: LABEL="CRS1" TYPE="oracleasm" PARTUUID="2311393d-01"
[root@oracle61 ~]# blkid /dev/sde1
/dev/sde1: PARTUUID="6b6667c4-01"
[root@oracle61 ~]# vi /etc/udev/rules.d/99-oracle-asm.rules
[root@oracle61 ~]# cat /etc/udev/rules.d/99-oracle-asm.rules
ENV{ID_PART_ENTRY_UUID}=="2311393d-01", OWNER="grid", GROUP="asmadmin", MODE="0660"
ENV{ID_PART_ENTRY_UUID}=="fa4ad1ee-01", OWNER="grid", GROUP="asmadmin", MODE="0660"
ENV{ID_PART_ENTRY_UUID}=="6b6667c4-01", OWNER="grid", GROUP="asmadmin", MODE="0660"
[root@oracle61 ~]# udevadm control --reload-rules
[root@oracle61 ~]# udevadm trigger
[root@oracle61 ~]# ls -l /dev/sdc1
brw-rw----. 1 grid asmadmin 8, 33 Mar 26 19:00 /dev/sdc1
[root@oracle61 ~]# ls -l /dev/sdd1
brw-rw----. 1 grid asmadmin 8, 49 Mar 26 19:00 /dev/sdd1
[root@oracle61 ~]# ls -l /dev/sde1
brw-rw----. 1 grid asmadmin 8, 65 Mar 26 19:00 /dev/sde1
```

Run Grid Installer

Login as grid:

```bash
cd $ORACLE_HOME
./gridSetup.sh -applyPSU /u01/33803476/

```
Expected Output
```log
[grid@oracle61 ~]$ cd $ORACLE_HOME
[grid@oracle61 grid_home]$ ./gridSetup.sh -applyPSU /u01/33803476/
Preparing the home to patch...
Applying the patch /u01/33803476/...
Successfully applied the patch.
The log can be found at: /tmp/GridSetupActions2026-03-26_04-35-56PM/installerPatchActions_2026-03-26_04-35-56PM.log
Launching Oracle Grid Infrastructure Setup Wizard...
```
```log
[root@oracle61 ~]# firewall-cmd --permanent --add-port=22/tcp
success
[root@oracle61 ~]# firewall-cmd --permanent --add-port=42424/tcp
success
[root@oracle61 ~]# firewall-cmd --permanent --add-port=6200/tcp
success
[root@oracle61 ~]# firewall-cmd --permanent --add-port=2300-2310/tcp
success
[root@oracle61 ~]# firewall-cmd --permanent --add-port=2300-2310/udp
success
[root@oracle61 ~]# firewall-cmd --permanent --add-source=10.10.10.0/24
success
[root@oracle61 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="172.16.2.0/24" accept'
success
[root@oracle61 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.0/24" accept'
success
[root@oracle61 ~]# firewall-cmd --reload
success
[root@oracle61 ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33 ens34
  sources: 10.10.10.0/24
  services: cockpit dhcpv6-client ssh
  ports: 1521/tcp 22/tcp 42424/tcp 6200/tcp 2300-2310/tcp 2300-2310/udp
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
        rule family="ipv4" source address="10.10.10.0/24" accept
        rule family="ipv4" source address="172.16.2.0/24" accept
```
```log
[root@oracle62 ~]# firewall-cmd --permanent --add-port=22/tcp
success
[root@oracle62 ~]# firewall-cmd --permanent --add-port=42424/tcp
success
[root@oracle62 ~]# firewall-cmd --permanent --add-port=6200/tcp
success
[root@oracle62 ~]# firewall-cmd --permanent --add-port=2300-2310/tcp
success
[root@oracle62 ~]# firewall-cmd --permanent --add-port=2300-2310/udp
success
[root@oracle62 ~]# firewall-cmd --permanent --add-source=10.10.10.0/24
success
[root@oracle62 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="172.16.2.0/24" accept'
success
[root@oracle62 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.0/24" accept'
success
[root@oracle62 ~]# firewall-cmd --reload
success
[root@oracle62 ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33 ens34
  sources: 10.10.10.0/24
  services: cockpit dhcpv6-client ssh
  ports: 1521/tcp 22/tcp 42424/tcp 6200/tcp 2300-2310/tcp 2300-2310/udp
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
        rule family="ipv4" source address="10.10.10.0/24" accept
        rule family="ipv4" source address="172.16.2.0/24" accept
[root@oracle62 ~]#
```
```log
[root@oracle61 19c]# rpm -ivh oracleasm-support-3.1.1-4.el8.x86_64.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:oracleasm-support-3.1.1-4.el8    ################################# [100%]
Created symlink /etc/systemd/system/multi-user.target.wants/oracleasm.service → /usr/lib/systemd/system/oracleasm.service.
[root@oracle61 19c]# rpm -ivh oracleasmlib-3.1.1-1.el8.x86_64.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:oracleasmlib-3.1.1-1.el8         ################################# [100%]
```
```log
[grid@oracle61 ~]$ scp 19c/oracleasmlib-3.1.1-1.el8.x86_64.rpm oracle62://tmp/
oracleasmlib-3.1.1-1.el8.x86_64.rpm              100%   52KB  45.2MB/s   00:00
[grid@oracle61 ~]$ scp 19c/oracleasm-support-3.1.1-4.el8.x86_64.rpm oracle62://tmp/
oracleasm-support-3.1.1-4.el8.x86_64.rpm         100%   87KB  52.2MB/s   00:00
[grid@oracle61 ~]$
```
```log
[root@oracle62 ~]# rpm -ivh /tmp/oracleasm-support-3.1.1-4.el8.x86_64.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:oracleasm-support-3.1.1-4.el8    ################################# [100%]
Created symlink /etc/systemd/system/multi-user.target.wants/oracleasm.service → /usr/lib/systemd/system/oracleasm.service.
[root@oracle62 ~]# rpm -ivh /tmp/oracleasmlib-3.1.1-1.el8.x86_64.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:oracleasmlib-3.1.1-1.el8         ################################# [100%]
[root@oracle62 ~]#
```

fix NTP time `(run as root user)`
```bash
touch /etc/ntp.conf
chmod 644 /etc/ntp.conf
touch /var/run/ntpd.pid
chmod 644 /var/run/ntpd.pid
ln -s /usr/sbin/chronyd /usr/sbin/ntpd
```
Now We can Proceed the GUI

![](picture/Grid_Installation_01.png)

![](picture/Grid_Installation_02.png)

![](picture/Grid_Installation_03.png)

![](picture/Grid_Installation_04_0.png)

![](picture/Grid_Installation_04_1.png)

![](picture/Grid_Installation_04_2.png)

![](picture/Grid_Installation_05.png)

![](picture/Grid_Installation_06.png)

![](picture/Grid_Installation_07.png)

![](picture/Grid_Installation_08.png)

![](picture/Grid_Installation_09.png)

![](picture/Grid_Installation_10.png)

![](picture/Grid_Installation_11.png)

![](picture/Grid_Installation_12.png)

![](picture/Grid_Installation_13.png)

![](picture/Grid_Installation_14.png)

![](picture/Grid_Installation_15.png)

![](picture/Grid_Installation_16.png)

![](picture/Grid_Installation_17.png)

![](picture/Grid_Installation_18.png)


### Check Cluster Status 
login as `root` user
```bash
source /home/grid/.grid_env
crsctl stat res -t
ps -ef | grep pmon
su - grid
asmcmd -v
```
expected output
```log
[root@oracle61 ~]# crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.LISTENER.lsnr
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.chad
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.net1.network
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.ons
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.proxy_advm
               OFFLINE OFFLINE      oracle61                 STABLE
               OFFLINE OFFLINE      oracle62                 STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.CRS1.dg(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       oracle62                 STABLE
ora.LISTENER_SCAN2.lsnr
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.LISTENER_SCAN3.lsnr
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.asm(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 Started,STABLE
      2        ONLINE  ONLINE       oracle62                 Started,STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.asmnet1.asmnetwork(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.cvu
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.oracle61.vip
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.oracle62.vip
      1        ONLINE  ONLINE       oracle62                 STABLE
ora.qosmserver
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       oracle62                 STABLE
ora.scan2.vip
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.scan3.vip
      1        ONLINE  ONLINE       oracle61                 STABLE
--------------------------------------------------------------------------------
[root@oracle61 ~]# ps -ef | grep pmon
grid      353155       1  0 19:53 ?        00:00:08 asm_pmon_+ASM1
root      502183  263297  0 21:27 pts/0    00:00:00 grep --color=auto pmon
[root@oracle61 ~]# su - grid
[grid@oracle61 ~]$ asmcmd -v
asmcmd version 19.0.0.0.0
[grid@oracle61 ~]$
```

## Database Installation

### Install DB Software Only 

download and import zip file ([LINUX.X64_193000_grid_home.zip](https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html)) to node1

Login as `oracle`:

```bash
unzip -q /tmp/LINUX.X64_193000_db_home.zip -d $ORACLE_HOME
$ORACLE_HOME/OPatch/opatch version
mv $ORACLE_HOME/OPatch/ $ORACLE_HOME/OPatch_bkp
unzip -q /tmp/p6880880_190000_Linux-x86-64.zip -d $ORACLE_HOME
$ORACLE_HOME/OPatch/opatch version
```

expected outcome

```log
[oracle@oracle61 ~]$ unzip -q /tmp/LINUX.X64_193000_db_home.zip -d $ORACLE_HOME
[oracle@oracle61 ~]$ $ORACLE_HOME/OPatch/opatch version
OPatch Version: 12.2.0.1.17

OPatch succeeded.
[oracle@oracle61 ~]$ mv $ORACLE_HOME/OPatch/ $ORACLE_HOME/OPatch_bkp
[oracle@oracle61 ~]$ unzip -q /tmp/p6880880_190000_Linux-x86-64.zip -d $ORACLE_HOME
[oracle@oracle61 ~]$ $ORACLE_HOME/OPatch/opatch version
OPatch Version: 12.2.0.1.49

OPatch succeeded.
[oracle@oracle61 ~]$
```

Fix oracl 12c DB INS-06006
```bash
$ORACLE_HOME/deinstall/sshUserSetup.sh -user oracle -hosts "oracle61 oracle62" -advanced -exverify -confirm -noPromptPassphrase
ssh oracle62.p2ok.site
ssh oracle62.p2ok.site
```

begin installation
```bash
cd $ORACLE_HOME
./runInstaller
```
expected output

```log
[oracle@oracle61 ~]$ cd $ORACLE_HOME
[oracle@oracle61 db_home]$ ./runInstaller
Launching Oracle Database Setup Wizard...
```

![](picture/Database_Installation_01.png)

![](picture/Database_Installation_02.png)

![](picture/Database_Installation_03.png)

![](picture/Database_Installation_04.png)

![](picture/Database_Installation_05.png)

![](picture/Database_Installation_06.png)

![](picture/Database_Installation_07.png)

![](picture/Database_Installation_08.png)

![](picture/Database_Installation_09.png)

![](picture/Database_Installation_10.png)

![](picture/Database_Installation_11.png)

click close 

![](attachments/Pasted%20image%2020220606133731.png)

### Create Data,FRA
Node 1:
```bash
source /home/grid/.grid_env
export ORACLE_BASE=/tmp
lsblk
asmcmd -V
asmcmd afd_lsdsk
asmcmd afd_label FRA1 /dev/sde1
asmcmd afd_label DATA1 /dev/sdd1
asmcmd afd_lsdsk
ls -l /dev/oracleafd/disks
```
```log
[root@oracle61 ~]# source /home/grid/.grid_env
[root@oracle61 ~]# export ORACLE_BASE=/tmp
[root@oracle61 ~]# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   128G  0 disk
├─sda1        8:1    0   600M  0 part /boot/efi
├─sda2        8:2    0     1G  0 part /boot
└─sda3        8:3    0 126.4G  0 part
  ├─ol-root 252:0    0    60G  0 lvm  /
  ├─ol-swap 252:1    0  12.8G  0 lvm  [SWAP]
  └─ol-home 252:3    0  53.6G  0 lvm  /home
sdb           8:16   0    64G  0 disk
└─sdb1        8:17   0    64G  0 part
  └─db-u01  252:2    0    64G  0 lvm  /u01
sdc           8:32   0    64G  0 disk
└─sdc1        8:33   0    64G  0 part
sdd           8:48   0    64G  0 disk
└─sdd1        8:49   0    64G  0 part
sde           8:64   0    64G  0 disk
└─sde1        8:65   0    64G  0 part
[root@oracle61 ~]# asmcmd -V
asmcmd version 19.0.0.0.0
[root@oracle61 ~]# asmcmd afd_lsdsk
--------------------------------------------------------------------------------
Label                     Filtering   Path
================================================================================
CRS11                      DISABLED   /dev/sdc1
[root@oracle61 ~]# asmcmd afd_label FRA1 /dev/sde1
[root@oracle61 ~]# asmcmd afd_label DATA1 /dev/sdd1
[root@oracle61 ~]# asmcmd afd_lsdsk
--------------------------------------------------------------------------------
Label                     Filtering   Path
================================================================================
CRS11                      DISABLED   /dev/sdc1
DATA1                      DISABLED   /dev/sdd1
FRA1                       DISABLED   /dev/sde1
[root@oracle61 ~]# ls -l /dev/oracleafd/disks/
total 12
-rw-rw-r--. 1 grid oinstall 10 Mar 26 19:51 CRS11
-rw-rw-r--. 1 grid oinstall 10 Mar 26 22:44 DATA1
-rw-rw-r--. 1 grid oinstall 10 Mar 26 22:43 FRA1
[root@oracle61 ~]#

```

Node 2:

```log
[root@oracle62 ~]# source /home/grid/.grid_env
[root@oracle62 ~]# export ORACLE_BASE=/tmp
[root@oracle62 ~]# asmcmd afd_lsdsk
--------------------------------------------------------------------------------
Label                     Filtering   Path
================================================================================
CRS11                      DISABLED   /dev/sdc1
DATA1                      DISABLED   /dev/sdd1
FRA1                       DISABLED   /dev/sde1
[root@oracle62 ~]#

```

Refresh AFD Disks (if not detected)

```bash
asmcmd afd_scan
asmcmd afd_lsdsk
```

login as `grid` user

```bash
asmca
```

Right-click Disk Groups → Select Create

![](picture/ASM_01.png)

click `Create`

![](picture/ASM_02.png)

click `OK`

![](picture/ASM_03.png)

then got 2 disk groups

![](picture/ASM_04.png)

go again for `FRA` disk

![](picture/ASM_05.png)

![](picture/ASM_06.png)

then close

```log
[grid@oracle61 ~]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.LISTENER.lsnr
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.chad
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.net1.network
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.ons
               ONLINE  ONLINE       oracle61                 STABLE
               ONLINE  ONLINE       oracle62                 STABLE
ora.proxy_advm
               OFFLINE OFFLINE      oracle61                 STABLE
               OFFLINE OFFLINE      oracle62                 STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.CRS1.dg(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.DATA.dg(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        ONLINE  OFFLINE                               STABLE
ora.FRA.dg(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        ONLINE  OFFLINE                               STABLE
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       oracle62                 STABLE
ora.LISTENER_SCAN2.lsnr
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.LISTENER_SCAN3.lsnr
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.asm(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 Started,STABLE
      2        ONLINE  ONLINE       oracle62                 Started,STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.asmnet1.asmnetwork(ora.asmgroup)
      1        ONLINE  ONLINE       oracle61                 STABLE
      2        ONLINE  ONLINE       oracle62                 STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.cvu
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.oracle61.vip
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.oracle62.vip
      1        ONLINE  ONLINE       oracle62                 STABLE
ora.qosmserver
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       oracle62                 STABLE
ora.scan2.vip
      1        ONLINE  ONLINE       oracle61                 STABLE
ora.scan3.vip
      1        ONLINE  ONLINE       oracle61                 STABLE
--------------------------------------------------------------------------------
[grid@oracle61 ~]$
```

### Create DB

Login as `oracle`:

```bash 
cd $ORACLE_HOME
dbca
```

![](picture/dbca_01.png)

![](picture/dbca_02.png)

![](picture/dbca_03.png)

![](picture/dbca_04.png)

![](picture/dbca_05.png)

![](picture/dbca_06.png)

![](picture/dbca_07.png)

![](picture/dbca_08.png)

![](picture/dbca_09.png)

![](picture/dbca_10.png)

![](picture/dbca_11.png)

![](picture/dbca_12.png)

![](picture/dbca_13.png)

![](picture/dbca_14.png)

![](picture/dbca_15.png)

![](picture/dbca_16.png)


## Patch DB 
### Node 1

#### Check
Run below command using oracle user 

```bash
cat /tmp/patch_db.txt
$ORACLE_HOME/OPatch/opatch prereq CheckSystemSpace -phBaseFile /tmp/patch_db.txt
vi dbversion.sh
chmod +x dbversion.sh
cat dbversion.sh
./dbversion.sh
```
```log
[oracle@oracle61 ~]$ cat /tmp/patch_db.txt
/home/oracle/38632161/

[oracle@oracle61 ~]$ $ORACLE_HOME/OPatch/opatch prereq CheckSystemSpace -phBaseFile /tmp/patch_db.txt
Oracle Interim Patch Installer version 12.2.0.1.49
Copyright (c) 2026, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/19c/oracle/ora_base/db_home
Central Inventory : /u01/19c/grid/oraInventory
   from           : /u01/19c/oracle/ora_base/db_home/oraInst.loc
OPatch version    : 12.2.0.1.49
OUI version       : 12.2.0.7.0
Log file location : /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatch/opatch2026-03-30_22-54-56PM_1.log

Invoking prereq "checksystemspace"

Prereq "checkSystemSpace" passed.

OPatch succeeded.
[oracle@oracle61 ~]$ vi dbversion.sh
[oracle@oracle61 ~]$ cat dbversion.sh
sqlplus -s / as sysdba <<SQL
set pages 100 lines 500
col comp_name for a40
col version for a15
col version_ful for a15
col status for a15
--col con_id for a10
select comp_name, version, version_full,status,con_id from cdb_registry order by con_id;
SQL
[oracle@oracle61 ~]$ chmod +x dbversion.sh
[oracle@oracle61 ~]$ ./dbversion.sh

COMP_NAME                                VERSION         VERSION_FULL                   STATUS              CON_ID
---------------------------------------- --------------- ------------------------------ --------------- ----------
Oracle Database Catalog Views            19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Database Vault                    19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Real Application Clusters         19.0.0.0.0      19.3.0.0.0                     VALID                    1
JServer JAVA Virtual Machine             19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle XDK                               19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Database Java Packages            19.0.0.0.0      19.3.0.0.0                     VALID                    1
OLAP Analytic Workspace                  19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle XML Database                      19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Workspace Manager                 19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Text                              19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Multimedia                        19.0.0.0.0      19.3.0.0.0                     VALID                    1
Spatial                                  19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle OLAP API                          19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Label Security                    19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Database Packages and Types       19.0.0.0.0      19.3.0.0.0                     VALID                    1

15 rows selected.

[oracle@oracle61 ~]$
```

#### Analyze

login as `root`

```bash
source /home/oracle/.db19_env
cd  $ORACLE_HOME
$ORACLE_HOME/OPatch/opatchauto apply /home/oracle/38632161/ -oh $ORACLE_HOME -analyze
```

```log
[root@oracle61 ~]# source /home/oracle/.db19_env
[root@oracle61 ~]# cd  $ORACLE_HOME
[root@oracle61 db_home]# $ORACLE_HOME/OPatch/opatchauto apply /home/oracle/38632161/ -oh $ORACLE_HOME -analyze

OPatchauto session is initiated at Mon Mar 30 22:57:39 2026

System initialization log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchautodb/systemconfig2026-03-30_10-57-55PM.log.

Session log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/opatchauto2026-03-30_10-59-09PM.log
The id for this session is 269Z

Executing OPatch prereq operations to verify patch applicability on home /u01/19c/oracle/ora_base/db_home
Patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home


Executing patch validation checks on home /u01/19c/oracle/ora_base/db_home
Patch validation checks successfully completed on home /u01/19c/oracle/ora_base/db_home


Verifying SQL patch applicability on home /u01/19c/oracle/ora_base/db_home
SQL patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home

OPatchAuto successful.

--------------------------------Summary--------------------------------

Analysis for applying patches has completed successfully:

Host:oracle61
RAC Home:/u01/19c/oracle/ora_base/db_home
Version:19.0.0.0.0


==Following patches were SUCCESSFULLY analyzed to be applied:

Patch: /home/oracle/38632161
Log: /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/core/opatch/opatch2026-03-30_23-00-29PM_1.log



OPatchauto session completed at Mon Mar 30 23:03:51 2026
Time taken to complete the session 5 minutes, 57 seconds
[root@oracle61 db_home]#
```

#### Apply

```bash
source /home/oracle/.db19_env
$ORACLE_HOME/OPatch/opatchauto apply /home/oracle/38632161/ -oh $ORACLE_HOME
```

```log
[root@oracle61 db_home]# $ORACLE_HOME/OPatch/opatchauto apply /home/oracle/38632161/ -oh $ORACLE_HOME

OPatchauto session is initiated at Mon Mar 30 23:06:32 2026

System initialization log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchautodb/systemconfig2026-03-30_11-06-47PM.log.

Session log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/opatchauto2026-03-30_11-07-31PM.log
The id for this session is 9NN5

Executing OPatch prereq operations to verify patch applicability on home /u01/19c/oracle/ora_base/db_home
Patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home


Executing patch validation checks on home /u01/19c/oracle/ora_base/db_home
Patch validation checks successfully completed on home /u01/19c/oracle/ora_base/db_home


Verifying SQL patch applicability on home /u01/19c/oracle/ora_base/db_home
"[/bin/sh -c 'cd /u01/19c/oracle/ora_base/db_home; ORACLE_HOME=/u01/19c/oracle/ora_base/db_home ORACLE_SID=PROD1 /u01/19c/oracle/ora_base/db_home/OPatch/datapatch -prereq -verbose']" command failed with errors. Please refer to logs for more details. SQL changes, if any, can be analyzed by manually retrying the same command.

SQL patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home


Preparing to bring down database service on home /u01/19c/oracle/ora_base/db_home
Successfully prepared home /u01/19c/oracle/ora_base/db_home to bring down database service


Bringing down database service on home /u01/19c/oracle/ora_base/db_home
Following database(s) and/or service(s) are stopped and will be restarted later during the session: prod
Database service successfully brought down on home /u01/19c/oracle/ora_base/db_home


Performing prepatch operation on home /u01/19c/oracle/ora_base/db_home
Prepatch operation completed successfully on home /u01/19c/oracle/ora_base/db_home


Start applying binary patch on home /u01/19c/oracle/ora_base/db_home
Binary patch applied successfully on home /u01/19c/oracle/ora_base/db_home


Running rootadd_rdbms.sh on home /u01/19c/oracle/ora_base/db_home
Successfully executed rootadd_rdbms.sh on home /u01/19c/oracle/ora_base/db_home


Performing postpatch operation on home /u01/19c/oracle/ora_base/db_home
Postpatch operation completed successfully on home /u01/19c/oracle/ora_base/db_home


Starting database service on home /u01/19c/oracle/ora_base/db_home
Database service successfully started on home /u01/19c/oracle/ora_base/db_home


Preparing home /u01/19c/oracle/ora_base/db_home after database service restarted
No step execution required.........


Trying to apply SQL patch on home /u01/19c/oracle/ora_base/db_home
SQL patch applied successfully on home /u01/19c/oracle/ora_base/db_home

OPatchAuto successful.

--------------------------------Summary--------------------------------

Patching is completed successfully. Please find the summary as follows:

Host:oracle61
RAC Home:/u01/19c/oracle/ora_base/db_home
Version:19.0.0.0.0
Summary:

==Following patches were SUCCESSFULLY applied:

Patch: /home/oracle/38632161
Log: /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/core/opatch/opatch2026-03-30_23-13-23PM_1.log



OPatchauto session completed at Tue Mar 31 00:57:55 2026
Time taken to complete the session 111 minutes, 8 seconds
[root@oracle61 db_home]#
```

### Node 2

#### Check
Run below command using `oracle` user 

```bash
vi /tmp/patch_db.txt
cat /tmp/patch_db.txt
$ORACLE_HOME/OPatch/opatch prereq CheckSystemSpace -phBaseFile /tmp/patch_db.txt
vi dbversion.sh
chmod +x dbversion.sh
cat dbversion.sh
./dbversion.sh
```
```log
[oracle@oracle62 ~]$ vi /tmp/patch_db.txt
[oracle@oracle62 ~]$ cat /tmp/patch_db.txt
/home/oracle/38632161/

[oracle@oracle62 ~]$ $ORACLE_HOME/OPatch/opatch prereq CheckSystemSpace -phBaseFile /tmp/patch_db.txt
Oracle Interim Patch Installer version 12.2.0.1.49
Copyright (c) 2026, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/19c/oracle/ora_base/db_home
Central Inventory : /u01/19c/grid/oraInventory
   from           : /u01/19c/oracle/ora_base/db_home/oraInst.loc
OPatch version    : 12.2.0.1.49
OUI version       : 12.2.0.7.0
Log file location : /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatch/opatch2026-03-30_21-52-34PM_1.log

Invoking prereq "checksystemspace"

Prereq "checkSystemSpace" passed.

OPatch succeeded.
[oracle@oracle62 ~]$ vi dbversion.sh
[oracle@oracle62 ~]$ chmod +x dbversion.sh
[oracle@oracle62 ~]$ cat dbversion.sh
sqlplus -s / as sysdba <<SQL
set pages 100 lines 500
col comp_name for a40
col version for a15
col version_ful for a15
col status for a15
--col con_id for a10
select comp_name, version, version_full,status,con_id from cdb_registry order by con_id;
SQL
[oracle@oracle62 ~]$ ./dbversion.sh

COMP_NAME                                VERSION         VERSION_FULL                   STATUS              CON_ID
---------------------------------------- --------------- ------------------------------ --------------- ----------
Oracle Database Catalog Views            19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Database Vault                    19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Real Application Clusters         19.0.0.0.0      19.3.0.0.0                     VALID                    1
JServer JAVA Virtual Machine             19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle XDK                               19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Database Java Packages            19.0.0.0.0      19.3.0.0.0                     VALID                    1
OLAP Analytic Workspace                  19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle XML Database                      19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Workspace Manager                 19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Text                              19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Multimedia                        19.0.0.0.0      19.3.0.0.0                     VALID                    1
Spatial                                  19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle OLAP API                          19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Label Security                    19.0.0.0.0      19.3.0.0.0                     VALID                    1
Oracle Database Packages and Types       19.0.0.0.0      19.3.0.0.0                     VALID                    1

15 rows selected.

[oracle@oracle62 ~]$
```

#### Analyze

login as `root`

```bash
source /home/oracle/.db19_env
cd $ORACLE_HOME
$ORACLE_HOME/OPatch/opatchauto apply /tmp/38940525/ -oh $ORACLE_HOME -analyze
```

```log
[root@oracle62 ~]# cd $ORACLE_HOME
[root@oracle62 db_home]# $ORACLE_HOME/OPatch/opatchauto apply /home/oracle/38632161/ -oh $ORACLE_HOME -analyze

OPatchauto session is initiated at Mon Mar 30 21:10:23 2026

System initialization log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchautodb/systemconfig2026-03-30_09-11-41PM.log.

Session log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/opatchauto2026-03-30_09-13-04PM.log
The id for this session is HJYY

Executing OPatch prereq operations to verify patch applicability on home /u01/19c/oracle/ora_base/db_home
Patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home


Executing patch validation checks on home /u01/19c/oracle/ora_base/db_home
Patch validation checks successfully completed on home /u01/19c/oracle/ora_base/db_home


Verifying SQL patch applicability on home /u01/19c/oracle/ora_base/db_home
SQL patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home

OPatchAuto successful.

--------------------------------Summary--------------------------------

Analysis for applying patches has completed successfully:

Host:oracle62
RAC Home:/u01/19c/oracle/ora_base/db_home
Version:19.0.0.0.0


==Following patches were SUCCESSFULLY analyzed to be applied:

Patch: /home/oracle/38632161
Log: /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/core/opatch/opatch2026-03-30_21-15-33PM_1.log



OPatchauto session completed at Mon Mar 30 21:20:06 2026
Time taken to complete the session 8 minutes, 29 seconds
[root@oracle62 db_home]#
```

#### Apply

```bash
source /home/oracle/.db19_env
$ORACLE_HOME/OPatch/opatchauto apply /home/oracle/38632161/ -oh $ORACLE_HOME
```

```log
[root@oracle62 db_home]# source /home/oracle/.db19_env
[root@oracle62 db_home]# $ORACLE_HOME/OPatch/opatchauto apply /home/oracle/38632161/ -oh $ORACLE_HOME

OPatchauto session is initiated at Mon Mar 30 21:54:35 2026

System initialization log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchautodb/systemconfig2026-03-30_09-54-49PM.log.

Session log file is /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/opatchauto2026-03-30_09-55-42PM.log
The id for this session is 57JU

Executing OPatch prereq operations to verify patch applicability on home /u01/19c/oracle/ora_base/db_home
Patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home


Executing patch validation checks on home /u01/19c/oracle/ora_base/db_home
Patch validation checks successfully completed on home /u01/19c/oracle/ora_base/db_home


Verifying SQL patch applicability on home /u01/19c/oracle/ora_base/db_home
SQL patch applicability verified successfully on home /u01/19c/oracle/ora_base/db_home


Preparing to bring down database service on home /u01/19c/oracle/ora_base/db_home
Successfully prepared home /u01/19c/oracle/ora_base/db_home to bring down database service


Bringing down database service on home /u01/19c/oracle/ora_base/db_home
Following database(s) and/or service(s) are stopped and will be restarted later during the session: prod
Database service successfully brought down on home /u01/19c/oracle/ora_base/db_home


Performing prepatch operation on home /u01/19c/oracle/ora_base/db_home
Prepatch operation completed successfully on home /u01/19c/oracle/ora_base/db_home


Start applying binary patch on home /u01/19c/oracle/ora_base/db_home
Binary patch applied successfully on home /u01/19c/oracle/ora_base/db_home


Running rootadd_rdbms.sh on home /u01/19c/oracle/ora_base/db_home
Successfully executed rootadd_rdbms.sh on home /u01/19c/oracle/ora_base/db_home


Performing postpatch operation on home /u01/19c/oracle/ora_base/db_home
Postpatch operation completed successfully on home /u01/19c/oracle/ora_base/db_home


Starting database service on home /u01/19c/oracle/ora_base/db_home
Database service successfully started on home /u01/19c/oracle/ora_base/db_home


Preparing home /u01/19c/oracle/ora_base/db_home after database service restarted
No step execution required.........


Trying to apply SQL patch on home /u01/19c/oracle/ora_base/db_home
No SQL patch operations are required on local node for this home

OPatchAuto successful.

--------------------------------Summary--------------------------------

Patching is completed successfully. Please find the summary as follows:

Host:oracle62
RAC Home:/u01/19c/oracle/ora_base/db_home
Version:19.0.0.0.0
Summary:

==Following patches were SUCCESSFULLY applied:

Patch: /home/oracle/38632161
Log: /u01/19c/oracle/ora_base/db_home/cfgtoollogs/opatchauto/core/opatch/opatch2026-03-30_22-05-42PM_1.log



OPatchauto session completed at Mon Mar 30 22:29:25 2026
Time taken to complete the session 34 minutes, 37 seconds
[root@oracle62 db_home]#
```

### Data Patch

login as `oracle` user
```bash
$ORACLE_HOME/OPatch/datapatch -verbose
```
```log
[oracle@oracle61 ~]$ $ORACLE_HOME/OPatch/datapatch -verbose
SQL Patching tool version 19.30.0.0.0 Production on Tue Mar 31 00:59:22 2026
Copyright (c) 2012, 2026, Oracle.  All rights reserved.

Log file for this invocation: /u01/19c/oracle/ora_base/cfgtoollogs/sqlpatch/sqlpatch_170951_2026_03_31_00_59_22/sqlpatch_invocation.log

Connecting to database...OK
Gathering database info...done

Note:  Datapatch will only apply or rollback SQL fixes for PDBs
       that are in an open state, no patches will be applied to closed PDBs.
       Please refer to Note: Datapatch: Database 12c Post Patch SQL Automation
       (Doc ID 1585822.1)

Warning: PDB PRODPDB1 is in mode MOUNTED and will be skipped.
Bootstrapping registry and package to current versions...done
Determining current state...done

Current state of interim SQL patches:
  No interim patches found

Current state of release update SQL patches:
  Binary registry:
    19.30.0.0.0 Release_Update 260126024251: Installed
  PDB CDB$ROOT:
    Applied 19.30.0.0.0 Release_Update 260126024251 successfully on 31-MAR-26 12.27.55.166855 AM
  PDB PDB$SEED:
    Applied 19.30.0.0.0 Release_Update 260126024251 successfully on 31-MAR-26 12.55.39.580538 AM

Adding patches to installation queue and performing prereq checks...done
Installation queue:
  For the following PDBs: CDB$ROOT PDB$SEED
    No interim patches need to be rolled back
    No release update patches need to be installed
    No interim patches need to be applied

SQL Patching tool complete on Tue Mar 31 01:01:28 2026
[oracle@oracle61 ~]$
```

### Check Version 
```bash
cat ./dbversion.sh
./dbversion.sh
```

```log
[oracle@oracle61 ~]$ cat ./dbversion.sh
sqlplus -s / as sysdba <<SQL
set pages 100 lines 500
col comp_name for a40
col version for a15
col version_ful for a15
col status for a15
--col con_id for a10
select comp_name, version, version_full,status,con_id from cdb_registry order by con_id;
SQL
```
run check
```log
[oracle@oracle61 ~]$ ./dbversion.sh

COMP_NAME                                VERSION         VERSION_FULL                   STATUS              CON_ID
---------------------------------------- --------------- ------------------------------ --------------- ----------
Oracle Database Catalog Views            19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Database Vault                    19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Real Application Clusters         19.0.0.0.0      19.30.0.0.0                    VALID                    1
JServer JAVA Virtual Machine             19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle XDK                               19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Database Java Packages            19.0.0.0.0      19.30.0.0.0                    VALID                    1
OLAP Analytic Workspace                  19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle XML Database                      19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Workspace Manager                 19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Text                              19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Multimedia                        19.0.0.0.0      19.30.0.0.0                    VALID                    1
Spatial                                  19.0.0.0.0      19.30.0.0.0                    LOADING                  1
Oracle OLAP API                          19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Label Security                    19.0.0.0.0      19.30.0.0.0                    VALID                    1
Oracle Database Packages and Types       19.0.0.0.0      19.30.0.0.0                    VALID                    1

15 rows selected.

[oracle@oracle61 ~]$
```

## Manage Cluster Services

Oracle RAC provides two main tools:

- `crsctl` → Manage **Clusterware** (CRS, CSS, EVM)
- `srvctl` → Manage **Cluster Resources** (DB, instances, listeners, services)

---

### Key Difference

| Tool     | Scope            | Used For |
|----------|-----------------|----------|
| crsctl   | Clusterware     | Start/stop CRS stack, check cluster health |
| srvctl   | Cluster         | Manage DB, ASM, listeners, services |

---

### Basic Usage

#### crsctl (Clusterware)

Check cluster status

```bash
crsctl check cluster -all
```
```log
[root@oracle61 ~]# crsctl check cluster -all
**************************************************************
oracle61:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
oracle62:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
[root@oracle61 ~]#
```

Status of resources

```bash
crsctl stat res -t
```

Stop cluster on all nodes

```bash
crsctl stop cluster -all
```
Start cluster on all nodes
```bash
crsctl start cluster -all
```
Cluster Control (Specific Node) <br>
Stop cluster services on a specific node:
```bash
crsctl stop cluster -n oracle61
```
Start cluster services on a specific node:
```bash
crsctl start cluster -n oracle62
```
CRS (High Availability Services)

Enable CRS auto-start:
```bash
crsctl enable crs
```
Check CRS status:
```bash
crsctl check crs
```
Stop CRS on local node:
```bash
crsctl stop crs
```
Force stop CRS if required:
```bash
crsctl stop crs -f
```
Help Commands

Display available options:
```bash
crsctl -h
crsctl check -h
```
Instance Management<br>
Stop specific instance:
```bash
srvctl stop instance -d PROD -i PROD2
```

Start specific instance:
```bash
srvctl start instance -d PROD -i PROD2
```
Database Status and Configuration<br>
Check database status:
```bash
srvctl status database -d prod
```
Display database configuration:
```bash
srvctl config database -d prod
```
Search configuration options:
```bash
srvctl modify database -h | grep -i pfile
```
Service Management<br>
Create service:
```bash
srvctl add service -d prod1 -pdb prodpdb1 -service sales -preferred PROD1,PROD2
```
Check service status:
```bash
srvctl status service -d prod
```
Start service:
```bash
srvctl start service -d prod -s sales
```
Stop service:
```bash
srvctl stop service -d prod -s sales
```
Remove service:
```bash
srvctl remove service -d prod -s sales
```

## SQL commands

Here are compact SQL commands to verify if a database is RAC<br>
Check if RAC Enabled<br>
login as `oracle` user

```bash
su - oracle
sqlplus / as sysdba
```
```log
[root@oracle61 ~]# su - oracle
[oracle@oracle61 ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Mon Mar 30 22:42:16 2026
Version 19.15.0.0.0

Copyright (c) 1982, 2022, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.15.0.0.0

SQL>
```

SQL command

```sql
SELECT value FROM v$parameter WHERE name = 'cluster_database';
```
```log
SQL> SELECT value FROM v$parameter WHERE name = 'cluster_database';

VALUE
--------------------------------------------------------------------------------
TRUE
```
Check Number of Instances
```sql
SELECT inst_id, instance_name, host_name FROM gv$instance;
```
```log
SQL> SELECT inst_id, instance_name, host_name FROM gv$instance;

INST_ID INSTANCE_NAME
---------- ----------------
HOST_NAME
----------------------------------------------------------------
         1 PROD1
oracle61.p2ok.site

         2 PROD2
oracle62.p2ok.site
```
Check Cluster Database View
```sql
SELECT * FROM v$active_instances;
```
```log
SQL> SELECT * FROM v$active_instances;

INST_NUMBER
-----------
INST_NAME
--------------------------------------------------------------------------------
    CON_ID
----------
          1
oracle61.p2ok.site:PROD1
         0

          2
oracle62.p2ok.site:PROD2
         0

INST_NUMBER
-----------
INST_NAME
--------------------------------------------------------------------------------
    CON_ID
----------
```

## References

- [Oracle RAC Guide][rac-guide]
- [Database Installation Guide for Linux][Database-Installation-Guide-for-Linux]
- [Real Application Clusters Installation Guide for Linux and UNIX][Real-Application-Clusters-Installation-Guide-for-Linux-and-UNIX]

---

[rac-guide]: https://github.com/icyb3r-code/DBAdmin/blob/master/Oracle/Contents/Oracle19c_RAC_on_OEL8.5/README.md#oracle-19c-rac
[Database-Installation-Guide-for-Linux]: https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/index.html
[Real-Application-Clusters-Installation-Guide-for-Linux-and-UNIX]: https://docs.oracle.com/en/database/oracle/oracle-database/19/rilin/index.html