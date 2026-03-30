# ERROR 

## ERROR: The home is not clean

```log
ERROR: The home is not clean. This home cannot be used since there was a failed OPatch execution in this home. Use a different home to proceed.
```

recommedation
```terminal
rm -rf /u01/19c/grid/grid_home/install/.patch*
rm -rf /u01/19c/grid/grid_home/install/patch*
```
or
```terminal
rm -rf $ORACLE_HOME/install/.patch*
rm -rf $ORACLE_HOME/install/patch*
```

## [INS-44000] Passwordless SSH connectivity is not setup

```log
Action - Refer to the logs for more details or contact Oracle Support Services.  
Additional Information:
Summary of node specific errors 
oracle19c02  
    - [INS-06006] Passwordless SSH connectivity not set up between the following node(s): [oracle19c02]. 
```

## PRVG-11550 : Package "cvuqdisk" is missing

recommendation
```terminal
cd $ORACLE_HOME/cv/rpm
rpm -ivh cvuqdisk*.rpm
```

## PRVF-9991 : Owner of device
```log
Access Control List check - This check verifies that the ownership and permissions are correct and consistent for the devices across nodes. Details:  
-  
PRVF-9991 : Owner of device "/dev/sdc1" did not match the expected owner. [Expected = "grid"; Found = "root"] on nodes: [oracle19c01]  - Cause:  Owner of the device listed was different than required owner.  - Action:  Change the owner of the device listed or specify a different device.  
-  
PRVF-9992 : Group of device "/dev/sdc1" did not match the expected group. [Expected = "asmadmin"; Found = "disk"] on nodes: [oracle19c01]  - Cause:  Group of the device listed was different than required group.  - Action:  Change the group of the device listed or specify a different device.
```
recommendation
```terminal
[root@oracle19c01 ~]# oracleasm configure -i
```
```output
Configuring the Oracle ASM system service.

This will configure the on-boot properties of the Oracle ASM system
service.  The following questions will determine whether the service
is started on boot and what permissions it will have.  The current
values will be shown in brackets ('[]').  Hitting <ENTER> without
typing an answer will keep that current value.  Ctrl-C will abort.

Default user to own the ASM disk devices [oracle]: grid
Default group to own the ASM disk devices [asmadmin]: asmadmin
Start Oracle ASM system service on boot (y/n) [y]: y
Scan for Oracle ASM disks when starting the oracleasm service (y/n) [y]: y
Maximum number of ASM disks that can be used on system [2048]:
Enable iofilter if kernel supports it (y/n) [y]: y
Writing Oracle ASM system service configuration: done

Configuration changes only come into effect after the Oracle ASM
system service is restarted.  Please run 'systemctl restart oracleasm'
after making changes.

WARNING: All of your Oracle and ASM instances must be stopped prior
to restarting the oracleasm service.
```
```terminal
[root@oracle19c01 ~]# systemctl restart oracleasm
[root@oracle19c01 ~]# oracleasm configure
```
```log
ORACLEASM_UID=grid
ORACLEASM_GID=asmadmin
ORACLEASM_SCANBOOT=true
ORACLEASM_SCANORDER=""
ORACLEASM_SCANEXCLUDE=""
ORACLEASM_SCAN_DIRECTORIES=""
ORACLEASM_USE_LOGICAL_BLOCK_SIZE="false"
ORACLEASM_CONFIG_MAX_DISKS="2048"
ORACLEASM_ENABLE_IOFILTER="true"
```

## PRVG-1101 : SCAN name

```log

DNS/NIS name service - This test verifies that the Name Service lookups for the Distributed Name Server (DNS) and the Network Information Service (NIS) match for the SCAN name entries.  Error: 
 - 
PRVG-1101 : SCAN name "oracle19c-cluster-scan" failed to resolve  - Cause:  An attempt to resolve specified SCAN name to a list of IP addresses failed because SCAN could not be resolved in DNS or GNS using ''nslookup''.  - Action:  Check whether the specified SCAN name is correct. If SCAN name should be resolved in DNS, check the configuration of SCAN name in DNS. If it should be resolved in GNS make sure that GNS resource is online. 
```
recommendation
edit
```

```
Add:
```conf

```
## PRVG-1019 : The NTP configuration file "/etc/ntp.conf" does not exist on nodes

```log
'/etc/ntp.conf' -   Error: 
 - 
PRVG-1019 : The NTP configuration file "/etc/ntp.conf" does not exist on nodes "oracle02,oracle01"  - Cause:  The configuration file specified was not available or was inaccessible on the specified nodes.  - Action:  To use NTP for time synchronization, create this file and set up its configuration as described in your vendor''s NTP document. To use CTSS for time synchronization the NTP configuration should be uninstalled on all nodes of the cluster. Refer to section "Preparing Your Cluster" of the book "Oracle Database 2 Day + Real Application Clusters Guide"
```
action
```bash
touch /etc/ntp.conf
chmod 644 /etc/ntp.conf
```
## PRVG-1019 : The NTP configuration file "/var/run/ntpd.pid" does not exist on nodes

```log
'/var/run/ntpd.pid' -   Error: 
 - 
PRVG-1019 : The NTP configuration file "/var/run/ntpd.pid" does not exist on nodes "oracle02,oracle01"  - Cause:  The configuration file specified was not available or was inaccessible on the specified nodes.  - Action:  To use NTP for time synchronization, create this file and set up its configuration as described in your vendor''s NTP document. To use CTSS for time synchronization the NTP configuration should be uninstalled on all nodes of the cluster. Refer to section "Preparing Your Cluster" of the book "Oracle Database 2 Day + Real Application Clusters Guide".
```
action
```bash
touch /var/run/ntpd.pid
chmod 644 /var/run/ntpd.pid
```

## PRVE-10077 : NOZEROCONF parameter not set

recommendation
edit
```
/etc/sysconfig/network
```
Add:
```conf
NOZEROCONF=yes
```
