---
layout: post
title:  Multiple NFS clients behind NAT, for FreeBSD PF firewall
<!--date:  2021-03-02 14:05 +0800 -->
categories: NFS NAT FreeBSD
---

The NFS server is accessible in the intranet. A NAT runs on a workstation with FreeBSD or Linux, which is a node in the intranet. Multiple NFS clients are in a private LAN behind the NAT. Routed or Bridged network is not preferred in order to keep the clients invisible outside the NAT.

Some of the clients are SPARC machines with Solaris 2.6 / 7 / 8 installed. The applications need the network disks to be mounted to access files as if they are other normal files in the file system. Therefore, neither CIFS (Samba) mount nor NFSv4 can be adopted. The `smb_client` program is also rejected. The only option is NFS mount with version<=3.

A problem is: After NAT, the source port usually >=1024, while NFS server may allow only privileged source ports (port<1024).


## Possible approaches

### Option 'insecure' in /etc/exports
The simplest solution is to turn on the 'insecure' option in /etc/exports of NFS servers if no other concern exists. The option 'insecure' makes the server permit source port >=1024.

For our case, security is a concern. We do not want any non-administrative users to privately mount the NFS server. Thus the _insecure_ option is not in our list.


### NFS proxy, like NFS-ganesha
I never success to use NFS-ganesha proxy FSAL to access NFS outside the workstation for some errors (OS: CentOS 7). To solve it beyonded my capability then, honestly.


