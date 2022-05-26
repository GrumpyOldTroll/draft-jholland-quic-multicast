---
title: "Multicast Extension for QUIC"
abbrev: "Multicast QUIC"
docname: draft-jholland-quic-multicast-latest
category: exp
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
    fullname: Jake Holland
    organization: Akamai Technologies, Inc.
    email: jakeholland.net@gmail.com
 -
    fullname: Lucas Pardue
    email: lucaspardue.24.7@gmail.com
 -
    fullname: Max Franke
    organization: TU Berlin
    email: mfranke@inet.tu-berlin.de

normative:

  I-D.draft-krose-multicast-security:
  I-D.draft-ietf-quic-multipath:
  RFC9000:
  RFC9001:
  RFC9221:

informative:

  RFC4607:

--- abstract

This document defines a multicast extension to QUIC to enable the efficient use of mullticast-capable networks to send identical data streams to many clients at once, coordinated through individual unicast QUIC connections.

--- middle

# Introduction

This document specifies an extension to QUIC version 1 {{RFC9000}} to enable the use of multicast IP transport of identical data packets for use in many individual QUIC connections.

The multicast data can only be consumed in conjunction with a unicast QUIC connection.
When support for multicast is negotiated, the server can optionally advertise the existence of one or more multicast channels that contain unidirectional data streams from server to client, and the client can optionally join the multicast channel and verify from integrity data the server provides that correct data is being received, then acknowledge the data from the multicast channel(s) over the unicast connection.

Enabling this can provide large scalability benefits for popular traffic over multicast-capable networks.

This document does not define any multicast transport except server to client and only includes semantics for source-specific multicast.

## Conventions and Definitions

{::boilerplate bcp14}

Commonly used terms in this document are described below.

(S,G): A tuple of IP addresses identifying a source-specific multicast channel, as described in {{RFC4607}}.

# Multicast Channel

A QUIC multicast channel (or just channel) is a one-way network path that a server can use as an alternate path to send QUIC connection data to a client.

Multicast channels are designed to leverage multicast IP and to be shared by many different connections simultaneously for unidirectional server-initiated data.
One or more servers can use the same QUIC multicast channel to send the same data to many clients, as a supplement to the individual QUIC connections between those servers and clients.

Each QUIC multicast channel has exactly one associated (S,G) that is used for the delivery of the multicast packets on the IP layer. Channels do not support any-source multicast semantics.
This however does not impose a requirement on how the underlying network stack has to handle the forwarding and delivery of multicast packets.

QUIC connections are defined in Section 5 of {{RFC9000}} and are not changed in this document; each connection is a shared state between a client and a server.

Channels carry only 1-RTT packets.
Packets associated with a channel contain a Channel ID in place of a Destination Connection ID.
(A Channel ID cannot be zero length.)
This adds a layer of indirection to the process described in Section 5.2 of {{RFC9000}} for matching packets to connections upon receipt.
Incoming packets received on the network path associated with a channel use the Channel ID to associate the packet with a joined channel.

A client with a matching joined channel always has at least one connection associated with the channel.
If a client has no matching joined channel, the packet is discarded.

Since the network path for a channel is unidirectional, packets associated with a channel are acknowledged with MP_CHANNEL_ACK frames {{channel-ack-frame}} instead of ACK frames.
Each channel has an independent packet number space.

The use of any particular channel is OPTIONAL for both the server and the client.
It is recommended that applications designed to leverage the multicast capabilities of this extension also provide graceful degradation for endpoints that do not or cannot make use of the multicast functionality.

The server has access to all data transmitted on any multicast channel it uses, and could optionally send this data with unicast instead.

No special handling of the data is required in a client application that has enabled multicast.
A datagram or any particular bytes from a server-initiated unidirectional stream can be delivered over the unicast connection or a multicast channel transparently to the client.

Client applications should have a mechanism that disables the use of multicast on connections with enhanced privacy requirements for the privacy-related reasons covered in {{I-D.draft-krose-multicast-security}}.

# Transport Parameter {#transport-parameter}

Support for multicast extensions in a client is advertised by means of a QUIC transport parameter:

 * name: multicast_client_params (TBD - experiments use 0xff3e800)

If a multicast_client_params transport parameter is not included, servers MUST NOT send any frames defined in this document.  (Given that a server never sends any MC_CHANNEL_JOIN frames, the clients also will never send any frames in this document so only the client-to-server advertisement is necessary.)

The multicast_client_params parameter has the structure shown below in {{fig-transport-parameter-format}}.

