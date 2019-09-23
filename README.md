# How to Configure PF on a FreeBSD 12.0 Digital Ocean Droplet

### Introduction

Networks are hostile places. Every organization will face this reality unless they are completely offline and have no internal threats. The firewall is arguably one of the most important lines of defense against cyber attacks. The ability to configure a firewall from scratch is an empowering skill that will enable the administrator to take control of their networks.

In this tutorial we build a firewall from the ground up on a FreeBSD 12.0 droplet with [PF](https://man.openbsd.org/pf), a renown packet filtering tool that is maintained by the [OpenBSD](https://www.openbsd.org) project. [PF](https://man.openbsd.org/pf) is part of the FreeBSD base system, and is supported by a community of dedicated users. It is known for its simple-syntax, user-friendliness, and extensive power. [PF](https://man.openbsd.org/pf) is a stateful firewall by default that stores information about connections in a *state table*.

## Prerequisites

* A Unix-like workstation (BSD, Linux, or Mac)
* A [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) that permits inbound SSH traffic. Refer to this [quickstart guide](https://www.digitalocean.com/docs/networking/firewalls/quickstart).
* An [SSH keypair](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account) with a copy of the public key uploaded to your account
* A standard 1G FreeBSD 12.0 droplet in the region of choice. 

## Step 1 - Create the droplet

Create a standard 1G FreeBSD 12.0 droplet in the region of choice from the [control panel](https://cloud.digitalocean.com/login) in your account. The filesystem is irrelevant, it can be ZFS or UFS. Your droplet should be configured with a **sudo** user, and proper SSH access. Public-key authentication is *strongly* recommended.

## Step 2 - Attach the droplet to a cloud-firewall

Upon creating our droplet we have no firewall enabled, therefore we risk being insecurely exposed to the open internet. Attach the droplet to a [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart) before doing anything else. Later on, after enabling [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), we will detach it from the [cloud-firewall](https://www.digitalocean.com/docs/networking/firewalls/quickstart).

## Step 3 - SSH into the droplet

```command
ssh sudouser@XXX.XXX.XX.XXX
```

## Step 4 - Build the preliminary ruleset

There are two approaches to building a ruleset: 1) default deny, and 2) default permit. The *default deny* approach blocks all traffic, and does not permit any traffic unless it is specified in a rule. The *default permit* approach passes all traffic, and does not block any traffic unless it is specified in a rule. We will use the *default deny* approach.

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) rulesets are written in a configuration file named `/etc/pf.conf`, which is the default location. It is okay to store it somewhere else, but you will have to specify that in `/etc/rc.conf`.

Create `/etc/pf.conf`:
```command
sudo vim /etc/pf.conf
```

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) filters packets according to three core actions: `block`, `pass`, and `match`. These actions are taken when a packet meets the criteria that we specify in our rules. The `pass` and `block` rules, as you may expect, `pass` and `block` traffic. The `match` rule performs an action on a packet without filtering it. For example we could perform *network address translation (NAT)* on a packet without moving it anywhere.

Add the first rule to `/etc/pf.conf`:
```
block all
```

This rule blocks all forms of traffic in all directions. Our `block` rule does not specify an `in` or `out` direction, therefore it defaults to both. The same logic applies to `pass` and `match` actions. 

This rule is perfectly legitimate for a local workstation that needs to be insulated from the world, however, it is largely impractical, and will not work with a remote droplet. Can you guess why? The reason is because we did not permit any *SSH* traffic. If [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) had been enabled we would have been locked out of our droplet. This is something to always keep in mind with remote machines. Every administrator has probably done this at least once! Let's revise the previous rule.

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

For the sake of consistency we will use port numbers, unless there is a reason to do otherwise. There is a detailed list of protocols and their respective port numbers in the `/etc/services` file which you are highly encouraged to look at.

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) processes rules from top-to-bottom, therefore our ruleset initially blocks all forms of traffic, then passes it if the criteria on the next line is matched, which in this case is *SSH* traffic.

We can now *SSH* into our droplet, but we are still blocking all forms of outgoing traffic. This will become problematic becuase we will not be able to access critical services such as installing software, accurate time settings, etc.

Let's allow some outward traffic:
```
block all
pass out proto tcp to port { 22 53 80 123 443 }
pass out proto udp to port { 53 123 }
```

