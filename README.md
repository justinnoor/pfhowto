# How to Configure PF on a FreeBSD 12.1 Digital Ocean Droplet Part 1 (1ST DRAFT)

### Introduction

Networks are hostile places. Every organization will face this reality unless they are completely offline and have no internal threats. The firewall is arguably one of the most important lines of defense against cyber attacks. The ability to configure a firewall from scratch is an empowering skill that enables the administrator take control of their networks.

In this tutorial we build a firewall from the ground up on a FreeBSD 12.01 droplet using [PF](https://man.openbsd.org/pf), a renown packet filtering application that is maintained by the [OpenBSD](https://www.openbsd.org) project. [PF](https://man.openbsd.org/pf) is part of the FreeBSD base system, and is a foundational tool in the world of Unix administration. It is known for its simple-syntax, user-friendliness, and astonishing power.

This lesson is *not* a command reference. It is designed to create a working base ruleset for FreeBSD droplets on the Digital Ocean platform. This base ruleset can serve as a formidable starting point for new droplets. After creating our base ruleset we will progress into more advanced concepts such as anchors, adaptive rulesets, blacklisting, and more.

## Goals

* Build a portable base firewall that can be used as a formidable starting point for new droplets
* Proactive defense, packet cleansing, traffic shaping
* Traffic queing
* Redundancy
* Implement logging and monitoring measures

## Prerequisites

* A Unix-like workstation (BSD, Linux, or Mac)
* A [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) that only permits inbound SSH traffic (refer to this [quickstart guide](https://www.digitalocean.com/docs/networking/firewalls/quickstart))
* An [SSH keypair](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account) with a copy of the public key uploaded to your account
* A standard 1G FreeBSD 12.01 droplet in the region of choice, either ZFS or UFS

## Step 1 - Create the droplet

Create a standard 1G FreeBSD 12.01 droplet in the region of choice from the [control panel](https://cloud.digitalocean.com/login) in your account. The filesystem is irrelevant, it can be ZFS or UFS. Give it a few minutes to finish.

## Step 2 - Attach the droplet to a cloud-firewall

Upon creating our droplet we have no firewall enabled, therefore we risk being insecurely exposed to the open internet. For this reason we attach the droplet to our [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) immediately before doing anything else. Later, after building our [PF](https://man.openbsd.org/pf) ruleset we will detach the droplet from the [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart).

## Step 3 - SSH into the droplet

Obtain the ip address of the droplet from the [control panel](https://cloud.digitalocean.com/login) and SSH into it:
```command
ssh root@XXX.XXX.XX.XXX
```

## Step 3 - Begin the base ruleset

Let's begin drafting our base ruleset without enabling PF. There are two approaches to building a firewall ruleset: 1) default deny, or 2) default permit. The default deny approach blocks all traffic, and will not permit any traffic unless it is specified in a rule. The default permit approach does the exact opposite. It passes all traffic, and will not block any traffic unless it is specified in a rule. In this tutorial we will take the default deny approach.

[PF](https://man.openbsd.org/pf) rulesets are written in a configuration file named `pf.conf`, which we have to create in the default location `/etc/pf.conf`. Storing it at a different location is perfectly fine, however, that location will have to be specified in `/etc/rc.conf`. One alluring aspect of PF is that rulesets are nothing more than text files, which greatly simplifies writing and editing them. Loading a ruleset is achieved with a single command, and there are no delicate procedures required to load new rulesets. With PF's masterful design, the state of the firewall is irrelevant. Simply load the new ruleset and the old one is gone.

Create `/etc/pf.conf`:
```super_user
vim /etc/pf.conf
```

[PF](https://man.openbsd.org/pf) filters packets with any of the following three actions: **block**, **pass**, or **match**. Actions are taken when the contents of the packet headers meet the criteria of the rules we write, and the parameters we specify.

We begin by blocking everything:
```
block all
```

This rule blocks all forms of traffic in either direction. Our **block** rule does not specify an **in** or **out** direction, therefore it defaults to both. The same logic applies to the **pass** and **match** actions.

Our first rule is a perfectly legitimate for a local workstation that needs to be insulated from the world, however, it is impractical, and it will not work with a remote droplet. Can you guess why? The reason for this is that it does not permit **SSH** traffic. If we had loaded this rule when [PF](https://man.openbsd.org/pf) was enabled, it would have locked us out of our droplet. This is something to always keep in mind with remote machines. Every administrator has probably done this at least once!

Let's allow **SSH** access:
```
pass in proto tcp to port 22
```

Alternatively, we can use the name of the protocol:
```
pass in proto tcp to port ssh
```

Using port numbers or protocol names is largely a choice of style. For the sake of consistency we will use port numbers, unless there is a valid reason to do otherwise. There is a detailed list of protocols and their respective port numbers in the `/etc/services` file, which you are highly encouraged to view.

PF rulesets are processed from top-to-bottom, therefore our ruleset initially blocks all forms of traffic, and then passes it if the criteria on the next line is met, which in this case is **SSH** traffic.

Our current ruleset allows us to **SSH** into our droplet, but it still blocks all forms of outgoing traffic. This prevents us from accessing some critical services from the internet. To address this we add a **pass out** rule:
```
pass out proto tcp to port { 22 53 80 123 443 }
```

Now our ruleset allows **SSH, HTTP, NTP, and HTTPS** traffic to pass in the outward direction. Notice we placed the port numbers inside of the curly brackets. This is a **list** in [PF](https://man.openbsd.org/pf) syntax, which is an enormously convenient feature designed to accomodate multiple parameters. Our preliminary ruleset now looks like this:
```
set skip on lo0
block all
pass in proto tcp to port { 22 }
pass out proto tcp to port { 22 53 80 123 443 }
pass out proto udp to port { 53 123 }
pass out inet proto icmp icmp-type echoreq
```

Notice we added some new rules. The `set skip` rule prevents us from filtering traffic on our loopback device, which is unnecessary. The **pass out** rule for **UDP** is for ports **53** and **123**, which will provide DNS and NTP services. Both DNS and NTP will occasionally toggle between the TCP and UDP protocols, hence the two separate rules. Finally, the last **pass out inet** rule for the **ICMP** messaging protocol will allow us to use [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) for troubleshooting purposes. The [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) command utilizes the *echo messages* of the ICMP protocol.

ICMP is an often controversial protocol, however, as long as it is approached with care there is no harm in using it. For our purposes we just want to use [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports), that's it. In our **pass inet** rule above we added *only* the `echoreq` parameter, therefore that will be the only type of message allowed to pass. The *echoreq* parameter is the actual message code name from the [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) protocol. [PF](https://man.openbsd.org/pf) allows us to incorporate these codes directly in our rulesets. 

Moving forward, we can now perform some essential tasks from inside of our droplet such as install packages, sync with an NTP server, transfer files with SCP, and more. By default, [PF](https://man.openbsd.org/pf) filters traffic statefully, meaning that it stores information about connections in what is known as the *state table*. Therefore if we establish any connections with the ports in our **pass out** rule, [PF](https://man.openbsd.org/pf) knows when incoming packets are affiliated with that connection. This speeds up the process because those packets are inspected against the *state table* when they arrive, as opposed to the entire rulesete. This also ensures that our connections are secure.

## Step 4 - Test our preliminary rules

Although we are far from finished, we have a working ruleset that provides basic protection. That said, let's test it on our droplet during these early stages so that we can detect any errors early on. We need to enable [PF](https://man.openbsd.org/pf), detach the droplet from our current [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart), reboot the droplet, and run some sanity checks.

Enable [PF](https://man.openbsd.org/pf) by modifying `/etc/rc.conf`:
```super_user
sysrc pf_enable="YES"
sysrc pflog_enable="YES"
```

Start [PF](https://man.openbsd.org/pf):
```super_user
service start pf
service start pf_log
```

Load the ruleset with [pfctl](https://man.openbsd.org/pfctl), the built-in command-line utility for [PF](https://man.openbsd.org/pf).
```super_user
pfctl -f /etc/pf.conf
```

If there are no messages, we are good-to-go!

Now we can let [PF](https://man.openbsd.org/pf) takeover by detaching the droplet from our current [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart). Go ahead and do that from the [control panel](https://cloud.digitalocean.com/login) in your account. Give it a few minutes to finish.

Reboot the droplet:
```super_user
reboot
```

The connection to our droplet will be dropped. Give it a few minutes to finish. Now [PF](https://man.openbsd.org/pf) is in charge!

Let's perform a few sanity checks to ensure that everything is working. 

SSH back into the droplet:
```command
ssh root@XXX.XXX.XX.XXX
```

Reload the ruleset:
```super_user
pfctl -f /etc/pf.conf
```

There should be no errors or messages.

Use [pfctl](https://man.openbsd.org/pfctl) to check out some stats:
```super_user
pfctl -si
```

Test internet connectivity with `ping`:
```super_user
ping 8.8.8.8
```

Test DNS by updating the **pkgs** repository:
```super_user
pkg upgrade
```

If any packages need to be upgraded, go ahead and do that. If all of these services are working, our ruleset is working, and we can proceed.

## Step 5 - Completing our base ruleset

### Incorporate macros and tables

In our preliminary ruleset above we hard coded all of our parameters into our rules. This works for now, but will likely become unmanageable as our ruleset expands. For organizational purposes [PF](https://man.openbsd.org/pf) offers **macros**, **lists**, and **tables**. We already used lists in some of our **pass out rules**, which were the port numbers that we placed inside of the curly brackets. Macros allow us to declare variables that are defined by several parameters, and tables do the same for groups of IP addresses. 

Let's begin by moving all of our parameters into macros.

Add the following to the top of the ruleset:
```
ext_if = "vtnet0"
tcp_services= "{ 22 53 80 123 443 }"
udp_services= "{ 53 123 }"
icmp_messages= "{ echoreq }"
```

Now we can add the variables to our rules, making our ruleset significantly easier read. Notice we also added a macro for our network interface **vtnet0**, which is the default interface on a singleton Digital Ocean droplet.

Next we introduce a table that contains a common group of IP addresses that are non-routable, and have no business traveling through the internet. Non-routable IP addresses often play a role in Denial of Service attacks (DOS) as well. Our network should not send or receive packets that contain these addresses in their headers. The IP addresses in our table below were derived from [RFC6890](https://tools.ietf.org/html/rfc6890), which documents special-purpose IP address registries, hence the name.

Add the following table below the macros:
```
table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }
```

### Traffic normalization

Traffic normalization is a broad term that refers to the mitigation of various packet discrepancies. Packets often arrive fragmented, with erroneous TCP flag combinations, or with bogus IP addresses (spoofed). Some of these issues originate from malicious actors, others are a result of the many implementations of the TCP/IP protocol suite. Fragmented packets are the source of myriad security exploits.

### Scrubbing

We use the **scrub** keyword will cleanup our fragmentation issues:
```
scrub in
```

The above rule applies to all incoming traffic. It will re-assemble fragmented packets, discard any cruft, and drop packets that have erroneous TCP flag combinations.

Please be aware that some applications require special attention in regards to normalization. Using an NFS filesystem for example may require that the `no-df` and `random-id` options are set because some operating systems will automatically set the **dont-fragment** bit. When in doubt, research all of the **scrub** options.

### Antispoofing

Spoofing is the practice of using fake IP addresses to disguise the real host address, typically for malicious purposes. [PF](https://man.openbsd.org/pf) knows when certain IP addresses are traveling in directionsr that are impossible.

Add the following rule:
```
antispoof quick for $ext_if
```

### Blocking non-routable IP addresses

We can block all of the IP addresses from our **rfc6890** table:
```
block in quick on egress from <rfc6890> to any
block return out quick on egress from any to <rfc6890>
```

Here we introduce the **quick** keyword, which tells [PF](https://man.openbsd.org/pf) to execute the rule without any further processing of the ruleset. This is desirable for handling non-routable and spoofed IP addresses because we want those packets to be discarded immediately. We also introduce the **return** option for our **block** rule, which blocks outbound packets with a source address from <rfc6890>, but also sends an RST message to the host that made this attempt. Finally, we introduce the **egress** keyword on our **block return out** rule, which automatically finds the default route. This is often a cleaner method of finding the default route, especially on larger networks.

## Step 6 - Summarize our progress

We have a complete base ruleset that can serve as a formidable starting point for any new droplet. Our ruleset can easily be expanded, and our macros, lists, and tables enable us to easily incorporate additional parameters. Currently, we are only allowing **SSH** traffic on port 22 into our machine. In the future, if we need to open up some more ports, we simply add them into the list in our **pass in** rule.

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
pass in proto tcp to port { 22 }
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass out inet proto icmp icmp-type $icmp4_messages
```

Now it's time to test our ruleset, and make a copy of it.

Use the `-nvf` flags to `pfctl` to take a dry run:
```super_user
pfctl -nvf /etc/pf.conf

If there are no errors, the ruleset is working. This also gives us a view of what our ruleset really looks like. [PF](https://man.openbsd.org/pf) allows us to write shortcuts for its rules, and now we can see the rules in their full form. This is a good way to see what's going on underneath the hood.

Load the ruleset:
```super_user
pfctl -f /etc/pf.conf
```

Make a copy of ruleset:
```
cp /etc/pf.conf /etc/pf.conf.orig
```

Test internet connectivity with `ping`:
```super_user
ping 8.8.8.8
```

Test DNS by updating the **pkgs** repository:
```super_user
pkg upgrade
```

If there are any packages to update, go ahead and do that.


## Step 6 - Introduce anchors

[PF](https://man.openbsd.org/pf) provides **anchors**, which are defined as *containers* that hold rules, tables, and even other anchors. Using an **anchor** is a clever way to source rules into the main ruleset, either on-the-fly with `pfctl`, or with an external text file. Let's demonstrate this using our <rf6890> addresss table. Instead of placing our table at the top of our ruleset, we'll place it in an external file, along with the corresponding filtering rules.

Create the file **/etc/non_routes**:
```super_user
vim /etc/non_routes
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

The above code is taken directly from our main ruleset. We can now remove that code from the main ruleset, except for the $ext_if macro, which needs to exist in both. Now these rules can be called from an anchor. We'll name the anchor rfc6890, however it can be given any name. We already have a copy of the original ruleset, so we will modify `/etc.pf.conf` directly.

Our `/etc/pf.conf/ now looks like this:

```
*--snip--*
set skip on lo0
scrub in
antispoof quick for $ext_if
anchor rfc6890
load anchor rfc6890 from "/etc/non_routes"
block all
*--snip--*
```

Reload the ruleset with the verbose flag:
```super_user
pfctl -vf /etc/pf.conf
```

Make a copy of that:
```super_user
cp /etc/pf.conf /etc/pf.conf.anchor1
```

One really intelligent feature of anchors is their ability to be used on-the-fly without reloading the main ruleset. This is useful for testing, dealing with a rogue host, and many other scenarios. For example we can an anchor *below* our **block all** rule.
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

Now we can implement rules on-the-fly using the anchor name with the `pfctl` utility from the command-line, without reloading the ruleset. For example we can block all traffic from a remote host that it acting strange.

Block a remote host:
```super_user
echo "block in quick from XXX.XXX.XX.X" | pfctl -a rogue_host -f -
```

## Step 7 - Defending against cryptography exploits

Cryptography exploits, such as brute force or key search attacks, occur when attackers systematically decrypt passwords in order to access the system. These are often performed with a great deal of sophistication and effort, and usually succeed if the target machine uses weak passwords. In the strongest terms, we recommend using public-key authentication for accessing a droplet, which will exponentially reduce the risks of a cryptography exploit. However, even with public-key authentication, attackers or netbots can still be a nuisance, and key search attacks are not impossible. In addition to public-key authentication, we can use [PF](https://man.openbsd.org/pf)'s state monitoring capabilities to detect and deny forced entry attempts. We can also ban the attackers IP addresses from the network. 

### Banning malicious IPs with PF's overload table

Create an empty table:

table <blackhats> persist

The **persist** keyword allows us to maintain empty tables. Don't worry about it too much for now, just know that it needs to be there. Next we modify our **pass in** rule for SSH traffic so that the number of simultaneous, or rapid, connections it receives is closely monitored. For example if port 22 receives 100 login attempts in 10 seconds, it is likely a brute force attack.

Modify the rule:
```
pass in proto tcp to port { 22 } \
    keep state (max-src-conn 10, max-src-conn-rate 6/3, \
        overload <blackhats> flush global)
```

These are nothing more than a few of [PF](https://man.openbsd.org/pf)'s stateful tracking options. The **max-src-conn 10** and **max-src-conn-rate 6/3** options drastically reduce the number of connections that can be made by a single host. If these options are exceeded, the **overload** option will perform its magic by uploading the source address of the attacker to the <blackhats> table. The **flush global** option will immediately drop the connection. We then create attacker's IP address is black-listed from the network until we remove it from the <blackhats> table. Refer to the [man pages](https://man.openbsd.org/pf.conf.5#STATEFUL_FILTERING) to better understand these options.

Over time the <blackhats> table might grow considerably over time. We can periodically clear the <blackhats> table with `pfctl` to avoid wasting memory. The above command will delete addresses from the <blackhats> table that have been in the table for 48 hours (172800 seconds).

Clear the <blackhats> table:
```super_user
pfctl -t blackhats -T expire 172800
```

We can also automate this with **cron**, FreeBSD's job scheduler, which uses shell scripts to run **cron jobs**. These are configured in what are known as **crontabs**, which can be thought of as configuration files. Let's run the `pfctl` command everday at midnight with a **cron job**.

Create a shell script:
```super_user
vim /usr/local/bin/clear_overload.sh
```

Add the following:
```
\#!/bin/sh

pfctl -t blackhats -T expire 172800
```

Make the file executable:
```super_user
chmod 755 /usr/local/bin/clear_overload.sh
```

Create a sudo user crontab:
```command
sudo crontab -e
```

Add the following:
```
\# minute	hour	mday	month	wday	command
\   *		0	*	*	*	/usr/local/bin/clear_overflow.sh


Utilizing [PF](https://man.openbsd.org/pf)'s stateful tracking capabilities for mitigating hostile connections is a fine-tuned approach because it is based on [PF](https://man.openbsd.org/pf)'s state table entries, which are extremely robust. 

## Step 8 - Monitoring and logging

Our firewall is of little use to us if we can't see what it's doing. Since [PF](https://man.openbsd.org/pf) filters packets at the kernel-level, we have a number options. Here we will discuss the essentials of monitoring and logging.



### Misc links

[bandwidth billing](https://www.digitalocean.com/docs/accounts/billing/bandwidth)






