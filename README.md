# How to Configure PF on a FreeBSD 12.0 Digital Ocean Droplet

### Introduction

Networks are hostile places. Every organization will face this reality unless they are completely offline and have no internal threats. The firewall is arguably one of the most important lines of defense against cyber attacks. The ability to configure a firewall from scratch is an empowering skill that will enable the administrator to take control of their networks.

In this tutorial we build a firewall from the ground up on a FreeBSD 12.0 droplet with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), a renown packet filtering tool that is maintained by the [OpenBSD](https://www.openbsd.org) project. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) is part of the FreeBSD base system, and is supported by a community of dedicated users. It is known for its simple-syntax, user-friendliness, and extensive power. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) is a stateful firewall by default that stores information about connections in a *state table*.

## Prerequisites

* A Unix or Unix-like workstation (BSD, Linux, or Mac)
* A [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) that permits inbound SSH traffic. Refer to this [quickstart guide](https://www.digitalocean.com/docs/networking/firewalls/quickstart).
* An [SSH keypair](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account) with a copy of the public key uploaded to your account
* A standard 1G FreeBSD 12.0 droplet in the region of choice. 

## Step 1 - Prepare the droplet

We begin by spinning up a fresh droplet and performing some essential configuration tasks on it.

### Choose your droplet

Create a standard 1G FreeBSD 12.0 droplet in the region of choice from the [control panel](https://cloud.digitalocean.com/login) in your account. The filesystem is irrelevant, it can be ZFS or UFS. Your droplet should be configured with a **sudo** user, and proper *SSH* access. *Public-key authentication* is strongly recommended.

### Attach the droplet to a cloud-firewall

Upon creating our droplet we have no firewall enabled, therefore we risk being insecurely exposed to the open internet. Attach the droplet to a [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) before doing anything else. Later, after enabling [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), we will detach the droplet from the [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart).

### SSH into the droplet

```command
ssh sudouser@XXX.XXX.XX.XXX
```

You should now be inside of the droplet. Feel free to have a look around and familiarize yourself with the directory tree. We are now ready to start building our [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) ruleset.

## Step 2 - Build the preliminary ruleset

In this step we lay the foundation for our firewall with some preliminary rules. There are two approaches to building a ruleset: 1) default deny, and 2) default permit. The *default deny* approach blocks all traffic, and will not permit any traffic unless it is specified in a rule. The *default permit* approach passes all traffic, and will not block any traffic unless it is specified in a rule. We will use the default deny approach.

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) rulesets are written in a configuration file named `/etc/pf.conf`, which is the default location. It is okay to store it somewhere else, but you will have to specify that in `/etc/rc.conf`. We will use the default location.

Create `/etc/pf.conf`:
```command
sudo vim /etc/pf.conf
```

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) filters packets according to three core actions: `block`, `pass`, and `match`. When combined with other options they form rules. An action is taken when a packet meets the criteria that we specify in those rules. As you may expect, `pass` and `block` will `pass` and `block` traffic. A `match` rule performs an action on a matching packet without passing or blocking it. For example we can perform *network address translation (NAT)* on a specific packet (a match) without actually moving it anywhere.

Add the first rule to `/etc/pf.conf`:
```
block all
```

This rule blocks all forms of traffic in all directions. It does not specify an `in` or `out` direction, therefore it defaults to both. The same logic applies to `pass` and `match` rules when no direction is given. 

This rule is perfectly legitimate for a local workstation that needs to be completely insulated from the world. However, it is largely impractical, and will not work with a remote droplet. Can you guess why? The reason is because we did not permit any *SSH* traffic. If [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) had been enabled we would have locked ourselves out of our droplet. This is something to always keep in mind with remote machines. Every administrator has probably done this at least once! Let's revise the previous rule.

Revise the previous rule:
```
block all
pass in proto tcp to port 22
```

Alternatively, we can use the name of the protocol:
```
block all
pass in proto tcp to port ssh
```

For the sake of consistency we will use port numbers, unless there is a reason to do otherwise. There is a detailed list of protocols and their respective port numbers in the `/etc/services` file, which you are highly encouraged to look at.

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) processes rules sequentially from top-to-bottom, therefore our ruleset initially blocks all traffic, but then passes it if the criteria on the next line is matched, which in this case is *SSH* traffic.