Our ruleset now permits outward *SSH*, *HTTP*, *DNS*, *NTP*, and *HTTPS* traffic, and blocks all inward traffic, except *SSH*. We placed the port numbers inside a pair of curly brackets, which forms a *list* in [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) syntax. We also added the *pass out* rule for the *UDP* protocol on *ports 53 and 123*. That is because *DNS* and *NTP* will sometimes toggle between both protocols. Do some Google searching on this topic. We're almost finished with the preliminary ruleset, but we need to add some final touches.

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

We added the `set skip` rule on our loopback device because it does not need to filter traffic. We also added a *tag* on our *SSH* rule which will label packets with the string *"SSH"* if they pass this rule. Finally, we added a *pass out inet* rule for the *ICMP* messaging protocol, which allows us to use the [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) utility for troubleshooting purposes. The *inet* option represents the *IPv4* address family.

ICMP is an often controversial protocol that is fraught with many assumptions. As long as it is approached with care there is no harm in using it. We just want to use [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) with a droplet, that's it. The [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) utility relies on [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) *echo messages*. Therefore in our *pass out inet* rule we only permit messages of type *echoreq*. Every other type of message will be blocked in all directions. If we need more message types in the future, we can easily add them. You may have noticed that [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) allowed us to incorporate these codes directly from the [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) protocol. Yes it did! See `man icmp` for additional codes.


## Step 5 - Test the preliminary ruleset

We have a working ruleset that can provide basic protection and functionality. Let's test it before we get too far. Rulesets are loaded with the [pfctl](https://www.freebsd.org/cgi/man.cgi?query=pfctl&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) utility, which is a built-in command-line tool. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) rulesets are nothing more than text files, which greatly simplifies the process. There are no delicate procedures involved with loading a new ruleset, simply load it and the old one is gone. There is rarely ever a need to flush an existing ruleset, and it is generally a bad idea.

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

View some state information:
```command
sudo pfctl -si
```

Test internet connectivity:
```command
ping 8.8.8.8
```

Test DNS by updating the *pkgs* repository:
```command
sudo pkg upgrade
```

If any packages need to be upgraded, go ahead and do that. If all of these services are working, that means our ruleset is working and we can now proceed. 

## Step 6 - Complete the base ruleset

Here we will complete the base ruleset. Rules in [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) are structured in a specific sequence: 1) options, 2) normalization, 3) queueing, 4) translation, and 5) filtering. If you find yourself getting confused in the order of things, refer to the sample rulesets at step 7, and at the end of the tutorial.

### Incorporate macros and tables

In our preliminary ruleset we hard coded all of the parameters into each rule, i.e., the port numbers. This will become unmanageable as our ruleset expands. For organizational purposes [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) offers *macros*, *lists*, and *tables*. We already used *lists*, which were the port numbers inside of the curly brackets. We can also assign a *list*, or even a single value, to a variable using *macros*. Let's reorganize our parameters.

Move our parameters into macros at the top of the ruleset:
```
ext_if = "vtnet0"
tcp_services= "{ 22 53 80 123 443 }"
udp_services= "{ 53 123 }"
icmp_messages= "{ echoreq }"
```

Now we can call the variable names in our rules, i.e., `$tcp_services`. 

Our previous rules now look like this:
```
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass inet proto icmp icmp-type $icmp4_messages
```

Next we implement a *table*, which is similar to a *macro*, but designed to hold groups of IP addresses. Let's create a *table* for non-routable IP addresses, which often play a role in *denial of service attacks (DOS)*. Our droplet should not send or receive packets to or from these IP addresses. In our *table* we use the IP addresses specified in [RFC6890](https://tools.ietf.org/html/rfc6890), which defines special-purpose IP address registries.

Add the table:
```
table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }
```

### Traffic normalization

*Traffic normalization* is a broad term that refers to the handling of various packet discrepancies. Packets often arrive fragmented, with erroneous TCP flag combinations, or with bogus IP addresses (spoofed). Some of these issues originate from malicious actors, others are the result of the many implementations of the *TCP/IP protocol* suite.

