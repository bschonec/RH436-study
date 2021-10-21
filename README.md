NOTE:  I have not taken the EX436 exam.  I am bound by non-disclosure agreements and I will not answer any questions related to the exam.  These are notes I've taken to study for the exam.

firewall-cmd --add-service high-availability --permanent
yum -y install pcs # pacemaker 
corosync-quorumtool
pcs cluster setup
pcs quorum status
pcs cluster start --all
pcs cluster enable --all

pcs host auth <nodex>

pcs cluster setup <name> <nodeX> <nodeY>
pcs auth nodec
pcs cluster node add <node>

# search for fencing agents, find the correct one.
yum -y install fence-agents-ipmilan

systemctl enable --now pcsd

man -k fence\_   # shows all fence man pages
man fence_ipmilan

pcs stonith list # Lists installed fence devices
pcs stonith create <name>      <fence agent> 
pcs stonith create fence_nodea fence_ipmilan pcmk_host_list=<node> ip=<ip> username= password= lanplus=1 power_timeout
pcs stonith status


## Options for managing quorum

wait_for_all
auto_tie_breaker
last_man_standing


pcs node standby node1.example.com  # migrate resources off this node and don't accept any new resources
pcs node unstandby node1
pcs node unstandby --all

pcs status cluster
pcs status resources
pcs status nodes
pcs status corosync
pcs status pcsd


pcs cluster stop --all && pcs quorum update wait_for_all=1 last_man_standing=1; pcs cluster start --all



# Chapter 3:  Protecting Data with Fencing 


man -k fence\_    #shows documentation for fencing

pcs stonith confi fence_node1....
pcs stonith update fence_node pcmk_host_list=node2.example.com


# Chapter 4:  Creating and Configuring Resources

/usr/lib/ocf/resource.d   # Providers of various resources
pcs resource list

pcs resource describe Filesystem | grep -i required
pcs resource disable <name>
pcs resource move <name> <node>  # this will add a constraint to prevent <name> from running on <node> forever.
pcs resource clear <resource>   # clears restraint

pcs resource defaults update resource-stickiness=500
pcs constraint location <resource> prefers nodeb.private.example.com=499


pcs constraint list --full
pcs constraint loction delete <id from previous command>

pcs constraint order A then B

pcs constraint location mysql prefers nodea=500 


## Creating and Configurting Resources

RPM packages need to be installed on all nodes that are configured to host the service.
SELinux needs to be configured properly on all nodes as well. (booleans, contexts, etc).

An apache resource group consists of (at least) IP address, Filesystem and Service.

pcs resource create <name> <TYPE> <args>
pcs constraint list

pcs resource move <resource name>   # this prevents the resource from starting on THIS node until the constraint is removed.

pcs resource clear <resource name>


# Chapter 5:  Troubleshooting High Availability Clusters

cat /etc/corosync/corosync.conf   # look for logging section

mailx and a MTA (ie: postfix) needs to be installed to deliver mail.
pcs alert create path=<path> id= options..
pcs resource describe MailTo
pcs alert show

pcs alert create id=mailme path=/var/lib/pacemaker/alert_smtp.sh options email_sender=dontreplay@example.com
pcs alert recient add  mailme value=student@workstaion.com



pcs resource config firstwebserver
pcs resource debug-start firstwebserver --full | less

When changing pacemaker settings, it's necessary to 'pcs cluster stop --all; pcs cluster start --all'

pcs resource failcount show



# Chapter 6: 

# Chapter 7:  Managing Two-node Clusters
pcmk_delay_max=30s  # Delays radomly between 0 and 30s for a fencing race.

 pcs resource defaults update priority=1  # Set any *new* resource's priority to one

voting disk:
yum install pcs corosync-qnetd




# Chapter 8:  Accessing iSCSI Storage

Always use the UUID of the iscsi block device when mounting in /etc/fstab because the path may change.

When using multipathed iSCSI, it's important to log in to both targers.


udevadm info /dev/sda | grep SERIAL
lsblk --fs /dev/sda # after formatting the filesystem.

mpathconf --enable   # creates multipathd.conf file.
/usr/share/doc/device-mapper-multipath/multipath.conf contains example of multipath setup


# Chapter 10:  Configuring LVM in Clusters

HA-LVM = active/passive only ext4 and xfs are acceptable
lvmlockd - multi mount.  Requires cluster aware file system eg: gfs2


Change /etc/lvm/lvm.conf  parameter: 'system_id_source = "uname"'

pcs resource create halvm LVM-activate vgname=vg_access_mode=system_id --group=
pcs resource create xfsfs Filesystem device= directory=/mnt fstype=xfs --group=


lvmlockd needs to be running on all cluster members 

vgcreate --shared name /dev/mapper/shared

lvcreate --activate sy -L 5G -n lvname vgname

pcs resource create sharedlvm LVM-activate vgname=sharedvg lvnameA


# Chapter 11:  Providing Storage with the GFS2 Cluster File System

yum install -y gfs-utils


pcs property set no-quorum-policy=freeze
gfs2_jadd     # journal must be created for each node