We can now *SSH* into our droplet, but we are still blocking all forms of outgoing traffic. This will become problematic because we cannot access some critical services from the internet, such as software repositories, time servers, etc.

Let's allow some outward traffic:
```
block all
pass out proto tcp to port { 22 53 80 123 443 }
pass out proto udp to port { 53 123 }
```

Our ruleset now permits outward *SSH*, *HTTP*, *DNS*, *NTP*, and *HTTPS* traffic, and blocks all inward traffic, except SSH. We placed the port numbers inside of the curly brackets, which forms a *list* in [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) syntax. We also added a *pass out* rule for the *UDP* protocol on ports 53 and 123 because DNS and NTP services sometimes toggle between the TCP and UDP protocols. We are almost finished with the preliminary ruleset, but we still need to add some final touches.

Complete the preliminary ruleset:
```
[label Preliminary Ruleset]
set skip on lo0
block all
pass in proto tcp to port { 22 } tag SSH
pass out proto tcp to port { 22 53 80 123 443 }
pass out proto udp to port { 53 123 }
pass out inet proto icmp icmp-type { echoreq }
```

We created a `set skip` rule for our loopback device because it does not need to filter traffic. We added a *tag* on our *SSH* rule which will mark packets with the string "SSH" if they pass this rule. Finally, we added a *pass out inet* rule for the *ICMP* messaging protocol, which allows us to use the [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) utility for troubleshooting purposes. The *inet* option represents the *IPv4* address family.

*ICMP* is an often controversial protocol that is fraught with many assumptions. As long as it is approached with care there is no harm in using it. We just want to use [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) with a droplet, that's it. The [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) utility relies on [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) *echo messages*. Therefore in our `pass out inet` rule we only permit messages of type *echoreq*. Every other message type will be blocked in all directions. If we need more message types in the future, we can easily add them to our rule. You may have noticed that [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) allowed us to incorporate these codes directly from the [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) protocol. Yes it did! See `man icmp` for additional codes.

We now have a working ruleset that provides basic protection and functionality. Let's test it before we get too far. 

## Step 3 - Test the preliminary ruleset

In this step we make the transition from our [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) to our [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) firewall, allowing [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) to completely take over. We will load the ruleset with the *[pfctl](https://www.freebsd.org/cgi/man.cgi?query=pfctl&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html)* utility, which is [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html)'s built-in command-line tool which you should get familiar with. 

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) rulesets are nothing more than text files, which greatly simplifies the process. There are no delicate procedures involved with loading a new ruleset. Simply load the new one, and the old one is gone. There is rarely ever a need to flush an existing ruleset, and it is generally considered a bad idea.

Enable [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html):
```command
sudo sysrc pf_enable="YES"
sudo sysrc pflog_enable="YES"
```

Reboot the droplet:
```command
sudo reboot
```

The connection will be dropped. Give it a few minutes to update.

SSH back into the droplet:
```command
ssh sudouser@XXX.XXX.XX.XXX
```

Load the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

If there are no errors or messages, we're good-to-go!

Now detach the droplet from the [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) at the [control panel](https://cloud.digitalocean.com/login) in your account. You will remove it from the list of droplets in your [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) interface.

Reboot the droplet again:
```command
sudo reboot
```

The connection will be dropped. Give it a few minutes to update.

SSH back into the droplet:
```command
ssh sudouser@XXX.XXX.XX.XXX
```

View some state table information:
```command
sudo pfctl -si
```

Test internet connectivity:
```command
ping 8.8.8.8
```

Test DNS by upgrading the *pkgs* repository:
```command
sudo pkg upgrade
```

If there are any packages to upgrade, upgrade them. If all of these services are working, that means our ruleset is working, and we can proceed. 

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) is now the acting firewall for our droplet. However, our preliminary ruleset is missing many of the perks that [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) has to offer. Let's incorporate some more of [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html)'s core features until we have a base ruleset that can serve as a formidable starting point for all of our droplets.

## Step 4 - Build a base ruleset

In this step we build our base ruleset, which is the most important task of this tutorial. Before we begin, it is important to understand that rulesets in [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) are structured in a particular sequence: 1) macros, 2) tables, 3) options, 4) normalization, 5) queuing, 6) translation, and 7) packet filtering. If you find yourself getting confused in the order of things, refer to the sample rulesets either at step 5, or at the end of the tutorial.

