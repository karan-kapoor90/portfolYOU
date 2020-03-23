---
title: Kubernetes Networking
tags: [kubernetes, cloud native, Networking, CNI]
style: 
color: 
description: A primer guide for Networking in Kubernetes
---


## Basics

To see network interfaces on a host, use the command `ip link`.

Once you have a link that connects to the same switch, say 192.168.10.0, you can assign IP addressed to hosts on that network using the command `ip addr add 192.168.10.10/24 dev eth0`. 

To connect different networks together (assume switch1 @ 192.168.1.0 and switch2 @ 192.168.2.0), you need to use a Router. The router is an intelligent device. It assigns an IP to both the networks (say 192.168.1.1 and 192.168.2.1 respectively)

On a linux machine, run the `route` command to see the network configuration of your machine. To enable traffic from network 1 to network 2, run the command `ip route add 192.168.2.0/24 via 192.168.1.1` where 192.168.1.1 is the gateway for network 1. This needs to be done on every machine in network1.  

To get a network connected to the internet, connect the router to the internet and set a configuration to push all traffic for an un-recognised IP address (such as the address to google.com etc.) to the gateway which is the router. Do this using `ip route add default via 192.168.2.1`, this will add an entry to the machine's routing table.

To make an entry into a machine's etc/hosts file `cat >> /etc/hosts`. To get a machine's hostname, do `hostname`.

Every server has a file called `/etc/resolv.conf` in which you give the DNS servers name against the keyword `nameserver` and the IP address for the DNS server. By default a server will look in the hosts file before looking at the DNS server. This priority can be changed by making a change in the `/etc/nsswitch.conf`. A free nameserver is 8.8.8.8 hosted by Google.

If a name is not available in either location, domain name resolution will fail.

For example there is a public website called web.mycompany.com. But if you work for this company and internally you wish to be able to refer to it as just web, you can do so by adding an entry into the `/etc/resolv.conf` with the `search` keyword. So for web.mycompany.com, add an entry `search mycompany.com`. Then if you ping web, it will automatically append mycompany.com to it and search for it, find it and resolve correctly. If you specify the domain in the query, meaning if you did a `ping web.mycompany.com` it would exclude including the search parameter mycompany.com to the DNS resolution request.

**DNS Management mapping***
A records - name to IPv4 mapping
AAAA records - name to IPv6 mapping
CNAME records - mapping a name to a name

`ping` may not always be the right tool to reach a website. `nslookup` does a lookup for a domain name in the DNS server, however it skips name resolutions from the /etc/hosts file. So any names in the hosts file will be ignored by nslookup. `dig` can also be used for DNS namespace lookup.

### Network Namespaces

Containers run on something called Process Namespaces which is a core Linux feature. What thart basically means is that every docker container is a process (pid) running on a host machine. Inside that PID there are other processes running such that they are not aware of the parent PID or of the other PID's on the host. This in turn also ensures that PID numbers between the containers and the hosts never clash. It also allows for running multiple docker containers to run on the same host machine without PID clashes. The host machine has all the permissions to look into the PID's running under a particular PID.

Run `ps aux` on the host machine to get a list of all the running processes. Run `ps aux` in a container and you'll see a much smaller list of processes that do not coincide with the host. 

Similarly there are Network Namespaces for network level isolation. Containers run in their own network namespace, meaning that they have their own ARP and Route tables and do not see the network interfaces of the host machine. They may have their own virtual network interfaces. 

To create a new network namespace on a linux host, do `ip netns add <ns_name>` for example `ip netns add blue`. To get a list of namespaces, do `ip netns`.

Now in order to see the network interfaces attached with a host machine, we run the command `ip link`. In order to view the network interfaces for a network namespace such as the `blue` we created earlier, use the command `ip netns exec blue ip link` or the command `ip link -n blue`. This will only show the loopback interfaces and not the eth entries.

Similarly for the arp tables. `arp` shows entries on the host level;, while `ip netns exec blue arp` runs it in the container namespace called blue.

The same is applicable for routing tables as well. `route` for the host machine routes while `ip netns exec blue route` for the routes defined on a container level.

To connect 2 network namespaces:

