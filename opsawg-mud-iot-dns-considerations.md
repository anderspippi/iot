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
  AmazonS3:
    title: "Amazon S3"
    target: "https://en.wikipedia.org/wiki/Amazon_S3"
    date: 2019
  Akamai: 
    title: "Akamai"
    target: "https://en.wikipedia.org/wiki/Akamai_Technologies"
    date: 2019
    
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

# Strategies to map names {#mapping}

The simplest strategy for translating names is for a MUD controller to take
is to do a DNS lookup on 
the name, and then use the resulting IP addresses to populate the physical ACLs.

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
known.   In this case, additional installation specific mechanisms are
probably needed to get the right view of DNS.

# DNS and IP Anti-Patterns for IoT device Manufacturers
  
This section describes a number of things with IoT manufacturers have been
observed to do in the field, each of which presents difficulties for MUD
enforcement points.

## Use of IP address literals in-protocol

A common pattern for a number of devices is to look for firmware updates in a
two step process.  An initial query is made (often over HTTPS, sometimes with
a POST, but the method is immaterial) to an authoritatve server.
The current firmware model of the device is sometimes provided and then the
authoritative server provides a determination if a new version is required,
and if so, what version.  In simpler cases, an HTTPS end point is queried
which provides the name and URL of the most recent firmware.  

The more complex case supports situations in which the device needs to be
running the latest patch release before it can apply the next major
release. For instance, a device running 1.4 must upgrade to at least version
1.9 before it is able to download version 2.0 of the firmware.

The authoritative upgrade server then responds with a URL of a firmware blob that the
device should download and install.  Best practice is that firmware is either
signed internally ({{-SUITARCH}}) so that it can be verified, or a hash of
the blob is provided.

The challenge for a MUD controller is in the details of the URL that is
provided.  An authoritative server might be tempted to provided an IP address
literal inside the protocol: there are two arguments for doing this.

One is that it eliminates problems to firmware updates that might be caused
by lack of DNS, or incompatibilities with DNS.  For instance bug that causes 
interoperability issues with some recursive servers would become unpatchable
for devices that were forced to use that recursive resolver type.

A second reason to avoid a DNS in the URL is when an inhouse content-distribution
system is involved that involves on-demand instances being added (or removed)
from a cloud computing architecture.  This model is typical of on-demand
video systems including Netflix (see [LOOKING FOR NETFLIX REF],
[WINDOWS UPDATE REF]), but this can occur in quite a number of other
situations.   Third-party content-distribution networks (CDN) tend to use DNS names
in order to isolate the content-owner from changes to the distribution
network.

[BEHAVE-BCP-REF] gives other good reasons why IP address literals are bad
ideas; in particular they work very poorly when devices have IPv6
capabilities, and are on IPv6-only networks with NAT64 (see {{RFC6146}}).

## Use of non-deterministic DNS names in-protocol

A second pattern is for a control protocol to connect to a known HTTP end
point.  This is easily described in MUD.  Within that control protocol
references are made to additional content at other URLs.  The values of those
URLs do not fit any easily described pattern and may point at arbitrary
names.

Those names are often within some third-party Content-Distribution-Network
(CDN) system, or may be arbitrary names in a cloud-provider storage system
such as Amazon S3 (such {{AmazonS3}}, or {{Akamai}}).

**INSERT** examples of non-deterministic CDN content.

Since it is not possible to predict a name for where the content will be, it
is not possible to include that into the MUD file.

This applies to the firmware update situation as well.

## Use of a too inclusive DNS name

Some CDNs make all customer content at a single URL (such as s3.amazonaws.com).  
This seems to be ideal from a MUD point of view: a completely predictable
URL.  The problem is that a compromised device could then connect to any S3
bucket, potentially attacking other buckets.

The MUD ACLs provide only for permitting end points and do not filter URLs
(nor could filtering be enforced within HTTPS). 

# DNS privacy and outsourcing vs MUD controllers

{{RFC 7858}} and {{RFC 8094}} provide for DNS over TLS and DTLS.  Other
recent proposals including doing DNS over HTTP.  But, even with traditional
DNS over Port-53 (Do53), it is possible to oursource DNS queries to other
places such as the QuadX systems ([QUADX reference]).

There are significant privacy issues with having IoT devices sending their
DNS queries to an outside entity.  Doing it over a secure transport (DoT/DoH)
is clearly better than doing so on port 53.  The providers of the secure
resolver service will still see the IoT device queries.

A described above in {{mapping}} the MUD controller needs to have access to
the same resolver(s) as the IoT device.  Use of the [QUADX] resolvers at
first seems to present less of a problem than use of some other less well
known resolver.  While any system may use [QUADX], in most cases those
services are massively replicated via anycast.  A MUD controller still needs
to be able to get talk to the same anycast instance as the IoT device to be
assured that it gets the same response.

# Recommendations to IoT device manufacturer on MUD and DNS usage

Inclusion of a MUD file with IoT devices is operationally quite simple.  
It requires only a few small changes to the DHCP client code to express the
MUD URL.  It can even be done without code changes via the use of a QR code
affixed to the packaging (see {{-securehomegateway-mud}}).

The difficult part is determining what to put into the MUD file itself.
There are currently tools that help with the definition and analysis of MUD
files, see {{mudmaker}}.  The remaining difficulty is now the semantic
contents of what is in the MUD file.  An IoT manufacturer must now spend some
time reviewing what the network communications that their device does.

This document has discussed a number of challenges that occur relating to how


# Privacy Considerations

TBD

# Security Considerations

TBD


--- back

# Appendices

