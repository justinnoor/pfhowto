# How to Configure PF on a FreeBSD 12.1 Digital Ocean Droplet Draft

### Introduction

Networks are hostile places. Every organization will face this reality unless they are completely offline and internally secure. The firewall is arguably one of the most important lines of defense in the realm of network security. The ability to configure a firewall from scratch is an empowering skill that enables the administrator take control of their networks.

In this tutorial we build a firewall from the ground up on a FreeBSD 12.01 droplet using [PF](https://man.openbsd.org/pf), a renown packet filtering application that is maintained by the [OpenBSD](https://www.openbsd.org) project. [PF](https://man.openbsd.org/pf) is part of the FreeBSD base system, and is a foundational tool in the world of Unix administration. It is known for its simple-syntax, user-friendliness, and astonishing power.

This lesson is *not* a command reference. Instead we will build a working ruleset that can used as a formidable starting point for new FreeBSD droplets. We start with the simplest ruleset possible and progress into more advanced concepts such as proactive defense, bandwidth management, redundancy, and more.

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

## Step 3 - Build a preliminary ruleset

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

Now our ruleset allows **SSH, HTTP, NTP, and HTTPS** traffic to pass in the outward direction. Notice we placed the port numbers inside of the curly brackets. This is a **list** in [PF](https://man.openbsd.org/pf) syntax, which is an enormously convenient feature of [PF](https://man.openbsd.org/pf) that will be further discussed below. For the sake of uniformity we will do the same for our **pass in** rule. Our preliminary ruleset now looks like this:
```
set skip on lo0
block all
pass in proto tcp to port { 22 }
pass out proto tcp to port { 22 80 123 443 }
pass out proto udp to port { 53 123 }
pass inet proto icmp icmp-type echoreq
```

That is our preliminary ruleset, so be sure that it is saved. Feel free to make a backup copy of it as well. Notice we added some new rules. The `set skip` rule prevents us from filtering traffic on our loopback device, which is unnecessary. The **pass out** rule for **UDP** is for ports **53** and **123**, which will provide DNS and NTP services. Both of these services occasionally toggle between the TCP and UDP protocols, hence they are specified in two separate rules. There is a wealth of information on the internnet as to why this happens. Finally, the **pass** rule for the **ICMP** protocol is so that we can use [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) for troubleshooting purposes.

ICMP is a sometimes controversial messaging protocol. As long as it is approached with care there is no harm in using it. For our purposes, we just want to use [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports), that's it. The [ping(8)](https://www.freebsd.org/cgi/man.cgi?query=ping&sektion=8&manpath=freebsd-release-ports) command utilizes *echo request* messages of the ICMP protocol. In our **pass** rule above we only added the `echoreq` parameter, so this will be the only type of message allowed pass. The *echoreq* parameter is the actual message code name from the [icmp(4)](https://www.freebsd.org/cgi/man.cgi?query=icmp&sektion=4&apropos=0&manpath=FreeBSD+12.0-RELEASE+and+Ports) protocol, which can be viewed by running `$ man icmp`.

Now we can perform some essential tasks from inside our droplet such as install packages, sync with an NTP server, transfer files with SCP, and more. By default, [PF](https://man.openbsd.org/pf) filters traffic statefully, meaning that it stores information about connections in what is known as the *state table*. Therefore if we establish any connections with the ports in our **pass out** rule, [PF](https://man.openbsd.org/pf) knows when incoming packets are affiliated with that connection. This speeds things up because those packets are inspected against the *state table* when they arrive.

## Step 4 - Test our preliminary ruleset

Although we are far from finished, we have a working ruleset. That said, let's test it on our droplet during these early stages so that we can detect any errors early on.

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

Let's perform a few more sanity checks to ensure that everything is working. 

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

Test name resolution by updating the **pkgs** repository:
```super_user
pkg upgrade
```

If any packages need to be updated go ahead and update them. If these services are working, our ruleset is working, and we can proceed.

## Step 4 - Introduce macros and lists

In our preliminary ruleset above we hard coded our parameters into our rules, such as the port numbers and the ICMP parameter. This works for now, but will probably become unmanageable as our ruleset expands. For organizational purposes [PF](https://man.openbsd.org/pf) offers lists and macros, which allow us to conolidate all of our parameters. Readabiliy is a key responsibility of maintaining a ruleset.

Let's egin by creating a macro for our network interface. On FreeBSD droplets the default interface on a single droplet is `vtnet0`. 

Add this macro to the top of our ruleset:
```super_user
public_if = "vtnet0"
```

## Step 5 - Proactive defense

## Step 6 - Bandwidth management

## Monitoring and logging

## Conclusion

## Links

[bandwidth billing](https://www.digitalocean.com/docs/accounts/billing/bandwidth)






