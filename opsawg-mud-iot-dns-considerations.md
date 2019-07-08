---
title: Operational Considerations for use of DNS in IoT devices
abbrev: mud-iot-dns
docname: draft-richardson-opsawg-mud-iot-dns-considerations

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

TBD

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

