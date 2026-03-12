---
layout: post
title: Iptables - Firewall Tool For Linux. Part-1
tags: [Linux Administration]
comments: true
---

Iptables is a administration tool for packet filtering and NAT or simply it is a firewall tool for linux.

Iptables will monitor traffic using tables where rules are defined. 

There is several diffrent tables, each table contains a number of built in chains and may also contain user defined chains.

Each chain is a set of rules that can match against a set of packets. Each rule specifies what to do with that packet that matches, this is called **target**.

If  the packet  does  not  match, the next rule in the chain is examined. If it does match, then the next rule is specified by the value of the target, which  can  be the name of a user-defined chain, one of the targets described in iptables-extensions(8), or one of the special values **ACCEPT,DROP or RETURN**.

Following are the possible special values that you can specify in the target.

~~~
ACCEPT – Let the packet to pass through.

DROP   – Drop the packet. This is best if you don't want the source to realize your system exists.

RETURN – Stop traversing this chain  and  resume  at the  next rule in the previous (calling) chain.
~~~

Iptable currently  have 5 tables, **filter, nat, mangle, raw, security**.
	
In this blog we will discuss about filter table. which is the default table.

Filter table consists of 3 chains.

~~~
INPUT   - Filter incoming traffic coming to server.

FORWARD - For packets routed through the server.

OUTPUT  - For controlling locally generated packets before routing.
~~~

### 1 - Install and Setup Iptables

To install Iptables run the below command on the terminal.

~~~
sudo apt-get update

sudo apt-get install iptables
~~~

Run the below command to check if iptables is installed successfully.

~~~
sudo iptables -L -v --line-number
~~~

![masscan](/assets/images/iptables-basic.png)

This command will list all rules from **filter table** (default table). 

* **-L** is to list all the rules (from all the chains).

* **-v** is to enable verbose mode.

* **--line-number** is to list rules with line number.

As you can see that all chain policy are set to **ACCEPT** and no rules specified, which is not secure, because there is no packet filtering.

You can change the default policy to **DROP** to drop all the packets and you can specify rules to allow connection from the source you wish.

For example you have a server with some highly sensitive information and you want to connect to that server from only your ip address then you have to do the following,

* Allow traffic from only your ip address (xx.xx.xx.xx), do the following.

~~~
iptables -A INPUT -p tcp -s xx.xx.xx.xx -j ACCEPT

iptables -A OUTPUT -p tcp -d xx.xx.xx.xx -j ACCEPT
~~~

* Change default policy of all chains to DROP.

~~~
iptables -P INPUT DROP

iptables -P OUTPUT DROP

iptables -P FORWARD DROP
~~~

All done, now only you can connect to your server. This is very much helpful for a server which contains highly confidential data.

### 2 - Managing Rules

To manage rules to a chain you have to know some options of iptables.

* **-A** - Append rules to chain.

* **-D** - Delete matching rule from chain.

* **-F** - Delete all rules.

* **-i** - Network interface name.

* **-p** - Specify protocol.

* **-s** - Specify source.

* **-d** - Destination specification.

* **--dport** - Specify the port.

* **-j** - Target for the rule.

To understand clearly i will put some examples.

#### 1 - Block an IP address.

~~~
iptables -A INPUT -s XX.XX.XX.XX -j DROP
~~~

#### 2 - Allow all SSH connection

~~~
iptables -A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT

iptables -A OUTPUT -o eth0 -p tcp --sport 22 -j ACCEPT
~~~

### 3 - Allow MySql from specif IP address.

~~~
iptables -A INPUT -p tcp -s XX.XX.XX.XX --dport 3306 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 3306 -j ACCEPT
~~~

There are a lot of things that you can do with iptables.

Have a good day!.
