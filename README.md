# How to Configure PF on a FreeBSD 12.0 Digital Ocean Droplet

### Introduction

Networks are hostile places. Every organization will face this reality unless they are completely offline and have no internal threats. The firewall is arguably one of the most important lines of defense against cyber attacks. The ability to configure a firewall from scratch is an empowering skill that will enable the administrator take control of their networks.

In this tutorial we build a firewall from the ground up on a FreeBSD 12.0 droplet using [PF](https://man.openbsd.org/pf), a renown packet filtering tool that is maintained by the [OpenBSD](https://www.openbsd.org) project. It is known for its simple-syntax, user-friendliness, and extensive power. [PF](https://man.openbsd.org/pf) ships with the FreeBSD base system and is supported by a community of dedicated users. By default, [PF](https://man.openbsd.org/pf) is a stateful firewall. It stores information about connections in a *state table*. 

This lesson is *not* a command reference. It is designed to help you get started with [PF](https://man.openbsd.org/pf) on the Digital Ocean platform. Our goal is to create a portable base ruleset that will serve as a formidable starting point for new droplets. We will progress into more advanced concepts such as adaptive rulesets, bandwidth management, blacklisting, and more.

## Prerequisites

* A Unix-like workstation (BSD, Linux, or Mac)
* A [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) that permits inbound SSH traffic. Refer to this [quickstart guide](https://www.digitalocean.com/docs/networking/firewalls/quickstart).
* An [SSH keypair](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account) with a copy of the public key uploaded to your account
* A standard 1G FreeBSD 12.0 droplet in the region of choice. 

## Step 1 - Create the droplet

Create a standard 1G FreeBSD 12.0 droplet in the region of choice from the [control panel](https://cloud.digitalocean.com/login) in your account. The filesystem is irrelevant, it can be either ZFS or UFS. Your droplet should be configured with a **sudo** user and proper SSH configuration.

## Step 2 - Attach the droplet to a cloud-firewall

Upon creating our droplet we have no firewall enabled, therefore we risk being insecurely exposed to the open internet. For now we will attach our droplet to a [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) before doing anything else. Later, after building our [PF](https://man.openbsd.org/pf) ruleset, we will detach the droplet from the [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart).

## Step 3 - SSH into the droplet

Obtain the IP address of the droplet from the [control panel](https://cloud.digitalocean.com/login) and SSH into it:
```command
ssh myuser@XXX.XXX.XX.XXX
```

## Step 3 - Build the preliminary ruleset

There are two approaches to building a firewall ruleset: 1) default deny, or 2) default permit. The *default deny* approach blocks all traffic, and does not permit any traffic unless it is specified in a rule. The *default permit* approach does the exact opposite. It passes all traffic, and does not block any traffic unless it is specified in a rule. In this tutorial we use the *default deny* approach.

[PF](https://man.openbsd.org/pf) rulesets are written in a configuration file named `/etc/pf.conf`, which is the default location. It is okay to store it somewhere else, but that will have to be specified in `/etc/rc.conf`.

Create `/etc/pf.conf`:
```command
sudo vim /etc/pf.conf
```

[PF](https://man.openbsd.org/pf) filters packets according to three core actions: **block**, **pass**, and **match**. These actions are taken when a packet meets the criteria that we specify in our rules. The **pass** and **block** rules, as one might expect, pass and block traffic. The **match** rule performs an action on a packet without actually filtering it. For example we could perform network address translation (NAT) on a packet without moving it anywhere. 

Let's begin by blocking everything.

Add the following to `/etc/pf.conf`:
```
block all
```

This rule blocks all forms of traffic in either direction. Our **block** rule does not specify an **in** or **out** direction, therefore it defaults to both. The same logic applies to **pass** and **match** actions. 

This rule is perfectly legitimate for a local workstation that needs to be insulated from the world, however, it is largely impractical, and will not work with a remote droplet. Can you guess why? The reason is because we did not permit any **SSH** traffic. If this rule had been implemented, we would have locked ourselves out of our droplet. This is something to always keep in mind with remote machines. Every administrator has probably done this at least once! Let's create a more practical starting point.

Update the previous rule:
```
block all
pass in proto tcp to port 22
```

Alternatively, we can use the name of the protocol:
```
block all
pass in proto tcp to port ssh
```

Using port numbers versus service names, or vice versa, is largely a choice of style, and sometimes a necessity. For the sake of consistency we will use port numbers, unless there is a reason to do otherwise. There is a detailed list of protocols and their respective port numbers in the `/etc/services` file which you are highly encouraged to look at.

[PF](https://man.openbsd.org/pf) processes rules from top-to-bottom, therefore our ruleset initially blocks all forms of traffic, then passes it if the criteria on the next line is matched, which in this case is **SSH** traffic.

We can now **SSH** into our droplet, but we are still blocking all forms of outgoing traffic. This will become problematic becuase we will not be able to access critical services such as installing software, accurate time settings, and more.

Let's allow some outward traffic:
```
block all
pass out proto tcp to port { 22 53 80 123 443 }
pass out proto udp to port { 53 123 }
```

Our ruleset now permits outward **SSH, HTTP, DNS, NTP, and HTTPS** traffic in the outward direction, and blocks all inward traffic, except SSH. Notice we placed the port numbers inside of the curly brackets. This is a **list** in [PF](https://man.openbsd.org/pf) syntax. We also added the **pass out** rule for the **UDP** protocol on ports **53** and **123**. That is because **DNS** and **NTP** will occasionally toggle between both protocols. Do some Google searching on this topic. 

We're just about finished with our preliminary ruleset, but we still need to add some final touches.

Complete the preliminary ruleset:
```
set skip on lo0
block all
pass in proto tcp to port { 22 } tag SSH
pass out proto tcp to port { 22 53 80 123 443 }
pass out proto udp to port { 53 123 }
pass out inet proto icmp icmp-type { echoreq }
```

We added the `set skip` rule on our loopback device because it does not need to filter traffic. We also added a **tag** on our **SSH** rule which will label packets with the string "SSH" if they pass this rule. This increases visibility with log files, and during troubleshooting. Finally, we added a **pass out inet** rule for the **ICMP** messaging protocol, which allows us to use the [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) utility for troubleshooting purposes. The **inet** option represents the **IPv4** address family.

ICMP is an often controversial protocol that is fraught with many assumptions, however, if it is approached with care there is no harm in using it. We just want to use [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) with a droplet, that's it. The [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) utility relies on [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) *echo messages*. Therefore in our **pass out inet** rule we *only* permit messages of type *echoreq*. Every other type of message will be blocked in all directions. If we need more message types in the future, we can easily add them. You may have noticed that [PF](https://man.openbsd.org/pf) allowed us to incorporate these codes directly from the [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) protocol. Yes it did! See `man icmp` for additional codes.


## Step 4 - Test the preliminary ruleset

Although we are far from finished, we have a working ruleset that provides basic functionality to a fresh droplet. That said, let's test it before we get too far. Rulesets are loaded with the [pfctl](https://man.openbsd.org/pfctl) utility, which is the built-in command-line tool for [PF](https://man.openbsd.org/pf). [PF](https://man.openbsd.org/pf) rulesets are nothing more than text files, which greatly simplifies the process of building and modifying rulesets. There are no delicate procedures required to load new rulesets because the state of the firewall is irrelevant to [PF](https://man.openbsd.org/pf). Simply load the new ruleset, and the old one is gone. There is rarely ever a need to flush an existing ruleset, and it is generally a bad idea.

Enable [PF](https://man.openbsd.org/pf) in `/etc/rc.conf`:
```command
sudo sysrc pf_enable="YES"
sudo sysrc pflog_enable="YES"
```

Reboot the droplet:
```command
sudo reboot
```

Load the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

If there are no errors or messages, we're good-to-go!

Reboot the droplet again:
```command
sudo reboot
```

Detach the droplet from your [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) from the [control panel](https://cloud.digitalocean.com/login) in your account. The connection will be dropped. [PF](https://man.openbsd.org/pf) is now the acting firewall. Give it a few minutes to update.

SSH back into the droplet:
```command
ssh root@XXX.XXX.XX.XXX
```

View some state information:
```command
sudo pfctl -si
```

Test internet connectivity:
```command
ping 8.8.8.8
```

Test DNS by updating the **pkgs** repository:
```command
sudo pkg upgrade
```

If any packages need to be upgraded, go ahead and do that. If all of these services are working, that means our ruleset is working and we can proceed. 

## Step 5 - Complete the base ruleset

Here we will complete the base ruleset. Rules in [PF](https://man.openbsd.org/pf) are structured in a particular order. If you find yourself getting confused in the order of things, refer to the sample rulesets at the end of the tutorial.

### Incorporate macros and tables

In our preliminary ruleset we hard coded all of the parameters into each rule, i.e., the port numbers. This will become unmanageable as our ruleset expands. For organizational purposes [PF](https://man.openbsd.org/pf) offers **macros**, **lists**, and **tables**. We already used lists, which were the port numbers inside of the curly brackets. We can also assign a **list**, or even a single value, to a variable using **macros**. Let's reorganize our parameters.

Move our parameters into macros:
```
ext_if = "vtnet0"
tcp_services= "{ 22 53 80 123 443 }"
udp_services= "{ 53 123 }"
icmp_messages= "{ echoreq }"
```

Now we can call the variable names in our rules using the dollar sign, i.e., `$tcp_services`. Another macro was also created for our network interface **vtnet0**, which is the default interface on a single FreeBSD droplet.

We can also use **tables**, which are similar to macros, but are designed to hold groups of IP addresses. Let's create a **table** for non-routable IP addresses, which often play a role in Denial of Service attacks (DOS). Our droplet should not send or receive packets to or from these source addressess. For our **table** we will use all of the IP addresses specified in [RFC6890](https://tools.ietf.org/html/rfc6890), which defines special-purpose IP address registries.

Create the table:
```
table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }
```

### Traffic normalization

Traffic normalization is a broad term that refers to the handling of various packet discrepancies. Packets often arrive fragmented, with erroneous TCP flag combinations, or with bogus IP addresses (spoofed). Some of these issues originate from malicious actors, others are the result of the many implementations of the TCP/IP protocol suite.

### Scrubbing

[PF](https://man.openbsd.org/pf) provides a **scrub** keyword to handle packet fragmentation. It will re-assemble fragmented packets, discard any cruft, and drop packets that have erroneous TCP flag combinations. We want to subject all incoming traffic to a **scrub** rule. 

Add the **scrub** rule:
```
scrub in
```

Please be aware that some applications require special attention in regards to scrubbing. NFS filesystems, for example, may require that the `no-df` and `random-id` options are set because there are some operating systems will automatically set the **dont-fragment** bit. When in doubt, research all of the **scrub** options. For most purposes the above rule works fine.

### Antispoofing

Spoofing is the practice of using fake IP addresses to disguise the real host address, typically for malicious purposes. [PF](https://man.openbsd.org/pf) knows when certain IP addresses are traveling in directions that are impossible, and has a built-in keyword that deals with them quickly.

Add antispoofing:
```
antispoof quick for $ext_if
```

We also introduced the **quick** keyword, which tells [PF](https://man.openbsd.org/pf) to execute a rule and exit the ruleset immediately. This is desirable for handling false IP addresses because we want those packets to be discarded immediately, and have no reason to process the remainder of the ruleset.

### Blocking non-routable IP addresses

Previously we created the `table <rfc6890>`, but we haven't created any rules against it.

Create rules against table <rfc6890>:
```
block in quick on egress from <rfc6890>
block return out quick on egress to <rfc6890>
```

We also introduced the **return** option, which compliments our **block out** rule. The **block return out** rule drops the packet, and also sends an RST message to the host that tried to make this connection. Finally, we introduced the **egress** keyword, which automatically finds the default route. This is often a cleaner method of finding the default route, especially on complex networks.

## Step 6 - Review the base ruleset

We now have a complete base ruleset that permits inbound **SSH** traffic, enforces some network hygiene measures, in place, and allows us access to some vital services from the internet.

Here is our complete base ruleset:
```
ext_if = "vtnet0"
tcp_services= "{ 22 53 80 123 443 }"
udp_services= "{ 53 123 }"
icmp4_messages= "{ echoreq }"
table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }

set skip on lo0
scrub in
antispoof quick for $ext_if
block in quick on $ext_if from <rfc6890> 
block return out quick on egress to <rfc6890>
block all
pass in log proto tcp to port { 22 } tag SSH
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass inet proto icmp icmp-type $icmp4_messages
```

## Step 6 - Test the base ruleset

We can take a test run using `pfctl -nvf /etc/pf.conf`, which runs the ruleset without loading it, and throws error messages if something is wrong. You'll notice that when you run this command the rules look much different. In reality, this is what they look like in their full-form. [PF](https://man.openbsd.org/pf) allows us to write shortcut versions of rules for readability, which is how we have written our rules thus far. Viewing the rules in their full-form with `pfctl -nvf /etc/pf.conf` is a good way to learn the [PF](https://man.openbsd.org/pf) syntax.

Take a test run:
```command 
sudo pfctl -nvf /etc/pf.conf
```

There should be no errors.

Load the base ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

There should be no errors.

Make a copy of the base ruleset:
```command
sudo cp /etc/pf.conf /etc/pf.conf.orig
```

Test internet connectivity:
```command
ping 8.8.8.8
```

Test DNS by updating the **pkgs** repository:
```command
sudo pkg upgrade
```

If there are any packages to update, go ahead and do that.

Reboot the droplet:
```command
sudo reboot
```

## Step 7 - Introduce anchors

[PF](https://man.openbsd.org/pf) **anchors** are used to source rules into the main ruleset, either on-the-fly with `pfctl`, or with an external text file. Here we will introduce **anchors** without actually adding them to our ruleset. We will demonstrate their usefulness with our **table <rf6890>**. 

Instead of writing tables in the main ruleset, we can write them in an external text file, and call them from an anchor. This applies not only to tables, but to any rule snippets.

Create a file named `/etc/non_routes`:
```command
sudo vim /etc/non_routes
```

Add the following rules:
```
ext_if = "vtnet0"

table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }

block in quick on $ext_if from <rfc6890> 
block return out quick on egress to <rfc6890>
```

Add the anchor:
```
*--snip--*
set skip on lo0
scrub in
antispoof quick for $ext_if
anchor non-routes_anchor
load anchor non-routes_anchor from "/etc/non_routes"
block all
*--snip--*
```

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Another intelligent feature of anchors is that they allow us to add rules to a running ruleset on-the-fly without reloading it. This can be useful for testing, quick-fixes, emergencies, etc. For example, let's imagine that an internal host is acting strange and we want to block it from making any external connections without hard coding a rule into the main ruleset. We can place an anchor just below the **block all** rule, and insert that rule manually from the command-line. We'll name the anchor `rogue_hosts`

Add the anchor:
```
*--snip--*
set skip on lo0
scrub in
antispoof quick for $ext_if
block in quick on $ext_if from <rfc6890> 
block return out quick on egress to <rfc6890>
block all
anchor rogue_host
*--snip--*
```

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Block the host from the command-line:
```command
sudo sh -c 'echo "block in quick from XXX.XXX.XX.X" | pfctl -a rogue_host -f -'
```

There are many pros and cons to using anchors, all of them are completely situational. Anchors may help reduce clutter in a ruleset by breaking it up into separate text files. Managing multiple files, however, might be cumbersome and hard to read. Anchors can be useful for adding rules on-the-fly, however, someone needs to be present to add those rules. In other scenarios anchors are necessary for third party tools that are designed to integrate with [PF](https://man.openbsd.org/pf).

## Step 8 - Defending against cryptography exploits

Cryptography exploits, such as brute force or key search attacks, occur when attackers systematically decrypt passwords in order to access the system. These are often performed with a great deal of sophistication, and usually succeed if the target machine has weak passwords. In the strongest terms we recommend using public-key authentication on your droplets. However, even with key authentication, attackers or netbots can be a nuisance, and key search attacks are not impossible. We can help prevent these attacks by using [PF](https://man.openbsd.org/pf)'s state monitoring capabilities, coupled with the **overload** mechanism, which will blacklist abusive IP addresses by adding them to a special table. 

### Blacklisting IP addresses with the overload mechanism

Add the overload table:
```
table <blackhats> persist
```

Modify our previous SSH rule:
```
pass in proto tcp to port { 22 } \
    keep state (max-src-conn 15, max-src-conn-rate 3/1, \
        overload <blackhats> flush global)
```

The **persist** keyword allows us to maintain empty tables. Next we added some **keep state** options to our previous SSH rule. These will act as the criteria for the **overload** mechanism. The idea is to significantly reduce the frequency at which external hosts are allowed to make connections to our droplet. For example, if port 22 receives 100 failed logins in 5 seconds, it is likely a brute force attack, and the source IP address should be blacklisted.

The **max-src-conn 15** option permits only 15 simultaneous connections from a single host. The **max-src-conn-rate 3/1** option will only allow 3 new connections per second from a single host. If these options are exceeded, the **overload** mechanism adds the source address of the host to the <blackhats> table, where it will remain until we remove it. Finally, the **flush global** option immediately drops the connection.

We have enforced strict measures on our SSH traffic for good reason, however, these restrictions are not suitable for every type of service. When servers experience sharp increases in connection attempts, it is not always an attacker. It could be a random web service that was dynamically assigned an IP address. Blocking it may or may not be the right decision, depending on the circumstances. Services that can afford liberal connection policies should not share **overload** mechanisms with services that cannot afford them.

Over time the <blackhats> table will grow and should be cleared periodically. It is unlikely that an attacker will continue using the same IP address, therefore it is non-sensical to keep it in the table for long periods of time. The table can be cleared manually with `pfctl`, or we can automate the process with **cron**, which is FreeBSD's job scheduler.

### Clear the overload table manually

Clear IP addresses that are 48 hours old:
```command
sudo pfctl -t blackhats -T expire 172800
```

### Clear the overload table with a cron job

Create a shell script:
```command
sudo vim /usr/local/bin/clear_overload.sh
```

Add the following:
```
#!/bin/sh

pfctl -t blackhats -T expire 172800
```

Make the file executable:
```command
sudo chmod 755 /usr/local/bin/clear_overload.sh
```

Create a crontab:
```command
sudo crontab -e
```

Add the following:
```
# minute	hour	mday	month	wday	command
  *		0	*	*	*	/usr/local/bin/clear_overflow.sh
```

This creates a **cron job** that runs the `clear_overload.sh` script everyday at midnight. It will remove IP addresses that are 48 hours old from the overload table <blackhats>.

## Step 9 - Traffic shaping (optional)

By default, [PF](https://man.openbsd.org/pf) filters packets on a first-come-first-serve basis. For high-traffic applications, such as a public website, we may wanto to process packets on ports 80 and 443 before processing any others. Another concern is bandwidth, which can be an expensive resource that the administrator may want to preserve for services that need it the most. Thankfully, [PF](https://man.openbsd.org/pf) provides the **prio** option for prioritizing packets, and the **queue** option for managing bandwidth. 

### Prioritizing traffic with prio

The **prio** option prioritizes traffic according to a number range of 0 thru 7. Zero is the slowest, and seven is the fastest. Below are some of examles of the **prio** option.

Increase the priority of inbound SSH traffic:
```
pass in proto tcp to port { 22 } set prio 4
```

Give the maximum priority to web traffic:
```
pass proto tcp to port { 80 443 } set prio 7
```

Lower the priority for *name* and *time* services:
```
pass out proto tcp to port { 53 123 } set prio 2
pass out proto udp to port { 53 123 } set prio 2
```

Prioritize web traffic to 6, and *ACK* and *lowdelay* packets to 7:
```
pass proto tcp to port { 80 443 } set prio (6,7)
```

The last example highlights a powerful feature of **prio**, which is the tuple option. The first value prioritizes regular packets, and the second value prioritizes *ACK* and *lowdelay* packets. TCP connections send *acknowledgement* (ACK) packets which contain no data. By default, ACK packets wait in line with regular packets, which means they have the potential to clog up the system, and should be moved along quickly. Furthermore, packets may arrive with a *lowdelay* set in their *type of service* (TOS) fields, which means they should also be processed before other packets. Therefore in the above rule we set the priority for regular web traffic to 6, and we set the priority for *ACK* and *lowdelay* packets to 7, all of which are high priorities.

### Managing bandwidth with queue

On it own, the **prio** option may not always suffice because it strictly reshuffles the order in which packets are filtered. In addition to **prio** we can also place limits on the amount of bandwidth that certain packets use with the **queue** option.

Let's assume we want to reserve the majority of our outward bandwidth for web services. Digital Ocean droplets provide 1000 GB/month of free outward data transfers. We can create a queue that allocates this amount to our network interface `vtnet0`. This will not be a perfect representation of our total bandwidth activity, however, it gives us a decent amount of control, and could also protect us from a rogue application eating up our bandwidth.

[PF](https://man.openbsd.org/pf) queues are structured like trees with parent-child hierarchies. Every queue structure has a root queue and a default queue. Bandwidth is specified in kilobits **K**, megabits **M**, or gigabits **G**, *per second*. Let's create a root queue named **rootq**, with two child queues named **web** and **catchall**.

Create a basic queue:
```
queue rootq on $ext_if bandwidth 200M
    queue web parent rootq bandwidth 150M
    queue catchall parent rootq bandwidth 50M default
```

We allocated 200M per second of bandwidth to **rootq**. We portioned 150M of that to its child queue **web**, and the remainder to its other child queue **catchall**. In reality, bandwidth activity is never as uniform as it appears in our basic queue. Queues will borrow bandwidth from eachother during traffic spikes, which can potentially hinder the availability of a service. [PF](https://man.openbsd.org/pf) provides some options to control this.

Revise the basic queue:
```
queue rootq on $ext_if bandwidth 200M max 200M
    queue web parent rootq bandwidth 150M
    queue catchall parent rootq bandwidth 50M min 15M max 50M burst 75M for 100ms default
```

We added a **max** keyword to both **rootq** and **catchall** so that they do not exceed their allocations through borrowing. We do not add the **max** keyword to the **web** queue because web services are the top priority. Since our **web** queue is essentially left unchecked, chances are it will borrow from **catchall**. For this reason we assigned the **min** keyword to **catchall** to ensure that it always has at least 15M of bandwidth available at all times. We also added the **burst** keyword to **catchall**, which allows it to exceed its 50M allocation temporarily in the event of a traffic spike.  Finally, we make **catchall** our **default** queue so that packets will automatically be sent there if they match no other rules. Despite the nuances and quirks involved with bandwidth management, in the end queues are essential when bandwidth is a limited resource.

Add subqueues:
```
queue rootq on $ext_if bandwidth 200M max 200M
    queue web parent rootq bandwidth 100M
        queue developers parent web bandwidth 50M max 50M
    queue catchall parent rootq bandwidth 50M min 15M max 50M burst 100M for 100ms default
```

Here we added a subqueue to the **web** queue, allocating 50M of its bandwith to a **developers** queue, using a 50M **max** to prevent it from borrowing.

Add a queue to the main ruleset:
```
pass out proto tcp to port { 80 443 } set queue web set prio (6,7)
```
Notice we coupled the **queue** option with the **prio** option because we can do that! This means we will have some pretty fast and efficient http traffic!

## Step 9 - Monitoring and logging

Our firewall is of little use if we cannot see what it is doing. Policy decisions in network security are highly dependent on packet analyses, which inevitably invlove examining log files. With [PF](https://man.openbsd.org/pf), logging occurs on a psuedo-device known as the **pflog** interface, which can be viewed with `ifconfig`. Since it is an interface, various userland tools can be used to access the data that it logs. 

To create logs we add the **log** keyword to any of our rules. [PF](https://man.openbsd.org/pf) will only log what we tell it to. Once the rules are enacted, [PF](https://man.openbsd.org/pf) makes copies of the packet headers, writing them to `/var/log/pflog` in binary format.

### Accessing log data with tcpdump

There are multiple ways to access [PF](https://man.openbsd.org/pf) log data, but few tools are capable of generating the level of detail that can be achieved with [tcpdump(8)](https://man.openbsd.org/tcpdump.8), which is part of the FreeBSD base system.

We begin by logging what's most important to us because logging everything will become unwieldly. We certainly want to know who is SSH-ing into our droplet. This is a rule that we will actually add to our main ruleset.

Add the **log** keyword to our SSH rule:
```
pass in log proto tcp to port { 22 }
```

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Now we should see some log data:
```command
sudo tcpdump -ner /var/log/pflog
```

You can also save it to a file:
```command
sudo tcpdump -ner /var/log/pflog > ssh_log
```

We can also view the logs in real-time directly from the `pflog0` interface. For a single droplet with no network activity, a terminal multiplexer might be required on the remote workstation, such as **tmux**. For example, we could SSH into the droplet on one terminal, initiate real-time logging, then SSH into the droplet on another terminal, and we should see this in real-time.

Initiate real-time logging:
```command
sudo tcpdump -nei pflog0
```

There is a wealth of information on the internet about **tcpdump**, including the official [website](https://www.tcpdump.org), which is extremely well documented. The **tcpdump** utility is one of the best packet analysis tools in the industry, and has been strongly supported for many decades. 

### Accecssing log files with pftop (recommended)

The **pftop** utility is an excellent tool for quickly viewing active states and rules in real-time. It is named after the well known **top** utility due to its similarities.

Install **pftop**
```command
sudo pkg istall pftop
```

Run **pftop**
```command
sudo pftop
```

It will display something like this:
```
PR    DIR SRC                   DEST                           STATE                AGE       EXP   PKTS BYTES
tcp   In  251.155.237.90:27537  157.225.173.58:22     ESTABLISHED:ESTABLISHED  00:12:35  23:59:55   1890  265K
tcp   In  222.186.42.15:25884   157.225.173.58:22       TIME_WAIT:TIME_WAIT    00:01:25  00:00:06     22  3801
udp   Out 157.245.171.59:4699   67.203.62.5:53           MULTIPLE:SINGLE       00:00:14  00:00:16      2   227
```

### Creating graphical representations of our packet filter with pfstat (optional)

Often times we need graphical representations of our packet filter to show clients, colleagues, or maybe the boss. The **pfstat** utility is a lightweight tool that is perfect for this. The way it works is that it populates a `/var/db/pfstat.db` database with data that it collects from `/dev/pf`. This is usually done continuously with a **cron** job, but can also be done manually from the command-line. We tell **pfstat** what to store in the database in the `/usr/local/etc/pfstat.conf` configuration file. We can then translate that data into graphs that are stored in either **jpg** or **png** image formats.

Unfortunately, with a single fresh droplet we do not have much data to work with, but we'll try our best! For demonstration purposes we'll make a simple graph that shows us how many packets have passed, and how many have been blocked.

Install pfstat:
```command
sudo pkg install pfstat
```

Create the configuration file:
```command
sudo vim /usr/local/etc/pfstat.conf
```

Add the following:
```
collect 1 = interface "vtnet0" pass packets in ipv4 diff
collect 2 = interface "vtnet0" block packets in ipv4 diff

image "/home/myuser/pfstat.jpg" {
        from 7 days to now
        width 300 height 200
	left
            graph 1 "pass in" "packets/s" color 0 192 0
	right
            graph 2 "block in" "packets/s" color 0 0 255
}
```

Create the cron job:
```command
sudo crontab -e
```

Add the following entries:
```
*	*	*	*	*	/usr/local/bin/pfstat -q

0	0	1	*	*	/usr/local/bin/pfstat -t 30:180
```

The database should now exist at `/var/db/pfstat.db`

The above entries will populate the database continuously. Now that we have automated the process the database will continue to grow, so we use the **-t** flag to periodically clear it out. The additional values both represent days. The first value represents uncompressed images, and the second represents compressed images. The above rule deletes uncompressed images that are 30 days old, and compressed images that are 180 days old. Uncompressed images are for high resolution images, and should be removed first to keep the size of the database manageable. See `man pfstat`. 

Generate a graph image:
```
sudo pfstat -p
```

Xorg is required to view graphical images. Since we have no intention of installing **Xorg**, or even forwarding X sessions to our droplet, we will copy the image from the droplet to our remote workstation using **scp**.

Copy the image to the remote workstation:
```command 
sudo scp myuser@XXX.XXX.XX.XX:pfstat.jpg /home/myuser
```

As mentioned, there's a strong chance that nothing will appear on the graph because we're using a fresh droplet. We can try flooding the server with ping from our local workstation to generate mass quantities of packets. The server will block the ping flood.

Flood the server with ping:
```command
sudo ping -f XXX.XXX.XX.XX
```

Copy the image again:
```command 
sudo scp myuser@XXX.XXX.XX.XX:pfstat.jpg /home/myuser
```

The image should now show some graphical data.

### Create additional log interfaces

It is absolutely possible to create additional log interfaces with a `/etc/hostname` file just as we would do with any other interface.

Create a `pflog1` interface:
```command
sudo vim `/etc/hostname.pflog1` 
```

Add the following:
```
up
``` 

Enable it in `/etc/rc.conf`: 
```command
sudo sysrc pflog1_enable="YES"
```

## Conclusion

We now have a portable base ruleset that can serve as a formidable starting point for all new droplets. Refer to the sample rulesets below, and tuck them away somewhere safe. Do not blindly copy them into your droplets without considering exactly what it is you want to accomplish with [PF](https://man.openbsd.org/pf). They are essentially building blocks that we can tailor into our networks. Hopefully this tutorial was useful in getting you up to speed with [PF](https://man.openbsd.org/pf), and also increasing your understanding of Unix networking. See you in the next one!

## Sample rulesets

### Sample ruleset 1 (identical to step 6)
```
ext_if = "vtnet0"
tcp_services= "{ 22 53 80 123 443 }"
udp_services= "{ 53 123 }"
icmp4_messages= "{ echoreq }"
table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }

set skip on lo0
scrub in
antispoof quick for $ext_if
block in quick on $ext_if from <rfc6890> 
block return out quick on egress to <rfc6890>
block all
pass in log proto tcp to port { 22 } tag SSH
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass inet proto icmp icmp-type $icmp4_messages
```

### Sample ruleset 2
```
ext_if = "vtnet0"
tcp_services= "{ 22 53 123 }"
udp_services= "{ 53 123 }"
web_services= "{ 80 443 }"
icmp4_messages= "{ echoreq }"
table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }

queue rootq on $ext_if bandwidth 200M max 200M
    queue web parent rootq bandwidth 150M
    queue catchall parent rootq bandwidth 50M min 15M max 50M burst 75M for 100ms default

set skip on lo0
scrub in
antispoof quick for $ext_if
block in quick on $ext_if from <rfc6890> 
block return out quick on egress to <rfc6890>
block all
pass in proto tcp to port { 22 } \
    keep state (max-src-conn 15, max-src-conn-rate 3/1, \
        overload <blackhats> flush global) tag SSH
pass proto tcp to port $web_services set queue web set prio (6,7)
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass inet proto icmp icmp-type $icmp4_messages
```

- **Sample ruleset 1** is a starting point for fresh droplets that are not yet running any public facing services.

- **Sample ruleset 2** builds on ruleset 1, and is suitable for a public facing web application. It includes some highly restrictive measures on its SSH port, and some bandwidth and packet prioritization features on its web traffic. The idea is to optimize web traffic to provide a pleasant consumer experience.