### NFS re-export, by user mode nfs server, like unfsd
I never try it as some discussions in [NFS export of unsupported filesystems (e.g. NFS re-export)](https://groups.google.com/forum/#!topic/alt.os.linux/oXW6JjIcqAw) mention lock-down may occurs.


### NAT + static-port, in FreeBSD PF
Add 'static-port' for _nat_ in pf.conf, the configuration file of FreeBSD PF program. The option keeps source port number unchanged.

/etc/pf.conf:

	ext_if="em0"
	int_if="em1"
	nat  on $ext_if inet  from $int_if:network to any -> ($ext_if) static-port

If there is only single client behind NAT, this is a good solution. Some virtual machine software programs seem to solve VM's NFS-mount problem by this approach.

However, the same approach may not always work for multiple clients. In my first try with 4 NFS clients (Solaris 8), the first 2 clients lose server responses after the 3rd and the 4th client mount the NFS. A very bad instance sometimes occurs: all clients use the same source port number.


### NAT + privileged ports, in FreeBSD PF
Add an extra _nat_ translating only privileged ports before the ordinary major _nat_ statement:

	 lan_nfs_cli = "{ 192.168.1.10, 192.168.1.11, 192.168.1.12, 192.168.1.13 } port 111:1023"
	 mainnas="192.168.2.11"
	 nat on $ext_if inet  from $lan_nfs_cli    to $mainnas -> ($ext_if) port 111:1023
	 nat on $ext_if inet  from $int_if:network to any      -> ($ext_if)

The available source ports are much less than the _insecure_ option of NFS server. Thus this approach is suitable when the count of clients behind NAT is not large.


## An example for NAT + privileged ports

Network hierarchy for the experiment:

* IP=192.168.2.11, NFS server
* IP=192.168.2.12. NAT
  - IP=192.168.1.10--13. NFS clients 1--4


### construct the hierarchy by QEMU

QEMU-4.1.0 was adopted to test the idea.

* Host: FreeBSD 12.1
* Guest 1: NFS server, Debian
* Guest 2: NAT-machine, FreeBSD 12.1-RELEASE.
* Guest 3-6: NFS clients 1-4, by FreeBSD bootonly installation CD (12.1-RELEASE-amd64) or Solaris 8

NAT-machine(Guest 2) invocation:

	qemu-system-x86_64 \
	       ...
	    -nic tap,ifname=tap2 \
	    -nic socket,listen=:20001 \
	    -nic socket,listen=:20002 \
	    -nic socket,listen=:20003 \
	    -nic socket,listen=:20004 \
	      ...

`-nic socket` appears 4 times with different ports for NFS clients 1-4. The action creates 4 network interfaces, em1-4, which then become members of a virtual bridge in NAT-machine.

Note: the `pfctl` program of NAT-machine FreeBSD may need to be built with -O0 option to work in early version of QEMU, including QEMU-4.1.0. Or 192.168.x.x will become 64.168.x.x and thus not work at all. To build it elsewhere and transfer binary executable file into NAT-machine is ok.


NFS client 1 invocation:

	qemu-system-x86_64 \
	      ...
	    -nic socket,connect=127.0.0.1:20001 \
	    -cdrom ...
	      ...

NFS clients 2-4 are similar except port number of `connect=...`. For qemu-sparc, `model=lance` can be inserted as an argument if necessary.

Updates: although `-nic socket` was used for experiments, simliar network hierarchy constructed by libvirt bridge also works fine.


### NAT-machine configurations

The following are related configurations.

/boot/loader.conf, if you use bridge and tap.

	if_bridge_load="YES"
	if_tap_load="YES"

/etc/rc.conf:

	defaultrouter="192.168.2.1"
	pf_enable="YES"
	gateway_enable="YES"
	
	cloned_interfaces="bridge0"
	ifconfig_bridge0="inet 192.168.1.1/24"
	autobridge_interfaces="bridge0"
	autobridge_bridge0="tap* em1 em2 em3 em4"
	ifconfig_em1="inet 192.168.1.31/24"
	ifconfig_em2="inet 192.168.1.32/24"
	ifconfig_em3="inet 192.168.1.33/24"
	ifconfig_em4="inet 192.168.1.34/24"
	  ...

The IP addresses for em1-4 are for dhcpd. Not a good setting because dhcpd blames on em1-4. But eventually it works. The gateway of NFS clients 1-4 (192.168.1.10--13) is bridge0 (192.168.1.1). 

/etc/pf.conf:

	set skip on lo0
	set block-policy return
	scrub in all
	
	 ext_if="em0"
	 int_if="bridge0"
	 lan_nfs_cli = "{ 192.168.1.10, 192.168.1.11, 192.168.1.12, 192.168.1.13 } port 111:1023"
	
	 mainnas="192.168.2.11"
	
	 nat on $ext_if inet  from $lan_nfs_cli    to $mainnas -> ($ext_if) port 111:1023
	 nat on $ext_if inet  from $int_if:network to any      -> ($ext_if)
	
	pass in all
	pass out all


### Check the result of NAT settings

All the results below were gotten when the port range was set to 111:1024 which is wrong. The correct range is 111:1023. 

Check the rules of PF:

	root@fbsd_pf:/mnt/t01-nfs # pfctl -sn; pfctl -sr
	nat on em0 inet from 192.168.1.10 port 111:1024 to 192.168.2.11 -> (em0) port 111:1024 round-robin
	nat on em0 inet from 192.168.1.11 port 111:1024 to 192.168.2.11 -> (em0) port 111:1024 round-robin
	nat on em0 inet from 192.168.1.12 port 111:1024 to 192.168.2.11 -> (em0) port 111:1024 round-robin
	nat on em0 inet from 192.168.1.13 port 111:1024 to 192.168.2.11 -> (em0) port 111:1024 round-robin
	nat on em0 inet from 192.168.1.0/24 to any -> (em0) round-robin
	nat on em0 inet from 192.168.1.0/24 to any -> (em0) round-robin
	scrub in all fragment reassemble
	pass in all flags S/SA keep state
	pass out all flags S/SA keep state


Check the translated results of NFS connections: 

	root@fbsd_pf:/mnt/t01-nfs # pfctl -ss | sort | grep ESTA
	all tcp 192.168.1.31:23452 -> 192.168.1.12:22       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.11:2049 <- 192.168.1.10:1023       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.11:2049 <- 192.168.1.11:1023       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.11:2049 <- 192.168.1.12:1023       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.11:2049 <- 192.168.1.13:1023       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:127 (192.168.1.13:1023) -> 192.168.2.11:2049       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:22 <- 192.168.2.1:35214       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:22 <- 192.168.2.1:52419       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:246 (192.168.1.10:1023) -> 192.168.2.11:2049       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:654 (192.168.1.12:1023) -> 192.168.2.11:2049       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:846 (192.168.1.11:1023) -> 192.168.2.11:2049       ESTABLISHED:ESTABLISHED

It happened to be a case that all clients assign port 1023 as source port. After NAT, the source ports for NFS clients 1-4 are translated to 246, 846, 654 and 127 on em0 respectively.


Without the line `nat ... port 111:1023` in pf.conf, the translated results may look like

	root@fbsd_pf:/etc # pfctl -ss | sort | grep ESTA
	all tcp 192.168.2.11:2049 <- 192.168.1.10:738       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:62929 (192.168.1.10:738) -> 192.168.2.11:2049       ESTABLISHED:ESTABLISHED

The source port on em0 is 62929 (>=1024) in this case and NFS mount will get a 'Permission denied' message unless server's insecure option is enabled.


### What if port range 111:1023 is not enough

Addresses may be used for the extra _nat_, too, as long as the network administrators allow.

	 nat on $ext_if inet  from $lan_nfs_cli    to $mainnas -> { 192.168.2.12, 192.168.2.13 } port 111:1023
	 nat on $ext_if inet  from $int_if:network to any      -> ($ext_if)

`192.168.2.13` is the alias of `192.168.2.12` in this experiment.


Check the rule:

	root@fbsd_pf:/etc # pfctl -sn
	nat on em0 inet from 192.168.1.10 port 111:1023 to 192.168.2.11 -> { 192.168.2.12, 192.168.2.13 } port 111:1023 round-robin
	nat on em0 inet from 192.168.1.11 port 111:1023 to 192.168.2.11 -> { 192.168.2.12, 192.168.2.13 } port 111:1023 round-robin
	nat on em0 inet from 192.168.1.12 port 111:1023 to 192.168.2.11 -> { 192.168.2.12, 192.168.2.13 } port 111:1023 round-robin
	nat on em0 inet from 192.168.1.13 port 111:1023 to 192.168.2.11 -> { 192.168.2.12, 192.168.2.13 } port 111:1023 round-robin
	nat on em0 inet from 192.168.1.0/24 to any -> (em0) round-robin
	nat on em0 inet from 192.168.1.0/24 to any -> (em0) round-robin


The result of two mounts shows different IP:port can be used: `192.168.2.13:478` and `192.168.2.12:418`

	root@fbsd_pf:/etc # pfctl -ss | sort | grep ESTA
	all tcp 192.168.2.11:2049 <- 192.168.1.10:843       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.13:478 (192.168.1.10:843) -> 192.168.2.11:2049       ESTABLISHED:ESTABLISHED

	   (... NFS client umounts and then re-mounts.)

	root@fbsd_pf:/etc # pfctl -ss | sort | grep ESTA
	all tcp 192.168.2.11:2049 <- 192.168.1.10:838       ESTABLISHED:ESTABLISHED
	all tcp 192.168.2.12:418 (192.168.1.10:838) -> 192.168.2.11:2049       ESTABLISHED:ESTABLISHED


### Linux iptables experiment

According to SNAT --to-source section in [Ubuntu Manpage: iptables](https://manpages.ubuntu.com/manpages/precise/en/man8/iptables.8.html):
> If no port range  is  specified,  then  source  ports
> below  512  will  be  mapped  to  other ports below 512: those between 512 and 1023
> inclusive will be mapped to ports below 1024, and other ports  will  be  mapped  to
> 1024 or above. Where possible, no port alteration will occur.

New enough iptables should reserve privileged ports natively.


With similar experiment settings to the previous:

* Host: Windows 10, QEMU 5.1.92
* Guest 1: NFS server, Debian 4.9.144-3
* Guest 2: NAT-machine, Debian 4.9.144-3, no entry at all in /etc/exports
* Guest 3-4: NFS clients 1 and 2, Debian 4.9.144-3 and FreeBSD bootonly installation CD (12.2-RELEASE-i386) respectively


The NAT settings (4 commands only):

	echo 1 > /proc/sys/net/ipv4/ip_forward
	iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
	iptables -A FORWARD -i ens3 -o br0 -m state --state RELATED,ESTABLISHED -j ACCEPT
	iptables -A FORWARD -i br0 -o ens3 -j ACCEPT


Then the clients can mount the shares from the NFS server simultaneously in my experiment.


## References
* [NFS server behind a PF firewall](http://blog.e-shell.org/227), via Google:<nfs client behind nat\>
* [Mount NFS export for machine behind a NAT](https://blog.bigon.be/2013/02/08/mount-nfs-export-for-machine-behind-a-nat/), via google:<nfs client behind nat\>
* [Cloud NAT address and port concepts \| Google Cloud](https://cloud.google.com/nat/docs/ports-and-addresses), via google:<nfs client nat\>
* [FreeBSD nat via PF: how to change from random UDP ports to incremental?](https://serverfault.com/questions/67249/freebsd-nat-via-pf-how-to-change-from-random-udp-ports-to-incremental), via google:<pf nat static-port\>
* [pfctl: Invalid argument. when using add with some netmasks](http://openbsd-archive.7691.n7.nabble.com/6-6-pfctl-Invalid-argument-when-using-add-with-some-netmasks-td381455.html), via google:<freebsd pf 192.168 become "64.168"\>
* [Quick-Tip: Linux NAT in Four Steps using iptables](https://www.revsys.com/writings/quicktips/nat.html), via google:<nat simple example iptables\>


## Updates
* 2021-03-02:
  * simple Linux iptables NAT settings that work for this topic.
  * libvirt bridge result similar to `-nic socket`.

[//]: <> (__END__)