### Scrubbing

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) provides a `scrub` keyword to handle packet fragmentation. It will re-assemble fragmented packets, discard any cruft, and drop packets that have erroneous TCP flag combinations. We want to subject all incoming traffic to a `scrub` rule. 

Add a `scrub` rule:
```
scrub in
```

For most purposes the above rule works fine. Please be aware, however, that some applications require special attention in regards to scrubbing. NFS filesystems, for example, may require that the `no-df` and `random-id` options are set because some operating systems will automatically set the *dont-fragment* bit. When in doubt, research the `scrub` options.

### Antispoofing

*Spoofing* is the practices of using a fake IP address to disguise a real address, typically for malicious purposes. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) knows when certain IP addresses are traveling in directions that are impossible. It has a built-in *antispoofing* mechanism that deals with these packets quickly.

Add *antispoofing*:
```
antispoof quick for $ext_if
```

We also introduced the `quick` keyword, which executes a rule without doing any further processing of the remainder of the ruleset. This is desirable for handling false IP addresses because we want to drop those packets immediately, and have no reason to process the entire ruleset.

### Blocking non-routable IP addresses

Previously we created the `table <rfc6890>`, but have not created any rules against it.

Create rules against `table <rfc6890>`:
```
block in quick on egress from <rfc6890>
block return out quick on egress to <rfc6890>
```

Here we introduced the `return` option, which compliments our `block out` rule. The `return` option drops the packet, and also sends an *RST message* to the host that tried to make this connection. Finally, we introduced the `egress` keyword, which automatically finds the *default route*. This is usually a cleaner method of finding the *default route*, especially on complex networks.

## Step 7 - Review and test the base ruleset

We now have a complete base ruleset that permits inbound *SSH* traffic, enforces some network hygiene policies, and allows access to some critical services from the internet.

Here is our complete base ruleset:
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

Your `/etc/pf.conf` should now look identical to the ruleset above.

Take a test run:
```command
sudo pfctl -nf /etc/pf.conf 
```

There should be no errors.

This runs the ruleset without loading it, and throws error messages if something is wrong.

Use the verbose option:
```command
sudo pfctl -nvf /etc/pf.conf 
```

This will print the rules in their full-form, which is what they really look like. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) lets us write shortcut versions of rules for readability, which is what we have done thus far. Viewing the rules in their full-form is a good way to learn the [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) syntax.

Load the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Test internet connectivity:
```command
ping 8.8.8.8
```

Test *DNS* by updating the `pkgs` repository:
```command
sudo pkg upgrade
```

If there are any packages to update, go ahead and do that.

Reboot the droplet:
```command
sudo reboot
```

The connection will be dropped. Give it a few minutes to update.

## Step 8 - Introduce anchors (optional)

[PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) *anchors* are for sourcing rules into the main ruleset, either on-the-fly with `pfctl`, or with an external text file. We can demonstrate this with our `table <rf6890>`. Instead of writing it in the main ruleset, we can write it to an external text file, and call it from an anchor. This applies not only to tables, but to any rule snippet.

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
anchor non_routes_anchor
load anchor non_routes_anchor from "/etc/non_routes"
block all
*--snip--*
```

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Reboot the droplet:
```command
sudo reboot
```

The connection will be dropped. Give it a few minutes to update.

SSH back into the droplet:
```command
ssh sudouser@XXX.XXX.XX.XXX

Verify the anchor is running:
```command
sudo pfctl -s Anchors
```

View the anchor in the ruleset:
```command
sudo pfctl -s rules
```

Another intelligent feature of *anchors* is that they allow us to add rules on-the-fly without having to reload the ruleset. This can be useful for testing, quick-fixes, emergencies, etc. For example, if an internal host is acting strange and we want to block it, we can place an anchor below the `block all` rule, and block it from the command-line. Once the *anchor* is in place we can reuse it anytime.

Reload the base ruleset from step 7.

Add an anchor below the `block all` rule:
```
*--snip--*
set skip on lo0
scrub in
antispoof quick for $ext_if
block in quick on $ext_if from <rfc6890> 
block return out quick on egress to <rfc6890>
block all
anchor rogue_hosts
*--snip--*
```

Reload the ruleset:
```command
sudo pfctl -f /etc/pf.conf
```

Reboot the droplet:
```command
sudo reboot
```