### Incorporate macros and tables

In the preliminary ruleset we hard coded all of our parameters into each rule, i.e., the port numbers that make up the lists. This will likely become unmanageable and difficult to read as our ruleset expands. For organizational purposes [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) offers *macros*, *lists*, and *tables*. As mentioned, we already used *lists* in the preliminary ruleset. We can also assign a list, or even a single value, to a variable using macros. Let's move our parameters into macros at the top of the ruleset:

```command
sudo vim /etc/pf.conf
```

Add the following rules:
```
ext_if = "vtnet0"
tcp_services= "{ 22 53 80 123 443 }"
udp_services= "{ 53 123 }"
icmp_messages= "{ echoreq }"
```

Now we can call the variable names instead of the parameters in our filtering rules.

Our filtering rules now look like this:
```
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass inet proto icmp icmp-type $icmp4_messages
```

Next we implement a *table*, which is similar to a *macro*, but designed to hold groups of IP addresses. Let's create a table for non-routable IP addresses, which often play a role in *denial of service attacks* (DOS). Our droplet should not send or receive packets to or from these addresses under any circumstances. In our table we use the IP addresses specified in [RFC6890](https://tools.ietf.org/html/rfc6890), which defines special-purpose IP address registries.

Create the table:
```
table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }
```

### Traffic normalization

*Traffic normalization* is a broad term that refers to the handling of various packet discrepancies. Packets often arrive fragmented, with erroneous TCP flag combinations, or with bogus IP addresses (spoofed). Some of these issues originate from malicious actors, others are a by-product of the many implementations of the *TCP/IP protocol* suite.

### Scrubbing

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) provides a `scrub` keyword to handle *packet fragmentation*. It re-assembles fragmented packets, discards any cruft, and also drops packets that have erroneous TCP flag combinations. We want to subject all incoming traffic to a `scrub` rule. 

Add a `scrub` rule:
```
scrub in
```

For most purposes the above rule works fine. Please be aware, however, that some applications require special attention in regards to scrubbing. *NFS* filesystems, for example, may require that the `no-df` and `random-id` options are set because some operating systems automatically set the *dont-fragment* bit. When in doubt, research the `scrub` options.

### Antispoofing

*IP Spoofing* is the practice of using a fake IP address to disguise a real IP address, typically for malicious purposes. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) knows when certain IP addresses are traveling in directions that are impossible. It has a built-in *antispoofing* mechanism for dealing with these packets quickly.

Add antispoofing:
```
antispoof quick for $ext_if
```

We also introduce the `quick` keyword, which executes a rule without doing any further processing of the ruleset. This is desirable for handling false IP addresses because we want to drop those packets immediately, and have no reason to evaluate the remainder of the ruleset.

### Blocking non-routable IP addresses

Previously we created the `table <rfc6890>`, but did not create any rules against it.

Create rules against `table <rfc6890>`:
```
block in quick on egress from <rfc6890>
block return out quick on egress to <rfc6890>
```

Here we introduce the `return` option, which compliments our `block out` rule. This will drop the packets, and also send an *RST messages* to the host that tried to make these connections. Finally, we introduce the `egress` keyword, which automatically finds the default route. This is usually a cleaner method of finding the default route, especially on complex networks.

That completes our base ruleset. Let's put all of the pieces together and review the final product before introducing additional topics.

## Step 5 - Review and test the base ruleset

In this step we review our base ruleset and test it to make sure it works. It is important that we fully understand it before covering any new territory.

