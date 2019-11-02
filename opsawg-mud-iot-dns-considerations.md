---
title: Operational Considerations for use of DNS in IoT devices
abbrev: mud-iot-dns
docname: draft-richardson-opsawg-mud-iot-dns-considerations-01

# stand_alone: true

ipr: trust200902
area: Operations
wg: OPSAWG Working Group
kw: Internet-Draft
cat: bcp

coding: us-ascii
pi:    # can use array (if all yes) or hash here
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:


- ins: M. Richardson
  name: Michael Richardson
  org: Sandelman Software Works
  email: mcr+ietf@sandelman.ca

normative:
  RFC7858:
  RFC8520:
  RFC1794:  # DNS round robin support
  
informative:
  RFC1034:

--- abstract

This document details concerns about how Internet of Things devices use IP
addresses and DNS names.  The issue becomes acute as network
operators begin deploying RFC8520 Manufacturer Usage Description (MUD)
definitions to control device access.

This document explains the problem through a series of examples of what can
go wrong, and then provides some advice on how a device manufacturer can best
make deal with these issues.  The recommendations have an impact upon device
and network protocol design.

--- middle

# Introduction

{{RFC8520}} provides a standardized way to describe how a specific purpose
device makes use of Internet resources.  Access Control Lists (ACLs) can be
defined in an RFC8520 Manufacturer Usage Description (MUD) file that permit
a device to access Internet resources by DNS name.

Use of a DNS name rather than IP address in the ACL has many advantages: not
only does the layer of indirection permit the mapping of name to IP address
to be changed over time, it also generalizes automatically to IPv4 and IPv6
addresses, as well as permitting loading balancing of traffic by many
different common ways, including geography.

At the MUD policy enforcement point -- the firewall -- there is a problem.
The firewall has only access to the layer-3 headers of the packet.  This
includes the source and destination IP address, and if not encrypted by IPsec, the
destination UDP or TCP port number present in the transport header.  The DNS
name is not present!

In order to implement this, there must be a mapping between the names in the
ACLs and layer-3 IP addresses.  The first section of this document details a
few strategies that are used.

The second section of this document details how common manufacturer
anti-patterns get in the way this mapping.

The third section of this document details how current trends in DNS
resolution such as public DNS servers, DNS over TLS (DoT), and DNS over
HTTPS (DoH) cause problems for the strategies employed.  Poor interactions
with content-distribution networks is a frequent pathology that results.

The fourth section of this document makes a series of recommendations ("best
current practices") for manufacturers on how to use DNS, and IP addresses
with specific purpose IoT devices.

The Privacy Considerations section concerns itself with issues that
DNS-over-TLS and DNS-over-HTTPS are frequently used to deal with.  The
question is how these concerns apply to IoT devices located within a
residence or enterprise is dealt with.

The Security Considerations section covers some of the negative outcomes
should MUD/firewall managers and IoT manufacturers choose not to cooperate.

# Strategies to map names

The simplest strategy for a MUD controller to take is to do a DNS lookup on
the name, and then use the resulting IP addresses to populate the ACLs.

There are a number of failures possible.  The most important one is in the
mapping of the names to IP addresses.  {{RFC1794}} describes how a common
mechanism that returns DNS A (or reasonably AAAA) records in a permutted
order.  As long as all possible A/AAAA records are returned then ACLs
can be setup for all possibilities.

There are a number of circumstances in which the list is not exhaustive.  The
simplest is when the round robin does not return all addresses.  This is
routinely done by geographical DNS load balancing system.  In such a system
the address that is returns depends upon the network locality of the asking
system.  There may also be further layers of round-robin indirection.

Aside from the list of records being incomplete, the list may have changed
between the time that the MUD controller did the lookup and the time that the
IoT device does the lookup, and this change can result in a failure in the
mapping.

In order to compensate for this, the MUD controller SHOULD regularly do DNS
lookups.  These lookups need to be rate limited in order to avoid load.
It may be necessary to avoid recursive DNS servers in order to avoid
receiving cached data.  Properly designed recursive servers should cache data
for many minutes to days, while the underlying DNS data can change at a
higher frequency, providing different answers to different queries.

A MUD controller that is aware of which recursive DNS server that the IoT
device will use can instead query that server on a periodic basis.  Doing so
provides three advantages:

1. any geographic load balancing will base the decision on the geolocation of
   the recursive DNS server, and the recursive name server will provide the
   same answer to the MUD controller as to the IoT device.

2. the resulting name to IP address mapping in the recursive name server will
   be cached, and will remain the same for the entire advertised Time-To-Live
   reported in the DNS query return.  This also allows the MUD controller to
   avoid doing unnecessary queries.
 
3. if any addresses have been omitted in a round-robin DNS process, the cache
   will have the set of addresses that were returned.

The naive method of trying to map IP addresses to names will in the ACLs will
not work: the reverse DNS map is frequently not populated, or if it is, it is
populated with a name that is not the same name as in the MUD file ACL.  This
is trivial to understand when virtual hosting for web servers is
considered. Many names map to a single IP address, but multiple names are
seldom populated into the reverse PTR records.

Additionally, mapping IP addresses to names in real time, when making packet
forwarding decisions is not practical from a performance point of view.

The solution of using the same caching recursive resolver as the target
device is very simple when the MUD controllers is located in a residential
CPE device.  The device is usually also the policy enforcement point for the
ACLs, and a caching resolver is typically located on the same device.  In
addition the convenience, there is a shared fate advantage: as all 
three components are running on the same device, if the device is rebooted,
clearing the cache, then all three components will get restarted when the
device is restarted.

Where the solution is more complex is when the MUD controller is located
elsewhere in an Enteprise, or remotely in a cloud such as when a Software
Defines Network (SDN) is used to manage the ACLs.  The DNS servers for a
particular device may not be known to the MUD controller, nor the MUD
controller be even permitted to make recusive queries that server if it is
known.   In this case, additional mechanisms are probably needed to get the
right view of DNS.

# DNS and IP Anti-Patterns for IoT device Manufacturers

TBD

# DNS privacy and outsourcing vs MUD controllers

TBD

# Recommendations on MUD and DNS co-existence

TBD

# Privacy Considerations

TBD

# Security Considerations

TBD


--- back

# Appendices