~~~
multicast_client_params {
  Capabilities Field (i),
  Max Aggregate Rate (i),
  Max Channel IDs (i),
  Max Joined Count (i),
  Hash Algorithms Supported (i),
  AEAD Algorithms Supported (i),
  Hash Algorithms List (16 * Hash Algorithms Supported),
  AEAD Algorithms List (16 * AEAD Algorithms Supported)
}
~~~
{: #fig-transport-parameter-format title="multicast_client_params Format"}

Capabilities Flags is a bit field structured as follows:

 - 0x1 is set if IPv4 channels are permitted
 - 0x2 is set if IPv6 channels are permitted

A server MUST NOT send MC_CHANNEL_ANNOUNCE ({{channel-announce-frame}}) frames with addresses using an IP Family that is not supported according to the Capabilities in the multicast_client_params, unless and until a later MC_CLIENT_LIMITS ({{client-limits-frame}})  frame adds permission for a different address family.

The Capabilities Field, Max Aggregate Rate, Max Channel IDs and Max Joined Count are the same as in MC_CLIENT_LIMITS frames ({{client-limits-frame}}) and provide the initial client values.

The AEAD Algorithms List field is in order of preference (most preferred occuring first) using values from the registry below. It lists the algorithms the client is willing to use to decrypt data in multicast channels, and the server MUST NOT send a MC_CHANNEL_JOIN to this client for any channels using unsupported algorithms:

  - <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>

The Hash Algorithms List field is in order of preference (most preferred occurring first) using values from the registry below. It lists the algorithms the client is willing to use to check integrity of data in multicast channels, and the server MUST NOT send a MC_CHANNEL_JOIN to this client for any channels using unsupported algorithms:

 - <https://www.iana.org/assignments/named-information/named-information.xhtml#hash-alg>

# Extension Overview

A client has the option of refusal and the power to impose upper bound maxima on several resources (see {{flow-control}}), but otherwise its join status for all multicast channels is entirely managed by the server.

 * A client MUST NOT join a channel without receiving instructions from a server to do so.
 * A client MUST leave joined channels when instructed by the server to do so.
 * A client MAY leave channels or refuse to join channels, regardless of instructions from the server.

## Channel Management

The client tells its server about some restrictions on resources that it is capable of processing with the initial values in the multicast_client_params transport parameter ({{transport-parameter}}) and later can update these limits with MC_CLIENT_LIMITS {{client-limits-frame}} frames. Servers ensure the set of channels the client is currently requested to join remains within these advertised client limits as covered in {{flow-control}}.

The server asks the client to join channels with MC_CHANNEL_JOIN ({{channel-join-frame}}) frames and to leave channels with MC_CHANNEL_LEAVE ({{channel-leave-frame}}) frames.

The server uses MC_CHANNEL_ANNOUNCE ({{channel-announce-frame}}) and MC_CHANNEL_PROPERTIES ({{channel-properties-frame}}) frames before any join or leave frames for the channel to describe the channel properties to the client, including values the client can use to ensure the server's requests remain within the limits it has sent to the server, as well as the keys necessary to decode packets in the channel.

When the server has asked the client to join a channel, it also sends MC_CHANNEL_INTEGRITY frames ({{channel-integrity-frame}}) to enable the client to verify packet integrity before processing the packet.
A client MUST NOT decode packets for a channel for which it has not received an applicable set of MC_CHANNEL_PROPERTIES ({{channel-properties-frame}}) frames containing the full set of data required, or for which it has not received a matching packet hash in an MC_CHANNEL_INTEGRITY ({{channel-integrity-frame}}) frame.

The server ensures that in aggregate, all channels that the client has currently been asked to join and that the client has not left or declined to join fit within the limits indicated by the initial values in the transport parameter or last MC_CLIENT_LIMITS ({{client-limits-frame}}) frame the server received.

## Client Response

The client sends back information about how it has responded to the server's requests to join and leave channels in MC_CLIENT_CHANNEL_STATE ({{client-channel-state-frame}}) frames.
MC_CLIENT_CHANNEL_STATE frames are only sent for channels after the server has requested the client to join the channel, and are thereafter sent any time the state changes.

Clients that receive and decode data on a multicast channel send acknowledgements for the data on the unicast connection using MC_CHANNEL_ACK ({{channel-ack-frame}}) frames.
Channels also will periodically contain PATH_CHALLENGE ({{RFC9000}} Section 19.17) frames, which cause clients to send MC_PATH_RESPONSE ({{path-response-frame}}) frames on the unicast connection in addition to their MC_CHANNEL_ACK frames.

A server can determine if a client can receive packets on a multicast channel if it receives MC_CHANNEL_ACK frames associated with that channel.
As such, it is in general up to the server to decide on the time after which it deems a client to be unable to receive packets on a given channel and take appropriate steps, e.g. sending a MC_CHANNEL_LEAVE frame to the client.
However, a client MAY unilaterally determine that it is unable to receive multicast packets on a channel and indicate this to the server by sending a MC_CLIENT_CHANNEL_STATE frame with state Left and reason No Traffic.

## Data Carried in Channels

Data transmitted in a multicast channel is encrypted with symmetric keys so that on-path observers without access to these keys cannot decode the data.
However, since potentially many receivers receive identical packets and identical keys for the multicast channel and some receivers might be malicious, the packets are also protected by MC_CHANNEL_INTEGRITY ({{channel-integrity-frame}}) frames transmitted over a separate integrity-protected path.

A client MUST NOT decode packets on a multicast channel for which it has not received a matching hash in an MC_CHANNEL_INTEGRITY frame over a different integrity-protected communication path.
The different path can be either the unicast connection or another multicast channel with packets that were verified with an earlier MC_CHANNEL_INTEGRITY frame.

## Stream Processing

Stream IDs in channels are restricted to unidirectional server initiated streams, or those with the least significant 2 bits of the stream ID equal to 3 (see {{RFC9000}} Section 2.1).

When a channel contains streams with ids above the client's unidirectional MAX_STREAMS, the server MUST NOT instruct the client to join that channel and SHOULD send a STREAMS_BLOCKED frame, as described in Sections 4.6 and 19.14 of {{RFC9000}}.

If the client is already joined to a channel that carries streams that exceed or will soon exceed the client's unidirectional MAX_STREAMS, the server SHOULD send a MC_CHANNEL_LEAVE frame.

If a client receives a STREAM frame with an ID above its MAX_STREAMS on a channel, the client MAY increase its unidirectional MAX_STREAMS to a value greater than the new ID and send an update to the server, otherwise it MUST drop the packet and leave the channel with reason Max Streams Exceeded.

Since clients can join later than a channel began, it is RECOMMENDED that clients supporting the multicast extensions to QUIC be prepared to handle stream IDs that do not begin at early values, since by the time a client joins a channel in progress the stream id count might have been increasing for a long time.
Clients should therefore begin with a high initial_max_streams_uni or send an early MAX_STREAMS type 0x13 value (see Section 19.11 of {{RFC9000}}) with a high limit.
Clients MAY use the maximum 2^60 for this high initial limit, but the specific choice is implementation-dependent.

# Flow Control {#flow-control}

The values used for unicast flow control cannot be used to limit the transmission rate of a multicast channel because a single client with a low MAX_STREAM_DATA or MAX_DATA value that did not acknowledge receipt could block many other receivers if the servers had to ensure that channels responded to each client's limits.

Instead, clients advertise resource limits that the server is responsible for staying within via MC_CLIENT_LIMITS ({{client-limits-frame}}) frames and their initial values from the transport parameter ({{transport-parameter}}).
The server advertises the expected maxima of the values that can contribute toward client resource limits within a channel in MC_CHANNEL_PROPERTIES ({{channel-properties-frame}}) frames.

If the server asks the client to join a channel that would exceed the client's limits with an up-to-date Client Limit Sequence Number, the client should send back a MC_CHANNEL_STATE_CHANGE with "Declined Join" and reason "Property Violation".
If the server asks the client to join a channel that would exceed the client's limits with an out-of-date Client Limit Sequence Number or a Channel Property Sequence Number that the client has not yet seen, the client should instead send back a "Declined Join" with "Desynchronized Limit Violation".
If the actual contents sent in the channel exceed the advertised limits from the MC_CHANNEL_PROPERTY, clients SHOULD leave the stream with a PROTOCOL_ERROR/Limit Violated state change.

# Congestion Control

The server maintains a full view of the traffic received by the client via the ACK frames coupled with the MC_CHANNEL_ACK ({{channel-ack-frame}}) frames.

Under sustained persistent loss, the server SHOULD instruct the client such that the aggregate rate of joined channels remains under the data rate successfully received by the client in the recent past.

# Data Integrity

TODO: import the {{I-D.draft-krose-multicast-security}} explanation for why extra integrity protection is necessary (many client have the shared key, so AEAD doesn't provide authentication against other valid clients on its own).

## Packet Hashes {#packet-hashes}

TODO: explanation and example for how to calculate the packet hash.
Note that the hash is on the unencrypted packet because it checks against a specific packet number, which is protected by AEAD.
(This approach also may help make better use of crypto hardware offload.)

# Packet Scheduling


# Connection Termination
Termination of the unicast connection behaves as described in Section 10 of {{RFC9000}}, with the following notable differences:

* On the client side, termination of the unicast connection means that it MUST leave all multicast channels and discard any state associated with them. Servers MAY stop sending to multicast channels if there are no unicast connections left that are associated with them.

* For determining the liveness of a connection, the client MUST only consider packets received on the unicast connection. Any packets received on a multicast channel MUST NOT be used to reset a timer checking if a potentially specified max_idle_timeout has been reached. If the unicast connection becomes idle and the server does not respond to a liveness test, the client MUST terminate the connection as described above.

## Stateless Reset
As clients can unilaterally stop the delivery of multicast packets by leaving the relevant (S,G), channels do not need stateless reset tokens.
Clients therefore do not share the stateless reset tokens of channels with the server. Instead, if an endpoint receives packets addressed to an (S,G) that it can not associate with any existing channel,
it MAY take the necessary steps to prevent the reception of further such packets, without the need to signal to the server that it should stop sending.

If a server or client somehow still detect a stateless reset for a channel, they MUST ignore it.


# Implementation and Operational Considerations

# New Frames

## MC_CHANNEL_ANNOUNCE {#channel-announce-frame}

Once a server learns that a client supports multicast through its transport parameters, it can send one or multiple MC_CHANNEL_ANNOUNCE frames (type=TBD-11..TBD-22) to share information about available channels with the client.
The MC_CHANNEL_ANNOUNCE frame contains the static properties of a channel that do not change during its lifetime.

MC_CHANNEL_ANNOUNCE frames are formatted as shown in {{fig-mc-channel-announce}}.

~~~
MC_CHANNEL_ANNOUNCE Frame {
  Type (i) = TBD-11..TBD-12 (experiments use 0xff3e811/0xff3e812),
  ID Length (8),
  Channel ID (8..160),
  Source IP (32..128),
  Group IP (32..128),
  UDP Port (16),
  Header AEAD Algorithm (16),
  Header Key Length (i),
  Header Key (..),
  AEAD Algorithm (16),
  Integrity Hash Algorithm (16)
}
~~~
{: #fig-mc-channel-announce title="MC_CHANNEL_ANNOUNCE Frame Format"}

Frames of type TBD-10 are used for IPv4 and both Source and Group address are 32 bits long. Frames of type TBD-11 are used for IPv6 and both Source and Group address are 128 bits long.

MC_CHANNEL_ANNOUNCE frames contain the following fields:

  * ID Length: The length in bytes of the Channel ID field.
  * Channel ID: The channel ID of the channel that is getting announced.
  * IP Family: Unset indicates IPv4, Set indicates IPv6 for both Source IP and Group IP.
  * Source IP: The IP Address of the source of the (S,G) for the channel.  Either a 32-bit IPv4 address or a 128-bit IPv6 address, as indicated by IP Family.
  * Group IP: The IP Address of the group of the (S,G) for the channel.  Either a 32-bit IPv4 address or a 128-bit IPv6 address, as indicated by IP Family.
  * UDP Port: The 16-bit UDP Port of traffic for the channel.
  * Header AEAD Algorithm: A value from <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>, used to protect the header fields in the channel packets.  The value MUST match a value provided in the "AEAD Algorithms List" of the transport parameter (see {{transport-parameter}}).
  * Header Key Length: Provides the length of the Key field.  It MUST match a valid key length for the Header AEAD Algorithm.
  * Header Key: A key for use with the Header AEAD Algorithm for protecting the header fields of 1-RTT packets in the channel as described in {{RFC9001}}.
      * **Author's Note:** I assume it’s not better to use a TLS CipherSuite because there is no KDF stage for deriving these keys (they are a strict server-to-client advertisement), so the Hash part would be unused? (<https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4>)
  * AEAD Algorithm: A value from <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>.  The value MUST match a value provided in the "AEAD Algorithms List" of the transport parameter (see {{transport-parameter}}).
  * Integrity Hash Algorithm: The hash algorithm used in integrity frames.
    * **Author's Note:** Several candidate iana registries, not sure which one to use?  Some have only text for some possibly useful values.  For now we use the first of these:
      - <https://www.iana.org/assignments/named-information/named-information.xhtml#hash-alg>
      - <https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-18>
      - (text-only): <https://www.iana.org/assignments/hash-function-text-names/hash-function-text-names.xhtml>

A client MUST NOT use the channel ID included in the MC_CHANNEL_ANNOUNCE frame as a connection ID for the unicast connection. If it is already in use, the client should retire it as soon as possible.
As the server knows which connection IDs are in use by the client, it MUST wait with the sending of a MC_CHANNEL_JOIN frame until the channel ID associated with it has been retired by the client.

As the properties in MC_CHANNEL_ANNOUNCE frames are immutable during the lifetime of a channel, a server SHOULD NOT send a MC_CHANNEL_ANNOUNCE frame for the same channel more than once to each client.

A server SHOULD send an MC_CHANNEL_ANNOUNCE frame for a channel before sending a MC_CHANNEL_PROPERTIES or MC_CHANNEL_JOIN frame for it.

## MC_CHANNEL_PROPERTIES {#channel-properties-frame}

An MC_CHANNEL_PROPERTIES frame (type=TBD-01) is sent from server to client, either with the unicast connection or in an existing joined multicast channel.
The MC_CHANNEL_PROPERTIES frame consists of the properties of a channel that are mutable and might change during the course of its lifetime.

A server can send an update to a prior MC_CHANNEL_PROPERTIES frame with a new sequence number increased by one.

It is RECOMMENDED that servers set an Until Packet Number and send regular updates to the MC_CHANNEL_PROPERTIES before the packet numbers in the channel exceed that value.

MC_CHANNEL_PROPERTIES frames are formatted as shown in {{fig-mc-channel-properties-format}}.

~~~
MC_CHANNEL_PROPERTIES Frame {
  Type (i) = TBD-01 (experiments use 0xff3e801),
  ID Length (8),
  Channel ID (8..160),
  Properties Sequence Number (i),
  From Packet Number (i),
  Until Packet Number (i),
  Key Length (i),
  Key (..),
  Max Rate (i),
  Max Idle Time (i),
  ACK Bundle Size (i)
}
~~~
{: #fig-mc-channel-properties-format title="MC_CHANNEL_PROPERTIES Frame Format"}


MC_CHANNEL_PROPERTIES frames contain the following fields:

  * ID Length: The length in bytes of the Channel ID field.
  * Channel ID: The channel ID for the channel associated with this frame.
  * Properties Sequence Number: Increases by 1 each time the properties for the channel are changed by the server.  The client tracks the sequence number of the MC_CHANNEL_PROPERTIES frame that set its current value, and only updates the value and the packet number range on which it's applicable if the Properties Sequence Number is higher.
  * From Packet Number, Until Packet Number: The values in this MC_CHANNEL_PROPERTIES frame apply only to packets starting at From Packet Number and continuing for all packets up to and including Until Packet Number.  If Until Packet Number is zero it indicates the current property values for this channel have no expiration (equivalent to the maximum value for packet numbers, or 2^62-1).  If a packet number is received outside of any previously received \[From,Until\] range, it has no applicable channel properties and MUST be dropped.
  * Key Length: Provides the length of the Key field.  It MUST match a valid key length for the AEAD Algorithm from the MC_CHANNEL_ANNOUNCE frame for this channel.
  * Key: Used to protect the packet contents of 1-RTT packets for the channel as described in {{RFC9001}}, with length given by Key Length.
    To maintain forward secrecy and prevent malicious clients from decrypting packets long after they have left or were removed from the unicast connection, servers SHOULD periodically send key updates using only unicast.
  * Max Rate: The maximum rate in Kibps of the payload data for this channel.Channel data MUST NOT exceed this rate over any 5s window, if it does clients SHOULD leave the channel with reason Max Rate Exceeded.
  * Max Idle Time: The maximum expected idle time of the channel.  If this amount of time passes in a joined channel without data received, clients SHOULD leave the channel with reason Max Idle Time Exceeded.
  * ACK Bundle Size: The minimum number of ACKs a client should send in a single QUIC packet. If the max_ack_delay would force a client to send a packet that only consists of MC_CHANNEL_ACK frames, it SHOULD instead wait with sending until at least the specified number of acknowledgements have been collected. However, the Client MUST send any pending acknowledgements at least once per Max Idle Time to prevent the Server from perceiving the channel as interrupted.

From Packet Number and Until Packet Number are used to indicate the packet number (Section 17.1 of {{RFC9000}}) the 1-RTT packets received over which these values are applicable.

A From Packet Number without an Until Packet Number has an unspecified termination.

If new property values appear and are different from prior values, the From Packet Number implicitly sets the Until Packet Number of the prior property value equal to one below the new From Packet Number for all the changed properties.

The properties of a channel MAY change during its lifetime. As such, a server SHOULD NOT send properties for channels except those the client has joined or will be imminently asked to join.

## MC_CHANNEL_JOIN {#channel-join-frame}

An MC_CHANNEL_JOIN frame (type TBD-02) is sent from server to client and requests that a client join the given transport addresses and ports and process packets with the given Channel ID according to the corresponding MC_CHANNEL_PROPERTIES.

A client cannot join a multicast channel without first receiving a MC_CHANNEL_ANNOUNCE and MC_CHANNEL_PROPERTIES frame which together set all the values for a channel.

If a client receives a MC_CHANNEL_JOIN for a channel for which it has not received both, it MUST respond with a MC_CLIENT_CHANNEL_STATE with State "Declined Join" and reason "Missing Properties". The server MAY send another MC_CHANNEL_JOIN after retransmitting the MC_CHANNEL_PROPERTIES and receiving an acknowledgement indicating receipt of the MC_CHANNEL_ANNOUNCE.

MC_CHANNEL_JOIN frames are formatted as shown in {{fig-mc-channel-join-format}}.

~~~
MC_CHANNEL_JOIN Frame {
  Type (i) = TBD-02 (experiments use 0xff3e802),
  MC_CLIENT_LIMIT Sequence Number (i),
  MC_CLIENT_CHANNEL_STATE Sequence Number (i),
  MC_CHANNEL_PROPERTIES Sequence Number (i),
  ID Length (8),
  Channel ID (8..160)
}
~~~
{: #fig-mc-channel-join-format title="MC_CHANNEL_JOIN Frame Format"}

The sequence numbers are the most recently processed sequence number by the server from the respective frame type. They are present to allow the client to distinguish between a broken server that has performed an illegal action and an instruction that's based on updates that are out of sync (either one or more missing updates to MC_CHANNEL_PROPERTIES not yet received by the client or one or more missing updates to MC_CLIENT_LIMITS or MC_CLIENT_CHANNEL_STATE not yet received by the server).

A client MAY perform the join if it has the sequence number of the corresponding channel properties and the client's limits will not be exceeded, even if the client sequence numbers are not up-to-date.
If the client does not join, it MUST send a MC_CLIENT_CHANNEL_STATE with "Declined Join" and a reason.

## MC_CHANNEL_LEAVE {#channel-leave-frame}

An MC_CHANNEL_LEAVE frame (type=TBD-03) is sent from server to client, and requests that a client leave the given channel.

If the client has already left or declined to join the channel, the MC_CHANNEL_LEAVE is ignored.

If a MC_CHANNEL_JOIN or an MC_CHANNEL_LEAVE with the same Channel ID and a higher MC_CLIENT_CHANNEL_STATE Sequence number has previously been received, the MC_CHANNEL_LEAVE is ignored.

Otherwise, the client MUST leave the channel and send a new MC_CLIENT_CHANNEL_STATE frame with reason Left as requested by server.

MC_CHANNEL_LEAVE frames are formatted as shown in {{fig-mc-channel-leave-format}}.

~~~
MC_CHANNEL_LEAVE Frame {
  Type (i) = TBD-03 (experiments use 0xff3e803),
  ID Length (8),
  Channel ID (8..160),
  MC_CLIENT_CHANNEL_STATE Sequence Number (i),
  After Packet Number (i)
}
~~~
{: #fig-mc-channel-leave-format title="MC_CHANNEL_LEAVE Frame Format"}

If After Packet Number is nonzero, wait until receiving that packet or a higher valued number before leaving.

## MC_CHANNEL_INTEGRITY {#channel-integrity-frame}

MC_CHANNEL_INTEGRITY frames are sent from server to client and are used to convey packet hashes for validating the integrity of packets received over the multicast channel as described in {{packet-hashes}}.

MC_CHANNEL_INTEGRITY frames are formatted as shown in {{fig-mc-channel-integrity-format}}.

~~~
MC_CHANNEL_INTEGRITY Frame {
  Type (i) = TBD-04..TBD-05 (experiments use 0xff3e804/0xff3e805),
  ID Length (8),
  Channel ID (8..160),
  Packet Number Start (i),
  [Length (i)],
  Packet Hashes (..)
}
~~~
{: #fig-mc-channel-integrity-format title="MC_CHANNEL_INTEGRITY Frame Format"}

For type TBD-05, Length is present and is a count of packet hashes.  For TBD-04, Length is not present and the packet hashes extend to the end of the packet.

The first hash in the Packet Hashes list is a hash of a 1-RTT packet with the Channel ID equal to the Channel ID in the MC_CHANNEL_INTEGRITY frame and packet number equal to the Packet Number Start field.
Subsequent hashes refer to the packets for the channel with packet numbers increasing by 1.

Packet hashes MUST have length with an integer multiple of the length indicated by the Hash Algorithm from the Channel Properties.

See {{packet-hashes}} for a description of the packet hash calculation.

## MC_CHANNEL_ACK {#channel-ack-frame}

The MC_CHANNEL_ACK frame (types TBD-06 and TBD-07; experiments use 0xff3e806..0xff3e807) is an extension of the ACK frame defined by {{RFC9000}}. It is used to acknowledge packets that were sent on multicast channels. If the frame type is TBD-07, MC_CHANNEL_ACK frames also contain the sum of QUIC packets with associated ECN marks received on the connection up to this point.

(TODO: Is it possible to reuse the multiple packet number space version of ACK_MP from Section 12.2 of {{I-D.draft-ietf-quic-multipath}}, defining channel id as the packet number space?  at 2022-05 they're identical.)

MC_CHANNEL_ACK frames are formatted as shown in {{fig-mc-channel-ack-format}}.

~~~
MC_CHANNEL_ACK Frame {
  Type (i) = TBD-06..TBD-07 (experiments use 0xff3e806, 0xff3e807),
  ID Length (8),
  Channel ID (8..160),
  Largest Acknowledged (i),
  ACK Delay (i),
  ACK Range Count (i),
  First ACK Range (i),
  ACK Range (..) ...,
  [ECN Counts (..)],
}
~~~
{: #fig-mc-channel-ack-format title="MC_CHANNEL_ACK Frame Format"}

## MC_PATH_RESPONSE {#path-response-frame}

MC_PATH_RESPONSE frames are sent from a client to a server in the unicast connection in response to a PATH_CHALLENGE received in a channel.  Like PATH_RESPONSE but includes a channel id.

MC_PATH_RESPONSE frames are formatted as shown in {{fig-mc-path-response-format}}.

~~~
MC_PATH_RESPONSE Frame {
  Type (i) = TBD-08 (experiments use 0xffe38008),
  ID Length (8),
  Channel ID (8..160),
  Data (64)
}
~~~
{: #fig-mc-path-response-format title="MC_PATH_RESPONSE Frame Format"}

## MC_CLIENT_LIMITS {#client-limits-frame}

MC_CLIENT_LIMITS frames are formatted as shown in {{fig-mc-client-limits-format}}.

~~~
MC_CLIENT_LIMITS Frame {
  Type (i) = TBD-09 (experiments use 0xff3e809),
  Client Limits Sequence Number (i),
  Capabilities Flags(i),
  Max Aggregate Rate (i),
  Max Channel IDs (i),
  Max Joined Count (i),
}
~~~
{: #fig-mc-client-limits-format title="MC_CLIENT_LIMITS Frame Format"}

The sequence number is implicitly 0 before the first MC_CLIENT_LIMITS frame from the client, and increases by 1 each new frame that's sent.
Newer frames override older ones.

Capabilities Flags is a bit field structured as follows:

 - 0x1 is set if IPv4 channels are permitted
 - 0x2 is set if IPv6 channels are permitted

For example, a Capabilities Flags value of 3 (0x11) indicates that both IPv4 and IPv6 channels are permitted.

Max Aggregate Rate allowed across all joined channels is in Kibps.

Max Channel IDs is the count of channel IDs that can be reserved and have properties.  Retired Channel IDs don't count against this value.

Max Joined Count is the count of channels that are allowed to be joined concurrently.

## MC_CHANNEL_RETIRE {#channel-retire-frame}

MC_CHANNEL_RETIRE frames are formatted as shown in {{fig-mc-channel-retire-format}}.

~~~
MC_CHANNEL_RETIRE Frame {
  Type (i) = TBD-0a (experiments use 0xff3e80a),
  ID Length (8),
  Channel ID (8..160)
}
~~~
{: #fig-mc-channel-retire-format title="MC_CHANNEL_RETIRE Frame Format"}

Retires a channel by id.  (We can't use RETIRE_CONNECTION_ID because we don't have a coherent sequence number.)

## MC_CLIENT_CHANNEL_STATE {#client-channel-state-frame}

MC_CLIENT_CHANNEL_STATE frames are sent from client to server to report changes in the client's channel state.
Each time the channel state changes, the Client Channel State Sequence number is increased by one.
It is a state change to the channel if the server requests that a client join a channel and the client declines the join, even though no join occurs on the network.

MC_CLIENT_CHANNEL_STATE frames are formatted as shown in {{fig-mc-client-channel-state-format}}.

~~~
MC_CLIENT_CHANNEL_STATE Frame {
  Type (i) = TBD-0b (experiments use 0xff3e80b),
  Client Channel State Sequence Number (i),
  ID Length (8),
  Channel ID (8..160),
  State (i),
  Reason (0..i)
}
~~~
{: #fig-mc-client-channel-state-format title="MC_CLIENT_CHANNEL_STATE Frame Format"}

State has these defined values:

 * 0x1: Left
 * 0x2: Declined Join
 * 0x3: Joined

If State is Joined, the Reason field is absent.

If State is Left or Declined Join, the Reason field is set to one of:

 * 0x0: Unspecified Other
 * 0x1: Left as requested by server
 * 0x2: Administrative Block
 * 0x3: Protocol Error
 * 0x4: Property Violation
 * 0x5: Unsynchronized Properties
 * 0x6: ID Collision
 * 0x10: Held Down
 * 0x11: Max Idle Time Exceeded
 * 0x12: Max Rate Exceeded
 * 0x13: High Loss
 * 0x14: Spurious Traffic
 * 0x15: Max Streams Exceeded
 * 0x16: No Traffic
 * 0x1000000-0x3fffffff: Application-specific Reason

A client might receive multicast packets that it can not associate with any channel ID. If these are addressed to an (S,G) that is used for reception in one or more known channels, it MAY leave these channels with reason "Spurious traffic".

(TODO: Or should we try to reuse PATH_ABANDON and/or PATH_STATUS?  I don’t think they’re sufficient, but maybe?):
  - {{I-D.draft-ietf-quic-multipath}}
  - <https://datatracker.ietf.org/doc/html/draft-liu-multipath-quic-04#section-9.1>

(Authors comment: The things server needs to know for state changes *could* maybe be inferred from ack responses but explicit seems better, allowing for a more proactive response under strain?)

# Frames Carried in Channel Packets

MC Channels will contain normal QUIC 1-rtt data packets (see Section 17.3.1 of {{RFC9000}}) except using the Channel ID instead of a Connection ID.  The packets are protected with the keys from MC_CHANNEL_PROPERTIES for the corresponding channel.

Data packet hashes will also be sent in MC_CHANNEL_INTEGRITY frames, as keys cannot be trusted for integrity due to giving them to too many receivers, as in {{I-D.draft-krose-multicast-security}}.

The 1-rtt packets in multicast channels will have a restricted set of frames.
Since the channel is strictly 1-way server to client, the general principle is that broadcastable shared server->client data frames can be sent, but frames that make sense only for individualized connections cannot.

Permitted:

 - PADDING Frames ({{RFC9000}} Section 19.1)
 - PING Frames ({{RFC9000}} Section 19.2)
 - RESET_STREAM Frames ({{RFC9000}} Section 19.4)
 - STREAM Frames ({{RFC9000}} Section 19.8)
 - DATAGRAM Frames (both types) ({{RFC9221}} Section 4)
 - PATH_CHALLENGE Frames ({{RFC9000}} Section 19.17)
 - MC_CHANNEL_PROPERTIES
 - MC_CHANNEL_LEAVE (however, join must come over unicast?)
 - MC_CHANNEL_INTEGRITY (not for this channel, only for another)
 - MC_CHANNEL_RETIRE

Not permitted:

 - 19.3.  ACK Frames
 - 19.6.  CRYPTO Frames (crypto handshake does not happen on mc channels)
 - 19.7.  NEW_TOKEN Frames
 - Flow control is different:
   - 19.5.  STOP_SENDING Frames
   - 19.9.  MAX_DATA Frames  (flow control for mc channels is by rate)
   - 19.10. MAX_STREAM_DATA Frames
   - 19.11. MAX_STREAMS Frames
   - 19.12. DATA_BLOCKED Frames
   - 19.13. STREAM_DATA_BLOCKED Frames
   - 19.14. STREAMS_BLOCKED Frames
 - Channel ID Migration can't use the "prior to" concept, not 0-starting
   - 19.15. NEW_CONNECTION_ID Frames
   - 19.16. RETIRE_CONNECTION_ID Frames
 - 19.18. PATH_RESPONSE Frames
 - 19.19. CONNECTION_CLOSE Frames
 - 19.20. HANDSHAKE_DONE Frames
 - MC_PATH_RESPONSE
 - MC_CLIENT_LIMITS
 - MC_CLIENT_CHANNEL_STATE
 - MC_CHANNEL_ACK

# Error Codes

# Security Considerations

(Authors comment: Mostly incorporate {{I-D.draft-krose-multicast-security}}.  Anything else?

e.g. if a different legitimate quic connection says someone
else's quic multicast stream is theirs, that's maybe a problem
worth protecting against.  Maybe we need a periodic asymmetric
challenge?  I'm thinking send a public key on the multicast
channel and in the unicast channels send an individualized MAC
signed with the private key and verify it with the public key,
so that in addition to validating that the unicast server knows
the contents of the multicast packets via the hashes it supplies,
the multicast stream provides a way for the client to validate
that the unicast stream is authorized to use it for data transport
via proof they know the private key corresponding to the public
key that arrived on the multicast channel.
Note this doesn't prevent unauthorized receipt of multicast
data packts, but does prevent a quic server from lying when
claiming a multicast data channel belongs to it, preventing
legit receivers from consuming it.

alternatively, can the multicast channel just periodically say
what domain name is expected for the quic connection and get the
same crypto guarantee of a proper sender via the domain's cert,
which was already checked on the unicast channel?)

# IANA Considerations

TODO: lots

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