Here is the complete base ruleset:
```
[label Base Ruleset]
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

Our base ruleset provides us with:

- A collection of macros that define key services
- Network hygiene policies to address packet fragmentation and illogical IP addresses
- A *default deny* filtering structure that blocks everything, and only permits what we specify
- Inward *SSH* access 
- Access to some critical services from the internet
- Access to the [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) utility for troubleshooting purposes

Be sure that your `/etc/pf.conf` is identical to the complete base ruleset above. After that we can test our packet filter.

Take a test run:
```command
sudo pfctl -nf /etc/pf.conf 
```

There should be no errors.

The `-nf` flags tell `pfctl` to run the ruleset without loading it, and will throw errors if anything is wrong.

Use the verbose option:
```command
sudo pfctl -nvf /etc/pf.conf 
```

The verbose option will print the rules in their full-form, which is what they actually look like. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) lets us write shortcut versions of rules for readability, which is how we have written them thus far. Viewing the rules in their full-form is a good way to learn the [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) syntax.

Load the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

If there are no errors, we are good-to-go!

Test internet connectivity:
```command
ping 8.8.8.8
```

Test DNS by upgrading the `pkgs` repository:
```command
sudo pkg upgrade
```

If there are any packages to upgrade, upgrade them.

Reboot the droplet:
```command
sudo reboot
```

The connection will be dropped. Give it a few minutes to update.

## Step 6 - Introduce anchors (optional)

In this step we introduc *anchors*, which are used for sourcing rules into the main ruleset, either on-the-fly with `pfctl`, or from an external text file. Anchors can contain rule snippets, tables, and even other anchors, or *nested anchors*. Let's demonstrate this by adding a table to an external file, and sourcing it into the ruleset with an anchor. Our table will include a group of internal hosts that we want to prevent from making any connections to the outside world.

Create a file named `/etc/punished`:
```command
sudo vim /etc/banned
```

Add the following:
```
table <punished> { 192.168.47.1 192.168.47.2 192.168.47.3 }

block return out quick on egress to <punished>
```

Add the anchor to the ruleset:
```command
sudo vim /etc/pf.conf
```

Add the following:
```
*--snip--*
block all
anchor punished_anchor
load anchor banned_anchor from "/etc/punished"
*--snip--*
```

Load the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Verify the anchor is running:
```command
sudo pfctl -s Anchors
```

Show the anchor rules:
```command
sudo pfctl -a banned_anchor -s rules
```

View the anchor in the ruleset:
```command
sudo pfctl -s rules
```

Another intelligent feature of *anchors* is their ability to accept rules on-the-fly without having to reload the ruleset. This can be useful for testing, quick-fixes, emergencies, etc. For example, let's say an internal host is acting strange and we want to block it. We can add an anchor to our ruleset and block it manually from the command-line. We'll name the anchor `rogue_hosts`. Once the anchor is in place, we can use it anytime. 

```command
sudo vim /etc/pf.conf
```

Add the anchor `rogue_hosts`:
```
*--snip--*
block all
anchor rogue_hosts
*--snip--*
```

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Verify the anchor is running:
```command
sudo pfctl -s Anchors
```

View the anchor in the ruleset:
```command
sudo pfctl -s rules
```

Now we add a rule that blocks the host:
```command
sudo sh -c 'echo "block return out quick on egress from 192.168.47.4" | pfctl -a rogue_hosts -f -'
```

Show the anchor rules:
```command
sudo pfctl -a rogue_hosts -s rules
```

You should see this:
```
block return out quick on egress inet from 192.168.47.4 to any
```

You should now understand the concept of *anchors*, and hopefully have some ideas on how to integrate them into your network. Whether or not to use anchors is completely situational. Just like any other feature there are pros and cons to using them. Occasionally they are necessary for external applications that are designed to integrate with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html).

## Step 7 - Defending against cryptography exploits (optional)

In this step we introduce some defense strategies for dealing with cryptography exploits, such as *brute force* or *key search attacks*. Thankfully [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) makes this easy for us. These types of exploits occur when the attacker systematically decrypts passwords to access a system. They usually succeed if the target machine has weak passwords. We strongly recommend public-key authentication on all droplets, however, even with these measures, attackers and netbots can still be a nuisance, and key search attacks are not impossible. 

Coupled with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html)'s state monitoring capabilities, the *overload mechanism* enables us to detect and ban suspicious IP addesses from the system, based on their behavior. This is an adaptive and automated approach to handling unruly hosts, which can be more practical than hard coding IP addresses into a table.

### Banning IP addresses with the overload mechanism

```command
sudo vim /etc/pf.conf
```

Add an overload table:
```
table <blackhats> persist
```

Modify our SSH rule:
```
pass in proto tcp to port { 22 } \
    keep state (max-src-conn 15, max-src-conn-rate 3/1, \
        overload <blackhats> flush global) tag SSH