```bash

#1. Create a connection between 2 virtal ethernet ports of type veth
$ ip link add veth-red type veth peer name veth-blue   # Create the connection pipe and include the interfaces that it will be connecting.

#2. Link the virtual ethernet ports to the correct network namespaces
$ ip link set veth-red netns red   # attach the virtual interface veth-red to the network interface red
$ ip link set veth-vlue netns blue   # attach the virtual interface veth-blue to the network interface blue

#3. Assing IP addresses to the virtual ethernet ports created on both the namespaces
$ ip -n red addr add 192.168.12.1 dev veth-red   # attaching the IP address to the red ethernet interface
$ ip -n blue addr add 192.168.12.2 dev veth-blue   # attaching the IP address to the blue ethernet interface

#4. Activate the network interfaces
$ ip -n red link set veth-red up
$ ip -n blue link set veth-blue up

#5. Ping the blue namespace from the red namespace to test
$ ip netns exec red ping 192.168.12.2   # ip address of blue being pinged from red ns.
```

***What does the network isolation look like?***

If you check the arp tables of the 2 namespaces as well as the host, you'll notice that the host has no idea of the network IP's and connectivity b/w the red and blue namespaces, however the red and blue namespaces have each other's IP's and MAC addresses in their respective ARP tables.

```bash
$ ip netns exec red arp   # will have an entry for the blue namespace IP address and MAC
$ ip netns exec blue arp   # will have an entry for the red nemepace IP addr and MAC
$ arp   # will bring up the host ARP details and will not have either of the namespaces's IP's
```


> This looks easy when you have 2 namespaces. When there are multiple namespaces to be connected, you need a virtual switch (bridge) on the host to do this virtual networking between the namespaces. You use softwares like Linux Bridge and Open vSwitch for this. 

To use a bridge, the following needs to be done. 

```bash
#1. Create a bridge ethernet interface on the host
$ ip link add v-net-0 type bridge

#2. It should appear in the list of ip-links available
$ ip link

#3. To bring it up (by default it is down at creation)
$ ip link set dev v-net-0 up

#4. (optional) if a connection already exists between 2 network interfaces that you're trying to connect, delete those interfaces as they will not make sense anymore, the network interfaces will be connecting via the new bridge now
$ ip -n red link del veth-red   # delete the red interface
$ ip -n red link del veth-blue   # delete the blue interface (this one actually gets deleted automatically since they're a pair)

#5. Connect the red network to the bridge
$ ip link add veth-red type veth peer name veth-red-br   # veth-red-br is the interface name on the bridge end
$ ip link add veth-blue type veth peer name veth-blue-br   # veth-blue-br is the interface name on the bridge end

#6. Now attach the veth-red interface name to the red network and the veth-red-br interface to the bridge network
$ ip link set veth-red netns red
$ ip link set veth-red-br master v-net-0

# and the same thing for the blue network

$ ip link set veth-blue netns blue
$ lp link set veth-blue-br master v-net-0

#7. Now attach IP addresses to the 2 networks

$ ip -n red addr add 192.168.12.1 dev veth-red
$ ip -n blue addr add 193.168.12.2 dev veth-blue

#8. Start the interface so the 2 networks are connected to the bridge
$ ip -n red link set veth-red up
$ ip -n blue link set veth-blue up

#9. To start connectivity between the 2 networks, the bridge in between also needs an IP so the machine knows where to route through. 
$ ip addr add 192.168.12.5/24 dev v-net-0   # assigns an IP address to the bridge 
```

In order for the bridged network connected above to have outside (w/ wo internet) connectivity, the network's routing table must have an entry for the gateway that will provide this access.
The gateway is a host connected to this whole network we setup earlier and also connected to the other network (or internet). Hence this hsot has multiple IP addresses, one in the virtual network we just created and one ethernet connection to the outside network. We use the gateway's IP on the bridged network side in the routing table to route traffic outside of the private network. 

```bash
#10. The gateway's IP address on the private network is assumed 192.168.12.5
$ ip netns exec blue ip route add 192.168.12.0/24 via 192.168.12.5
```

> At this point, the outside network will be reachable but the responses from the external servers will not come back because there is no NAT at the gateway, hence the gateway doesn't know how to route messages back to the individual hosts making the call. We need to setup NAT such that the Gateway server replaces the FROM details from the outside response and puts its own name and address instead, and sends out the requests to external servers with it's own IP address so the internal network is not exposed to the outside world.

```bash
#11. Setting up NAT on the gatway node
$ iptables -t nat -A POSTROUTING -s 192.168.12.0/24 -j MASQUERADE

#12. Setup the gatway node as the Default route on the routing table so it can reach the internet from the gateway
$ ip netnx exec blue ip route add default via 192.168.12.5
```

To facilitate traffic moving from the public internet into the private network, for example, to access a website hosted on a machine on the private network, we can either expose the IP address of the private machine or setup a port forwarding rule on the gateway host. 

```bash
#13. Forward all requests on the port 80 of the gateway node to port 80 of the machine 192.168.12.2 in the private network
$ iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.12.2:80 -j NAT   #  
```
