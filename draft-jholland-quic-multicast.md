---
title: "Multicast Extension for QUIC"
abbrev: "Multicast QUIC"
docname: draft-jholland-quic-multicast-latest
category: std
v: 3
ipr: trust200902
area: TSV
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

venue:

  group: "QUIC"
  type: "Individual Draft"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "GrumpyOldTroll/draft-jholland-quic-multicast"
  latest: "https://GrumpyOldTroll.github.io/draft-jholland-quic-multicast/draft-jholland-quic-multicast.html"

author:
 -
    fullname: Jacob Holland
    organization: Akamai Technologies, Inc.
    email: jakeholland.net@gmail.com

normative:

  I-D.draft-krose-multicast-security:
  I-D.draft-ietf-quic-multipath:
  RFC9000:

informative:

--- abstract

This document defines a multicast extension to QUIC to enable the efficient use of mullticast-capable networks to send identical data streams to many clients at once, coordinated through their individual connections to the same service.

--- middle

# Introduction

This document specifies an extension to QUIC version 1 {{RFC9000}} to enable the use of multicast transport of identical data streams from a set of sending servers to potentially many individual clients on many individual connections while making efficient use of multicast-capable networks.

This proposal tries to reuse as much as possible from {{RFC9000}} and {{I-D.draft-ietf-quic-multipath}}.

The multicast data can only be consumed in conjunction with a unicast
QUIC connection.  When support for multicast is negotiated, the server
can optionally advertise existence of one or more multicst channels that
contain unidirectional data streams from server to client, and the client
can optionally join the multicast channel, confirm to the server that
correct data is being received, and cut over to simply checking the
integrity of the multicast data via authentication anchors sent over
the unicst connection and acknowledging the data from the multicast
channel(s) over the unicast connection.

This document does not define any multicast transport except server to
client.

## Conventions and Definitions

{::boilerplate bcp14}

# Handshake Negotiation and Transport Parameter

# Multicast Session Advertisement, Subscription, and Removal

# Congestion Control

# Computing Path RTT

# Packet Scheduling

# Implementation and Operational Considerations

# New Frames

# Error Codes

# Security Considerations

Mostly incorporate draft-krose-multicast-security.  Anything else?

e.g. if a different legitimate quic connection says someone
else's quic multicast stream is theirs, that's maybe a problem
worth protecting against.  Maybe we need a periodic asymmetric
challenge?  I'm thinking send a public key on the multicast
channel and in the unicast channels send a MAC signed with
the private key, so that in addition to validating that the
unicast sender knows the contents of the multicast packets
via an AMBI-like hash, the multicast stream provides a way to
prove that the unicast stream is authorized to use it for data
transport via proof they know the channel's private key.
(Note this doesn't prevent unauthorized receipt of multicast
data packts, but does prevent a quic server from lying when
claiming a multicast data channel belongs to it, preventing
legit receivers from consuming it.)

# IANA Considerations

TODO

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