```

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

The `persist` keyword allows us to maintain empty tables. Next we use some of [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html)'s `keep state` options as the criteria for the `overload` mechanism. The idea is to significantly reduce the frequency at which external hosts are allowed to make connections to our droplet. For example, if port 22 receives 100 failed logins in 10 seconds, it is likely a *brute force attack*, and the source IP address should be banned.

The `max-src-conn 15` option permits only 15 simultaneous connections from a single host. The `max-src-conn-rate 3/1` option will only allow 3 new connections per second from a single host. If these options are exceeded, the `overload` mechanism adds the source address to the `<blackhats>` table, where it will remain until we remove it. Finally, the `flush global` option immediately drops the connection.

We have enforced strict measures on our SSH traffic for good reason, however, these are not suitable for all services. Increased connections are not always attackers. They can be random web connections that were dynamically assigned IP addresses. Blocking them may or may not be the right solution, depending on the circumstances. Services that can afford liberal connection policies should not share the same `overload` mechanism with services that cannot.

Over time the `<blackhats>` table will grow and should be periodically cleaned out. It is unlikely that an attacker will continue using the same IP address, so it is nonsensical to store it in the table for long periods of time. We can clear the table manually with `pfctl`, or automate the process with *cron*, FreeBSD's *job scheduler*.

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

This *cron job* runs the `clear_overload.sh` script everyday at midnight, removing IP addresses that are 48 hours old from the *overload table* `<blackhats>`.

We now have a mechanism in place that will safegaurd our *SSH* traffic. Notice how simple it is to defend ourselves against such a sophisticated method of intrusion. That is the essence of [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html).

## Step 8 - Logging and Monitoring

In this step we introduce *logging* and *monitoring*. Our firewall is of little use if we cannot see what it is doing. Policy decisions in network security are highly dependent on packet analyses, which inevitably invlove examining log files. With [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), logging occurs on a psuedo-device known as the *pflog interface*. Since it is an interface, various userland tools can be used to access its logs. 

To create logs with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), we add the `log` keyword to any of our filtering rules. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) then makes copies of the packet headers that match those rules, and writes them to `/var/log/pflog` in *binary format*. The `log` keyword was already added to our *SSH* rule in the base ruleset at step 5. Rarely ever do we need to log everything. That would become unwieldy, and would also waste memory. When creating log files we start with what's most important to us, and expand as needed.

### Create additional log interfaces

If multiple log interfaces are needed we can create them with a `/etc/hostname` file, just as we would with any other interface.

Create a `pflog1` interface:
```command
sudo vim /etc/hostname.pflog1 
```

Add the following:
```
up
``` 

Enable it in `/etc/rc.conf`: 
```command
sudo sysrc pflog1_enable="YES"
```

### Accessing log data with tcpdump

There are multiple ways to access log data, but few tools are capable of generating the level of detail that can be achieved with *[tcpdump(8)](https://man.openbsd.org/tcpdump.8)*, which is part of the FreeBSD base system. 

Obtain log data with `tcpdump`:
```command
sudo tcpdump -ner /var/log/pflog
```

Save log data to a file:
```command
sudo tcpdump -ner /var/log/pflog > ssh_log
```

We can also view logs in real-time from the *pflog0 interface*.

Initiate real-time logging:
```command
sudo tcpdump -nei pflog0
```

There is a wealth of information on the internet about [tcpdump(8)](https://man.openbsd.org/tcpdump.8), including the official [website](https://www.tcpdump.org), which is extremely well documented. The [tcpdump(8)](https://man.openbsd.org/tcpdump.8) utility is one of the best packet analysis tools in the industry. 

### Accecssing log files with pftop

The *pftop utility* is an excellent tool for quickly viewing active states and rules in real-time. It is named after the well known *top utility* because of its similarities.

Install `pftop`:
```command
sudo pkg install pftop
```

Run `pftop`:
```command
sudo pftop
```

It should display something like this:
```
PR    DIR SRC                   DEST                           STATE                AGE       EXP   PKTS BYTES
tcp   In  251.155.237.90:27537  157.225.173.58:22     ESTABLISHED:ESTABLISHED  00:12:35  23:59:55   1890  265K
tcp   In  222.186.42.15:25884   157.225.173.58:22       TIME_WAIT:TIME_WAIT    00:01:25  00:00:06     22  3801
udp   Out 157.245.171.59:4699   67.203.62.5:53           MULTIPLE:SINGLE       00:00:14  00:00:16      2   227
```

### Creating graphical representations of log data with pfstat

Often times we need graphical representations of our packet filter to show clients, colleagues, or maybe the boss. The *pfstat utility* is a lightweight tool that is perfect for this. It populates a `/var/db/pfstat.db` database with data that is parsed from `/dev/pf`. We tell pfstat what to store in the database through a configuration file named `/usr/local/etc/pfstat.conf`. The database can be populated manually from the command-line, or automated with a *cron job*, which is what we will use. The data is then translated into graphs that are illustrated in either *jpg* or *png* image formats. 

Unfortunately, with a single fresh droplet we might not have much data to work with, but we'll try our best! We'll make a simple graph that displays how many packets have been blocked or passed.

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

image "/home/sudouser/pfstat.jpg" {
        from 7 days to now
        width 300 height 200
	left
            graph 1 "pass in" "packets/s" color 0 192 0
	right
            graph 2 "block in" "packets/s" color 255 0 0
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

Give it a couple of seconds to activate. The database should now exist in `/var/db/pfstat.db`. 

```command
ls -l /var/db/pfstat.db
```

The first *cron* entry continuously runs the `pfstat -q` command, which populates the database with information that it queries from `/dev/pf`. The second entry runs the command `pfstat -t 30:180` to periodically clean the database. The colon-separated key-pair values represent numbers of days. The first value represents *uncompressed entries*, and the second represents *compressed entries*. The above rule deletes uncompressed entries that are 30 days old, and compressed entries that are 180 days old. Uncompressed entries are used for high resolution graphs, and should be removed first to keep the size of the database manageable. See `man pfstat`. 

Generate a graph image:
```
sudo pfstat -p
```

There should now be a `pfstat.jpg` image in the home directory. *Xorg* is required to view graphical images. Since we have no intention of installing Xorg, or even forwarding *X sessions* to our droplet, we will copy the image from the droplet to our remote workstation using *scp*.

Exit out of the droplet:
```command
exit
```

Copy the image to your local workstation:
```command 
sudo scp myuser@XXX.XXX.XX.XX:pfstat.jpg /home/myuser
```

Find and view the image on your local workstation. As mentioned, there's a good chance that nothing will appear on the graph because we're using a fresh droplet with no data. If that's the case, give the database time to grow and keep checking. You could also try adding more log rules since we are only logging *SSH* connections in the base ruleset.

We now have the ability to see who is making connections to our droplet, and the types of connections that are being made. Hopefully you are starting to think about how you can implement a logging and monitoring workflow.

## Step 9 - Revert back to the base ruleset

In this final step we quickly revert back to the base ruleset. Throughout this tutorial we introduced some advanced concepts that we probably don't need right away. We want to keep things as simple as possible, implementing advanced features as we need them. That said, let's reload the base ruleset before signing off.

Reload the base ruleset:
```command
sudo vim /etc/pf.conf
```

Add the base ruleset:

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

It never hurts to reboot:
```command
sudo reboot
```

## Conclusion

We now have a portable base ruleset that can serve as a formidable starting point for all of our new FreeBSD droplets. Tuck it way somewhere safe, make copies of it, and think about how you can tailor it to fit your needs. Refer to the sample rulesets below as a reference, and more importantly, the [pf.conf(5)](https://www.freebsd.org/cgi/man.cgi?query=pf.conf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) man pages, which are the ultimate source of authority. Hopefully this tutorial was useful in getting you up to speed with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), and hopefully it provided a glimpse into the world of Unix networking and the *TCP/IP protocol*. See you in the next one!

### Sample ruleset 1 (identical to step 5)
```
[label Base Ruleset]
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
[label Simple Web Server Ruleset]
ext_if = "vtnet0"
tcp_services= "{ 22 53 123 }"
udp_services= "{ 53 123 }"
web_services= "{ 80 443 }"
icmp4_messages= "{ echoreq }"
table <blackhats> persist
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
pass in proto tcp to port { 22 } \
    keep state (max-src-conn 15, max-src-conn-rate 3/1, \
        overload <blackhats> flush global) tag SSH
pass proto tcp to port $web_services
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass inet proto icmp icmp-type $icmp4_messages
```

- Sample rulest 1 is our base ruleset, which is a formidable starting point for fresh droplets that are not yet hosting any public or high traffic services.

- Sample ruleset 2 builds on ruleset 1. It includes some highly restrictive measures on port 22, and permits web traffic on ports 80 and 443.