The connection will be dropped. Give it a few minutes to update.

SSH back into the droplet:
```command
ssh sudouser@XXX.XXX.XX.XXX

Verify the anchor is running:
```command
sudo pfctl -s Anchors
```

View the anchor in the ruleset:
```command
sudo pfctl -s rules
```

Block a from the command-line:
```command
sudo sh -c 'echo "block return out quick on egress from XXX.XXX.XX.XX" | pfctl -a rogue_hosts -f -'
```

Show the anchor rules:
```command
sudo pfctl -a rogue_hosts -s rules
```

You should see this:
```
block return out quick on egress inet from XXX.XXX.XX.XX to any
```

There are many pros and cons to using *anchors*, all of which are completely situational. *Anchors* can help reduce clutter in a ruleset by breaking it up into separate files, but managing multiple files could be cumbersome and hard to read. *Anchors* can be useful for adding rules on-the-fly, however, someone must be present to add those rules. In other scenarios *anchors* are necessary for external applications that are designed to integrate with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html).

## Step 9 - Defending against cryptography exploits

Cryptography exploits, such as *brute force* or *key search attacks*, occur when attackers systematically decrypt passwords to access the system, usually with bad intentions. They are often performed with a great deal of sophistication, and usually succeed if the target machine has weak passwords. We strongly recommend public-key authentication on all droplets, however, even with these measures, attackers and netbots can be a nuisance, and *key search attacks* are not impossible. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) features an *overload mechanism* that enables us to *blacklist* suspicious IP addesses from the system.

### Blacklisting IP addresses with the overload mechanism (optional)

Create an overload table:
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

The `persist` keyword allows us to maintain empty tables. Next we used some of [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html)'s `keep state` options as the criteria for the `overload` mechanism. The idea is to significantly reduce the frequency at which external hosts are allowed to make connections to our droplet. For example, if port 22 receives 100 failed logins in 10 seconds, it is likely a *brute force attack*, and the source IP address should be *blacklisted*.

The `max-src-conn 15` option permits only 15 simultaneous connections from a single host. The `max-src-conn-rate 3/ option will only allow 3 new connections per second from a single host. If these options are exceeded, the `overload` mechanism adds the source address of the host to the `<blackhats>` table, where it will remain until we remove it. Finally the `flush global` option immediately drops the connection.

We have enforced strict measures on our SSH traffic for good reason, however, these are not suitable for all services. Increased connection attempts are not always attackers. They can be random web connections that were dynamically assigned IP addresses. Blocking them may or may not be the right solution, depending on the circumstances. Services that can afford liberal connection policies should not share the same `overload` mechanism with services that cannot.

Over time the `<blackhats>` table will grow and should be cleared periodically. It is unlikely that an attacker will continue using the same IP address, so it is non-sensical to store it in the table for long periods. We can clear the table manually with `pfctl`, or automate the process with *cron*, FreeBSD's *job scheduler*.

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

## Step 10 - Traffic shaping (optional)

By default, [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) filters packets on a first-come-first-serve basis. For high-traffic applications this may not be optimal. We may want to process certain packets before others, and we can do that with the `prio` option. The `prio` option prioritizes traffic according to a number range of 0 thru 7. Zero is the slowest, and seven is the fastest. Below are some of examles.

### Prioritizing traffic with prio

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

The last example highlights a powerful feature of *prio*, which is the tuple option. The first value prioritizes regular packets, and the second prioritizes *ACK* and/or *lowdelay* TOS packets. TCP connections send *acknowledgement (ACK)* packets which contain no *data payload*. By default, *ACK packets* wait in line with regular packets, which means they have the potential to needlessly clog up the system, and should therefore be processed quickly, with a higher priority than others. Packets may also arrive with a *lowdelay* set in their *type of service* (TOS) fields. Those packets should also be processed before others.

## Step 11 - Monitoring and logging

Our firewall is of little use if we cannot see what it is doing. Policy decisions in network security are highly dependent on packet analyses, which inevitably invlove examining log files. With [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), logging occurs on a psuedo-device known as the *pflog interface*. Since it is an interface, various userland tools can be used to access its logs. 

