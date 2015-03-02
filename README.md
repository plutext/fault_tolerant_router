# Fault Tolerant Router
[![Gem Version](https://badge.fury.io/rb/fault_tolerant_router.svg)](http://badge.fury.io/rb/fault_tolerant_router)
[![Dependency Status](https://gemnasium.com/drsound/fault_tolerant_router.svg)](https://gemnasium.com/drsound/fault_tolerant_router)
[![Code Climate](https://codeclimate.com/github/drsound/fault_tolerant_router/badges/gpa.svg)](https://codeclimate.com/github/drsound/fault_tolerant_router)

## In brief

Do you have multiple internet connections (uplinks) with several providers? Do you want to transparently use all of the available bandwidth? Do you want to remain online even if some of the uplinks go down? This tool may help you!

## A more formal description

Fault Tolerant Router is a daemon, running in background on a Linux router or firewall, monitoring the state of multiple internet uplinks/providers and changing the routing accordingly. LAN/DMZ internet traffic (outgoing connections) is load balanced between the uplinks using Linux *multipath routing*. The daemon monitors the state of the uplinks by routinely pinging well known IP addresses (Google public DNS servers, etc.) through each outgoing interface: once an uplink goes down, it is excluded from the *multipath routing*, when it comes back up, it is included again. All of the routing changes are notified to the administrator by email.

Fault Tolerant Router is well tested and has been used in production for several years, in several sites.

## Interaction between *multipath routing*, *iptables* and *ip policy routing*
The system is based on the interaction between Linux *multipath routing*, *iptables* and *ip policy routing*. Outgoing (from LAN/DMZ to WAN) and incoming (from WAN to LAN/DMZ) connections have a different behaviour:
* **Outgoing connections (from LAN/DMZ to WAN)**:
  * **New connections**:  
The outgoing interface (uplink) is decided by the Linux *multipath routing*, in a round-robin fashion. Then, just before the packet leaves the router (in the *iptables* POSTROUTING chain), *iptables* marks the connection with the outgoing interface id, so that all subsequent connection packets will be sent through the same interface.
NB: all the packets of the same connection should be originating from the same IP address, otherwise the server you are connecting to would refuse them (unless you are using specific protocols).
  * **Established connections**:  
Before the packet is routed (in the *iptables* PREROUTING chain), *iptables* marks it with the outgoing interface id that was previously assigned to the connection. This way, thanks to *ip policy routing*, the packet will pass through a specific routing table directing it to the connection outgoing interface.
* **Incoming connections (from WAN to LAN/DMZ)**:  
The incoming interface is obviously decided by the connecting host, connecting to one of the IP addresses assigned to an uplink interface. Just after the packet enters the router (in the *iptables* PREROUTING chain), *iptables* marks the connection with the incoming interface id. Then the packet reaches the LAN or DMZ, a return packet is generated by the receiving host and sent back to the connecting host. Once this return packet hits the router, before it is actually routed (in the *iptables* PREROUTING chain), *iptables* marks it with the outgoing interface id that was previously assigned to that connection. This way, thanks to *ip policy routing*, the return packet will pass through a specific routing table directing it to the connection outgoing interface.

## The uplink monitor daemon

The daemon monitors the state of the uplinks by pinging well known IP addresses through each uplink: if enough pings are successful the uplink is considered up, if not it's considered down. If an uplink state change is detected, the default *multipath routing* table (used for LAN/DMZ to WAN new connections) is changed accordingly and the administrator is notified by email.

The IP addresses to ping and the number of required successful pings is configurable. In order not to get false positives or negatives here are some things to consider:
* Some ping packets can randomly get lost along the way, don't require 100% of the pings to be successful!
* Some of the hosts you are pinging (*tests/ips* configuration parameter) may be temporarily down.
* It's better not to ping too near hosts (for example your provider routers), because your provider could be temporarily disconnected from the rest of the internet (it happened...), so your uplink would result as up while it's actually unusable.
* Sometimes an uplink can be not completely up or completely down, it's just "disturbed" and looses a lot of packets, being almost unusable: it's better to consider such uplink as down, so don't require too few successful pings, otherwise it may be considered up, because a few pings may pass through a "disturbed" link.

The order of IP addresses in *tests/ips* configuration parameter is not important, because the list is shuffled before every uplink check.

If no uplink is up, all of them are added to the default *multipath routing* table, to get some bandwidth as soon as one uplink comes back up.

## Requirements
* [Ruby](https://www.ruby-lang.org)
* A Linux kernel with the following compiled in options (they are standard in mainstream Linux distributions):
  * CONFIG_IP_ADVANCED_ROUTER
  * CONFIG_IP_MULTIPLE_TABLES
  * CONFIG_IP_ROUTE_MULTIPATH

## Installation
`$ gem install fault_tolerant_router`

## Usage
1. Configure your router interfaces as usual but **don't** set any default route. An interface may have more than one IP address if needed.
2. Save an example configuration file in /etc/fault_tolerant_router.conf (use the `--config` option to set another location):  
`$ fault_tolerant_router generate_config`
3. Edit /etc/fault_tolerant_router.conf
4. _(Optional)_ Demo how Fault Tolerant Router works, to familiarize with it:  
`$ fault_tolerant_router --demo monitor`
5. Generate *iptables* rules and integrate them with your existing ones:
`$ fault_tolerant_router generate_iptables`
6. _(Optional)_ Test email notification, to be sure SMTP parameters are correct and the administrator will get notifications:  
`$ fault_tolerant_router email_test`
7. Run the daemon:  
`$ fault_tolerant_router monitor`  
Previous command will actually run Fault Tolerant Router in foreground. To run it in background you should use your Linux distribution specific method to start it as a system service. See for example [start-stop-daemon](http://manned.org/start-stop-daemon).
If you want a quick and dirty way to run the program in background, just add an ampersand at the end of the command line:  
`$ fault_tolerant_router monitor &`

## Configuration file
The fault_tolerant_router.conf configuration file is in [YAML](http://en.wikipedia.org/wiki/YAML) format. Here is the explanation of the options:
* **uplinks**: array of uplinks. The example configuration has 3 uplinks, but you can have from 2 to as many as you wish.
  * **interface**: the network interface where the uplink is attached. Until today Fault Tolerant Router has always been used with each uplink on it's own physical interface, never tried with VLAN interfaces (it's in the to do list).
  * **ip**: primary IP address of the network interface. You can have more than one IP address assigned to the interface, just specify the primary one.
  * **gateway**: the gateway on this interface, usually the provider's router IP address.
  * **description**: used in the alert emails.
  * **weight**: optional parameter, it's the preference to assign to the uplink when choosing one for a new outgoing connection. Use when you have uplinks with different bandwidths. See http://www.policyrouting.org/PolicyRoutingBook/ONLINE/CH05.web.html
  * **default_route**: optional parameter, default value is *true*. If set to *false* the uplink is excluded from the *multipath routing*, i.e. the uplink will never be used when choosing one for a new outgoing connection. Exception to this is if some kind of outgoing connection is forced to pass through this uplink, see [Iptables rules](#iptables-rules) section. Even if set to *false*, incoming connections are still possible. Use cases to set it to *false*:
    * Want to reserve an uplink for incoming connections only, excluding it from outgoing LAN internet traffic. Tipically you may want this because you have a mail server, web server, etc. listening on this uplink.
    * Temporarily force all of the outgoing LAN internet traffic to pass through the other uplinks, to stress test the other uplinks and determine their bandwidth
    * Temporarily exclude the uplink to do some reconfiguration, for example changing one of the internet providers.
* **downlinks**
  * **lan**: LAN interface
  * **dmz**: DMZ interface, leave blank if you have no DMZ
* **tests**
  * **ips**: an array of IPs to ping to verify the uplinks state. You can add as many as you wish. Predefined ones are Google DNS, OpenDNS DNS, other public DNS. Every time an uplink is tested the ips are shuffled, so listing order has no importance.
  * **required_successful**: number of successfully pinged ips to consider an uplink to be functional
  * **ping_retries**: number of ping retries before giving up on an ip
  * **interval**: seconds between a check of the uplinks and the next one
* **log**
  * **file**: log file path
  * **max_size**: maximum log file size (in bytes). Once reached this size, the log file will be rotated.
  * **old_files**: number of old rotated files to keep
* **email**
  * **send**: set to *true* or *false* to enable or disable email notification
  * **sender**: email sender
  * **recipients**: an array of email recipients
  * **smtp_parameters**: see http://ruby-doc.org/stdlib-2.2.0/libdoc/net/smtp/rdoc/Net/SMTP.html
* **base_table**: just need to change if you are already using [multiple routing tables](http://lartc.org/howto/lartc.rpdb.html), to avoid overlapping
* **base_priority**: just need to change if you are already using [ip rule](http://lartc.org/howto/lartc.rpdb.html), to avoid overlapping
* **base_fwmark**: just need to change if you are already using packet marking, to avoid overlapping

## *Iptables* rules
*Iptables* rules are generated with the command:
`$ fault_tolerant_router generate_iptables`  
Rules are in [iptables-save](http://manned.org/iptables-save.8) format, you should integrate them with your existing ones.
Documentation is included as comments in the output, here is a dump using the standard example configuration:
```
#Integrate with your existing "iptables-save" configuration, or adapt to work
#with any other iptables configuration system

*mangle
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:INPUT ACCEPT [0:0]

#New outbound connections: force a connection to use a specific uplink instead
#of participating in the multipath routing. This can be useful if you have an
#SMTP server that should always send emails originating from a specific IP
#address (because of PTR DNS records), or if you have some service that you want
#always to use a particular slow/fast uplink.
#
#Uncomment if needed.
#
#NB: these are just examples, you can add as many options as needed: -s, -d,
#    --sport, etc.

#Example Provider 1
#[0:0] -A PREROUTING -i eth0 -m state --state NEW -p tcp --dport XXX -j CONNMARK --set-mark 1
#Example Provider 2
#[0:0] -A PREROUTING -i eth0 -m state --state NEW -p tcp --dport XXX -j CONNMARK --set-mark 2
#Example Provider 3
#[0:0] -A PREROUTING -i eth0 -m state --state NEW -p tcp --dport XXX -j CONNMARK --set-mark 3

#Mark packets with the outgoing interface:
#
#- Established outbound connections: mark non-first packets (first packet will
#  be marked as 0, as a standard unmerked packet, because the connection has not
#  yet been marked with CONNMARK --set-mark)
#
#- New outbound connections: mark first packet, only effective if marking has
#  been done in the section above
#
#- Inbound connections: mark returning packets (from LAN/DMZ to WAN)

[0:0] -A PREROUTING -i eth0 -j CONNMARK --restore-mark

#New inbound connections: mark the connection with the incoming interface.

#Example Provider 1
[0:0] -A PREROUTING -i eth1 -m state --state NEW -j CONNMARK --set-mark 1
#Example Provider 2
[0:0] -A PREROUTING -i eth2 -m state --state NEW -j CONNMARK --set-mark 2
#Example Provider 3
[0:0] -A PREROUTING -i eth3 -m state --state NEW -j CONNMARK --set-mark 3

#New outbound connections: mark the connection with the outgoing interface
#(chosen by the multipath routing).

#Example Provider 1
[0:0] -A POSTROUTING -o eth1 -m state --state NEW -j CONNMARK --set-mark 1
#Example Provider 2
[0:0] -A POSTROUTING -o eth2 -m state --state NEW -j CONNMARK --set-mark 2
#Example Provider 3
[0:0] -A POSTROUTING -o eth3 -m state --state NEW -j CONNMARK --set-mark 3

COMMIT


*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

#DNAT: WAN --> LAN/DMZ. The original destination IP (-d) can be any of the IP
#addresses assigned to the uplink interface. XXX.XXX.XXX.XXX can be any of your
#LAN/DMZ IPs.
#
#Uncomment if needed.
#
#NB: these are just examples, you can add as many options as you wish: -s,
#    --sport, --dport, etc.

#Example Provider 1
#[0:0] -A PREROUTING -i eth1 -d 1.0.0.2 -j DNAT --to-destination XXX.XXX.XXX.XXX
#Example Provider 2
#[0:0] -A PREROUTING -i eth2 -d 2.0.0.2 -j DNAT --to-destination XXX.XXX.XXX.XXX
#Example Provider 3
#[0:0] -A PREROUTING -i eth3 -d 3.0.0.2 -j DNAT --to-destination XXX.XXX.XXX.XXX

#SNAT: LAN/DMZ --> WAN. Force an outgoing connection to use a specific source
#address instead of the default one of the outgoing interface. Of course this
#only makes sense if more than one IP address is assigned to the uplink
#interface.
#
#Uncomment if needed.
#
#NB: these are just examples, you can add as many options as needed: -d,
#    --sport, --dport, etc.

#Example Provider 1
#[0:0] -A POSTROUTING -s XXX.XXX.XXX.XXX -o eth1 -j SNAT --to-source YYY.YYY.YYY.YYY
#Example Provider 2
#[0:0] -A POSTROUTING -s XXX.XXX.XXX.XXX -o eth2 -j SNAT --to-source YYY.YYY.YYY.YYY
#Example Provider 3
#[0:0] -A POSTROUTING -s XXX.XXX.XXX.XXX -o eth3 -j SNAT --to-source YYY.YYY.YYY.YYY

#SNAT: LAN --> WAN

#Example Provider 1
[0:0] -A POSTROUTING -o eth1 -j SNAT --to-source 1.0.0.2
#Example Provider 2
[0:0] -A POSTROUTING -o eth2 -j SNAT --to-source 2.0.0.2
#Example Provider 3
[0:0] -A POSTROUTING -o eth3 -j SNAT --to-source 3.0.0.2

COMMIT


*filter

:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
:LAN_WAN - [0:0]
:WAN_LAN - [0:0]

#This is just a very basic example, add your own rules for the INPUT chain.

[0:0] -A INPUT -i lo -j ACCEPT
[0:0] -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

[0:0] -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

[0:0] -A FORWARD -i eth0 -o eth1 -j LAN_WAN
[0:0] -A FORWARD -i eth0 -o eth2 -j LAN_WAN
[0:0] -A FORWARD -i eth0 -o eth3 -j LAN_WAN
[0:0] -A FORWARD -i eth1 -o eth0 -j WAN_LAN
[0:0] -A FORWARD -i eth2 -o eth0 -j WAN_LAN
[0:0] -A FORWARD -i eth3 -o eth0 -j WAN_LAN

#This is just a very basic example, add your own rules for the FORWARD chain.

[0:0] -A LAN_WAN -j ACCEPT
[0:0] -A WAN_LAN -j REJECT

COMMIT
```

## To do
* Test using VLAN interfaces: Fault Tolerant Router has always been used with physical interfaces, each uplink on it's own physical interface.
* Implement routing through [realms](http://www.policyrouting.org/PolicyRoutingBook/ONLINE/CH07.web.html): this way we could connect all of the uplinks to a single Linux physical interface through a switch, without using VLANs.
* i18n
* If no uplinks are up, set tests/interval configuration option to 0, to get bandwidth as soon as an uplink comes back up

## License
GNU General Public License v2.0, see LICENSE file

## Author
Alessandro Zarrilli (Poggibonsi - Italy)  
alessandro@zarrilli.net
