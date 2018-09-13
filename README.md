About
=========

There are various ways to manage your vagrant machines hostnames and IP
addresses.  Most of them require you to install additional vagrant plugins,
some of them are landrush,hostmanager and various others. some of them work,
some not.

One of them is vagrant-landrush, which works nicely but has problems in multi
user environments (spinning up an DNS Service for each vagrant vm)

This short manual should give you an easier way to setup DNS and DHCP
management of your Vagrant environment with libvirt backend.


Goal
-------------

The goal is to have a round robin style DNS and DHCP assignment for your
vagrant virtual machines. All machines should get an hostname from the dhcp
service and shall be resolvable forward and reverse from the host system aswell
as the other vagrant machines running.

As example, lets say we have an dhcp range that goes from:

 10.1.0.3   to  10.1.0.5

and associated hostnames like:

```
 sep003
 sep004
 sep005
```

that should match the ip given by the dhcp lease. The lease disappears as the
vagrant machine halts and can be re-used likewise by other machines if free.

It should be possible doing this without the need of third party plugins just
by modifying our libvirt network configuration. 

Prerequisites
-------------

Your vagrant boxes must honor DHCP's "hostname" option and allow to set their
hostname via DHCP for better integration.


Configuring Libvirt
-------------

Define a new libvirt network from the provided esample using:

```
virsh net-define examplenet.xml
```

Note: you may have to assign a special bridge name in the config file to
avaoid libvirt re-using an existing one.

 (<bridge name='virbrXX' stp='on' delay='0'/>)


And start the network:

```
virsh net-start my_cloud
```

This should spin up a new libvirt network with its according dnsmasq process:

```
$ ps axw | grep my_cloud
11714 ?        S      0:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/my_cloud.conf --leasefile-ro --dhcp-script=/usr/lib/libvirt/libvirt_leaseshelper

```

The example provides a lease range for three possible IP's and hostnames.

Spinning up vagrant box
-------------

Now initialize a new Vagrant box (this example uses debian9):

```
vagrant init generic/debian9
```

And edit your Vagantfile to contain the following section:

```
  config.vm.provider :libvirt do |domain|
        domain.management_network_name = "my_cloud"
        domain.management_network_address = "10.1.0.0/24"
  end
```

Followed by:

```
vagrant up
```

As the VM starts, it receives its dhcp lease and hostname from the dnsmasq
service:

```
~$ virsh net-dhcp-leases my_cloud
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
 2018-09-13 18:10:58  52:54:00:02:7b:cb  ipv4      10.1.0.4/24               sep004   
```

Getting into the box the hostname should have been set too:

```
~/mynet$ vagrant ssh
[..]
vagrant@sep004:~$ 
```

As all boxes use the local dnsmasq dns server, other boxes can happily
forward and reverse lookup each other:

```
~/mynet-2$ vagrant ssh
[..]
vagrant@sep003:~$ host sep004
sep004.mycloud.local has address 10.1.0.4
vagrant@sep003:~$ host 10.1.0.4
4.0.1.10.in-addr.arpa domain name pointer sep004.mycloud.local.
```

Freeing leases
-------------

Leases are automatically freed as the vagrant box is halted:

```
:~/mynet$ vagrant halt
==> default: Halting domain...
abi@cefix:~/mynet$ virsh net-dhcp-leases my_cloud
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
 2018-09-13 18:14:17  52:54:00:16:ee:ab  ipv4      10.1.0.3/24               sep003          -
```