To create logs with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), we add the `log` keyword to the rules we want to log. [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) then makes copies of the packet headers that match those rules, and writes them to `/var/log/pflog` in *binary format*. Rarely ever do we need to log everything. That would become unwieldy, and would also waste memory. When creating log files we start with what's most important to us, and expand as needed.

### Accessing log data with tcpdump

There are multiple ways to access log data, but few tools are capable of generating the level of detail that can be achieved with [tcpdump(8)](https://man.openbsd.org/tcpdump.8), which is part of the FreeBSD base system. The `log` keyword was already added to our SSH rule in the base ruleset from step 7.

Obtain log data with `tcpdump`:
```command
sudo tcpdump -ner /var/log/pflog
```

Save log data to a file (optional):
```command
sudo tcpdump -ner /var/log/pflog > ssh_log
```

We can also view logs in real-time from the *pflog0 interface*.

Initiate real-time logging:
```command
sudo tcpdump -nei pflog0
```

There is a wealth of information on the internet about `tcpdump`, including the official [website](https://www.tcpdump.org), which is extremely well documented. The `tcpdump` utility is one of the best packet analysis tools in the industry. 

### Accecssing log files with pftop (recommended)

The *pftop utility* is an excellent tool for quickly viewing active states and rules in real-time. It is named after the well known *top utility* because of its similarities.

Install `pftop`:
```command
sudo pkg istall pftop
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

### Creating graphical representations of log data with pfstat (optional)

Often times we need graphical representations of our packet filter to show clients, colleagues, or maybe the boss. The *pfstat utility* is a lightweight tool that is perfect for this. It populates a `/var/db/pfstat.db` database with data that is parsed from `/dev/pf`. The data can then be collected from the database with a *cron job*, or manually from the command-line. We tell *pfstat* what to store in the database through a configuration file named `/usr/local/etc/pfstat.conf`. The data is then translated into graphs that are illustrated in either *jpg* or *png* image formats.

Unfortunately, with a single fresh droplet we don't have much data to work with, but we'll try our best! We'll make a simple graph that shows how many packets have been either passed or blocked.

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

The above entries will populate the database continuously. Since we have automated the process, the database will grow continuously, which is why we added a second rule with the `-t` flag to periodically clean it out. The additional values represent days. The first value represents *uncompressed images*, and the second represents *compressed images*. The above rule deletes *uncompressed images* that are 30 days old, and *compressed images* that are 180 days old. *Uncompressed images* are for high resolution images, and should be removed first to keep the size of the database manageable. See `man pfstat`. 

Generate a graph image:
```
sudo pfstat -p
```

*Xorg* is required to view graphical images. Since we have no intention of installing *Xorg*, or even forwarding *X sessions* to our droplet, we will copy the image from the droplet to our remote workstation using `scp`.

Copy the image to a remote workstation:
```command 
sudo scp myuser@XXX.XXX.XX.XX:pfstat.jpg /home/myuser
```

As mentioned, there's a strong chance that nothing will appear on the graph because we're using a fresh droplet. We can try flooding the server with *ping* from our local workstation to generate mass quantities of packets. The server will block the *ping flood*, which should appear on the graph image.

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

If multiple log interfaces are needed we can create them with a `/etc/hostname` file, just as we would with any other interface.

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

We now have a portable base ruleset that can serve as a formidable starting point for all new droplets. Refer to the sample rulesets below, make copies of them and tuck them away somewhere safe. Do not blindly copy them without analyzing your exact needs. They are intended to be building blocks that can be molded into any scenario. Hopefully this tutorial was useful in getting you up to speed with [PF](https://www.freebsd.org/cgi/man.cgi?query=pf&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html), and hopefully you also picked up a few tips on Unix networking. See you in the next one!

### Sample ruleset 1 (identical to step 7)
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
[label Web Server Ruleset]
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
pass proto tcp to port $web_services set prio (6,7)
pass out proto tcp to port $tcp_services
pass out proto udp to port $udp_services
pass inet proto icmp icmp-type $icmp4_messages
```

- Sample rulest 1 is the base ruleset, which is a formidable starting point for fresh droplets that are not yet hosting any public or high traffic services.

- Sample ruleset 2 builds on ruleset 1, and is suitable for a public facing web application. It includes some highly restrictive measures on its SSH port, and some *prioritization features* on ports 80 and 443.


