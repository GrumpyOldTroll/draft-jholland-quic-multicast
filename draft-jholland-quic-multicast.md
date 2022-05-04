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

normative:

  I-D.draft-krose-multicast-security:
  I-D.draft-ietf-quic-multipath:
  RFC9000:
  RFC9001:
  RFC9221:

informative:

--- abstract

This document defines a multicast extension to QUIC to enable the efficient use of mullticast-capable networks to send identical data streams to many clients at once, coordinated through individual unicast QUIC connections.

--- middle

# Introduction

This document specifies an extension to QUIC version 1 {{RFC9000}} to enable the use of multicast IP transport of identical data packets for use in many individual QUIC connections.

The multicast data can only be consumed in conjunction with a unicast QUIC connection.
When support for multicast is negotiated, the server can optionally advertise existence of one or more multicast channels that contain unidirectional data streams from server to client, and the client can optionally join the multicast channel and verify from integrity data the server provides that correct data is being received, then acknowledge the data from the multicast channel(s) over the unicast connection.

Enabling this can provide large scalability benefits for popular traffic over multicast-capable networks.

This document does not define any multicast transport except server to client.

## Conventions and Definitions

{::boilerplate bcp14}

# Multicast Channel

A QUIC multicast channel (or just channel) is a one-way network path that a server can use as an alternate path to send QUIC connection data to a client.

Multicast channels are designed to leverage multicast IP and to be shared by many different connections simultaneously for unidirectional server-initiated data.
One or more servers can use the same QUIC multicast channel to send the same data to many clients, as a supplement to the individual QUIC connections between those servers and clients.

QUIC connections are defined in Section 5 of {{RFC9000}} and are not changed in this document; each connection is a shared state between a client and a server.

Channels carry only 1-RTT packets.
Packets associated with a channel contain a Channel ID in place of a Destination Connection ID.
(A Channel ID cannot be zero length.)
This adds a layer of indirection to the process described in Section 5.2 of {{RFC9000}}} for matching packets to connections upon receipt.
Incoming packets received on the network path associated with a channel use the Channel ID to associate the packet with a joined channel.

A client with a matching joined channel always has at least one connection associated with the channel.
If a client has no matching joined channel, the packet is discarded.

Since the network path for a channel is unidirectional, packets associated with a channel are acknowledged with MP_CHANNEL_ACK frames {{channel-ack-frame}} instead of with ACK frames.
Each channel has an independent packet number space.

The use of any particular channel is OPTIONAL for both the server and the client.
It is recommended that applications designed to leverage the multicast capabilities of this extension also provide graceeful degradation for endpoints that do not or cannot make use of the multicast functionality.

The server has access to all data transmitted on any multicast channel it uses, and could optionally send this data with unicast instead.

No special handling of the data is required in a client application that has enabled multicast.
A datagram or any particular bytes from a server-initiated unidirectional stream can be delivered over the unicast connection or a multicast channel transparently to the client.

Client applications should have a mechanism that disables the use of multicast on connections with enhanced privacy requirements for the privacy-related reasons covered in {{I-D.draft-krose-multicast-security}}.

# Transport Parameter {#transport-parameter}

Support for multicast extesnsions in a client is advertised by means of a QUIC transport parameter:

 * name: multicast_client_params (TBD - experiments use 0xff3e800)

If a multicast_client_params transport parameter is not included, servers MUST NOT send any frames defined in this document.  (Given that a server never sends any MC_CHANNEL_JOIN frames, the clients also will never send any frames in this document so only the client-to-server advertisement is necessary.)

The multicast_client_params parameter has the structure shown below in {{fig-transport-parameter-format}}.

~~~
multicast_client_params {
  Permit IPv4 (1),
  Permit IPv6 (1),
  Reserved (6),
  Max Aggregate Rate (i),
  Max Channel IDs (i),
  Hash Algorithms Supported (i),
  AEAD Algorithms Supported (i),
  Hash Algorithms List (16 * Hash Algorithms Supported),
  AEAD Algorithms List (16 * AEAD Algorithms Supported)
}
~~~
{: #fig-transport-parameter-format title="multicast_client_params Format"}

The Permit IPv4, Permit IPv6, Max Aggregate Rate, and Max Channel IDs fields are the same as in MC_CLIENT_LIMITS frames ({{client-limits-frame}}) and provide the initial client values.

<<<<<<< HEAD
The AEAD Algorithms List field is in order of preference (most preferred occuring first) using values from the registry below. It lists the algorithms the client is willing to use to decrypt data in multicast channels, and the server MUST NOT send a MC_CHANNEL_JOIN to this client for any channels using unsupported algorithms:

  - <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>

The Hash Algorithms List field is in order of preference (most preferred occurring first) using values from the registry below. It lists the algorithms the client is willing to use to check integrity of data in multicast channels, and the server MUST NOT send a MC_CHANNEL_JOIN to this client for any channels using unsupported algorithms:

 - <https://www.iana.org/assignments/named-information/named-information.xhtml#hash-alg>

# Extension Overview

A client has the option of refusal and the power to impose upper bound maxima on several resources (see {{flow-control}}), but otherwise its join status for all multicast channels is entirely managed by the server.

 * A client MUST NOT join a channel without receiving instructions from a server to do so
 * A client MUST leave joined channels when instructed by the server to do so
 * A client MAY leave channels or refuse to join channels, regardless of instructions from the server

## Channel Management

The client tells its server about some restrictions on resources that it is capable of processing with the initial values in the multicast_client_params transport parameter ({{transport-parameter}}) and later can update these limits with MC_CLIENT_LIMITS {{client-limits-frame}} frames. Servers ensure the set of channels the client is currently requested to join remains within these advertised client limits as covered in {{flow-control}}.

The server asks the client to join channels with MC_CHANNEL_JOIN ({{channel-join-frame}}) frames and to leave channels with MC_CHANNEL_LEAVE ({{channel-leave-frame}}) frames.

The server uses MC_CHANNEL_PROPERTIES ({{channel-properties-frame}}) frames before any join or leave frames for the channel to describe the channel properties to the client, including values the client can use to ensure the server's requests remain within the limits it has sent to the server, as well as the keys necessary to decode packets in the channel.

When the server has asked the client to join a channel, it also sends MC_CHANNEL_INTEGRITY frames ({{channel-integrity-frame}}) to enable the client to verify packet integrity before processing the packet.
A client MUST NOT decode packets for a channel for which it has not received an applicable set of MC_CHANNEL_PROPERTIES ({{channel-properties-frame}}) frames containing the full set of data required, or for which it has not received a matching packet hash in an MC_CHANNEL_INTEGRITY ({{channel-integrity-frame}}) frame.

The server ensures that in aggregate, all channels that the client has currently been asked to join and that the client has not left or declined to join fit within the limits indicated by the initial values in the transport parameter or last MC_CLIENT_LIMITS ({{client-limits-frame}}) frame the server received.

## Client Response

The client sends back information about how it has responded to the server's requests to join and leave channels in MC_CLIENT_CHANNEL_STATE ({{client-channel-state-frame}}) frames.
MC_CLIENT_CHANNEL_STATE frames are only sent for channels after the server has requested the client to join the channel, and are thereafter sent any time the state changes.

Clients that receive and decode data on a multicast channel send acknowledgements for the data on a unicast connection using MC_CHANNEL_ACK ({{channel-ack-frame}}) frames.
Channels also will periodically contain PATH_CHALLENGE ({{RFC9000}} Section 19.17) frames, which cause clients to send MC_PATH_RESPONSE ({{path-response-frame}}) frames on the unicast connection in addition to their MC_CHANNEL_ACK frames.

## Data Carried in Channels

Data transmitted in a multicast channel is encrypted with symmetric keys so that on-path observers without access to these keys cannot decode the data.
However, since potentially many receivers receive identical packets and identical keys for the multicast channel and some receivers might be malicious, the packets are also protected by MC_CHANNEL_INTEGRITY ({{channel-integrity-frame}}) frames transmitted over a separate integrity-protected path.

A client MUST NOT decode packets on a multicast channel for which it has not received a matching hash in an MC_CHANNEL_INTEGRITY frame over a different integrity-protected communication path.
The different path can be either the unicast connection or another multicast channel with packets that were verified with an earlier MC_CHANNEL_INTEGRITY frame.

## Stream Processing

Stream IDs in channels are restricted to unidirectional server initiated streams, or those with the least significant 2 bits of the stream ID equal to 3 (see {{RFC9000}} Section 2.1).

Since a server has access to all data in channels it uses, a server can always avoid stream ID collisions with the stream IDs carried in channels, and can usually (depending on the timing) avoid allowing channels to exceed the client's max_streams_uni by requesting that clients leave channels before their limits would be exceeded.

However, since clients can join later than a channel began, clients supporting the multicast extensions to QUIC should be prepared to handle stream IDs that do not begin at early values, since by the time a client joins a channel in progress the stream id count might have been increasing for a long time.
Clients should therefore begin with a high initial_max_streams_uni or send an early MAX_STREAMS type 0x13 value (see Section 19.11 of {{RFC9000}}) with a high limit.

MC_CHANNEL_PROPERTIES can provide a recommended value for max_streams_uni to allow for uninterrupted transport using the multicast channel.

Servers also can send MC_CHANNEL_STREAM_BOUNDARY_OFFSET ({{channel-stream-boundary-offset-frame}}) frames to indicate an application-layer boundary in a stream carried inside a channel.
These frames enable new clients joining a channel to start receiving application data from the indicated stream as though the stream data at that offset had an offset of 0.

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

# Stateless Reset
As clients can unilaterally stop the delivery of multicast packets by leaving the relevant Groups, channels do not need stateless reset tokens.
Clients therefore do not share the stateless reset tokens of channels with the server. Instead, if an endpoint receives packets addressed to a group IP that it can not associate with any existing channel,
it MAY take the necessary steps to prevent the reception of further such packets, without the need to signal to the server that it should stop sending.

If a server or client somehow still detect a stateless reset for a channel, they MUST ignore it.


# Implementation and Operational Considerations

# New Frames

## MC_CHANNEL_PROPERTIES {#channel-properties-frame}

An MC_CHANNEL_PROPERTIES frame (type=TBD-01) is sent from server to client, either with the unicast connection or in an existing joined multicast channel.

A server can send an update to a prior MC_CHANNEL_PROPERTIES with a new sequence number increased by one.
Any values omitted by not setting the corresponding Selector bit remain unchanged from their last value.
It is RECOMMENDED that servers set an Until Packet Number and send regular updates to the MC_CHANNEL_PROPERTIES before the packet numbers in the channel exceed that value.

A client cannot join a multicast channel without first receiving a set of MC_CHANNEL_PROPERTIES frames that set each of the values for the channel.


MC_CHANNEL_PROPERTIES frames are formatted as shown in {{fig-mc-channel-properties-format}}.

~~~
MC_CHANNEL_PROPERTIES Frame {
  Type (i) = TBD-01 (experiments use 0xff3e801),
  Properties Sequence Number (i),
  ID Length (8),
  Channel ID (8..160),
  From Packet Number (i),
  Content Field (i),
  Until Packet Number (0..i),
  [Source IP] (0..128),
  [Group IP] (0..128),
  [UDP Port] (0..16)
  [Header AEAD Algorithm] (0..16),
  [Header Key Length] (0..i),
  [Header Key] (..),
  AEAD Algorithm (0..16),
  Key Length (0..i),
  Key (..),
  Integrity Hash Algorithm (0..i),
  Max Rate (0..i),
  Max Idle Time (0..i),
  Max Streams (0..i),
  ACK Bundle Size (0..i),
}
[] = immutable values
~~~
{: #fig-mc-channel-properties-format title="MC_CHANNEL_PROPERTIES Frame Format"}

The Content Field is a bit field defining the contents of this MC_CHANNEL_PROPERTIES frame, plus some boolean values that are part of the properties of the channel.

Properties:

 - 0x0001: Key phase
 - 0x0002: IP Family

Frame Contents:

 - 0x0004: Has Until Packet Numbeer
 - 0x0008: Has Addresses
 - 0x0010: Has SSM
 - 0x0020: Has Header Key
 - 0x0040: Has Key
 - 0x0080: Has Hash Algorithm
 - 0x0100: Has Max Rate
 - 0x0200: Has Max Idle Time
 - 0x0400: Has Max Streams
 - 0x0800: Has Ack Bundle Size

The 'Has' fields in the Content Field determine presence or absence of the corresponding values in the rest of the frame, as described below.
If a field is not included in a Channel Property frame, it remains unchanged from its previous value.
If no prior value is known, requests to join the channel MUST result in a Declined Join with reason Unsynchronized Properties.

 * From Packet Number, Until Packet Number: the mutable values in this MC_CHANNEL_PROPERTIES frame apply only to packets starting at From Packet Number and continuing for all packets up to and including Until Packet Number.  If Until Packet Number is omitted it indicates the current property values for this channel have no expiration at (equivalent to the maximum value for packet numbers, or 2^62-1).  If a packet number is received outside of any prior (From,Until) range, it has no applicable channel properties and MUST be dropped.
 * Properties Sequence Number: increases by 1 each time the properties for the channel are changed by the server.  For mutable properties, the client tracks the sequence number of the MC_CHANNEL_PROPERTIES frame that set its current value, and only updates the value and the packet number range on which it's applicable if the Properties Sequnce Number is higher.

### Immutable Properties

These values cannot change during the lifetime of the channel.  If a new value is received that is not the same as a prior value, the client MUST close the connection with a PROTOCOL_ERROR.

 * IP Family: Used only when Has Addresses is set in the Content Field (ignored otherwise).  Unset indicates IPv4, Set indicates IPv6 for both Source IP (if present) and Group IP.
 * Source IP: Present if Has Addresses and Has SSM are both set in the Content Field.  The IP Address of the source of the (S,G) for the channel.  Either a 32-bit IPv4 address or a 128-bit IPv6 address, as indicated by IP Family.
 * Group IP: Present if Has Addresses is set in the Content Field.  The IP Address of the group of the (S,G) for the channel.  Either a 32-bit IPv4 address or a 128-bit IPv6 address, as indicated by IP Family.
 * UDP Port: Present if Has Addressess is set in the Content Field.  The 16-bit UDP Port of traffic for the channel.
 * Header AEAD Algorithm: Present when Has Header Key is set in the Content Field.  A value from <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>, used to protect the header fields in the channel packets.  The value MUST match a value provided in the "AEAD Algorithms List" of the transport parameter (see {{transport-parameter}}).
 * Header Key Length: Present when Has Header Key is set in the Content Field.  Provides the length of the Key field.  It MUST match a valid key length for the Header AEAD Algorithm.
 * Header Key: Present when Has Header Key is set in the Content Field.  A key for use with the Header AEAD Algorithm for protecting the header fields of 1-RTT packets in the channel as described in {{RFC9001}}.
   * **Author's Note:** I assume it’s not better to use a TLS CipherSuite because there is no KDF stage for deriving these keys (they are a strict server-to-client advertisement), so the Hash part would be unused? (<https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4>)

### Mutable Properties

These values MAY change during the lifetime of the channel.
From Packet Number and Until Packet Number are used to indicate the packet number (Section 17.1 of {{RFC9000}}) the 1-RTT packets received) over which these values are applicable.
A From Packet Number without an Until Packet Number has an unspecified termination.
If new property values appear and are different from prior values, the From Packet Number implicitly sets the Until Packet Number of the prior property value equal to one below the new From Packet Number for all the changed properties.

 * Key Phase: Used only when Has Key is set (otherwise ignored).  If set, indicates the Key Phase value is 1, or if unset indicates the Key Phase value is 0 in the 1-RTT packets that will be in use for this key (see Section 6 of {{RFC9001}}).
 * AEAD Algorithm: Present when Has Key is set in the Content Field.  A value from <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>.  The value MUST match a value provided in the "AEAD Algorithms List" of the transport parameter (see {{transport-parameter}}).
 * Key Length: present when Has Key is set in the Content Field.  Provides the length of the Key field.  It MUST match a valid key length for the AEAD Algorithm.
 * Key: present when Has Key is set in the Content Field, with length given by Key Length.  Used to protect the packet contents of 1-RTT packets for the channel as described in {{RFC9001}}.

   To maintain forward secrecy and prevent malicious clients from decrypting packets long after they have left or were removed from the unicast connection, servers SHOULD periodically send key updates using only unicast.
 * Integrity Hash Algorithm: present when Has Hash Algorithm is set in the Content Field.  The hash algorithm used in integrity frames.
   * **Author's Note:** Several candidate iana registries, not sure which one to use?  Some have only text for some possibly useful values.  For now we use the first of these:
     - <https://www.iana.org/assignments/named-information/named-information.xhtml#hash-alg>
     - <https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-18>
     - (text-only): <https://www.iana.org/assignments/hash-function-text-names/hash-function-text-names.xhtml>
 * Max Rate: Present when Has Max Rate is set in the Content Field.  The maximum rate in Kibps of the payload data for this channel.Channel data MUST NOT exceed this rate over any 5s window, if it does clients SHOULD leave the channel with reason Max Rate Exceeded.
 * Max Idle Time: Present when Has Max Idle Time is set in the Content Field.  The maximum expected idle time of the channel.  If this amount of time passes in a joined channel without data received, clients SHOULD leave the channel with reason Max Idle Time Exceeded.
 * Max Streams: Present when Has Max Streams is set in the Content Field.  The maximum stream ID that might appear in the channel.  If a client joined to this channel can raise its Max Streams limit up to or above this value it SHOULD do so, otherwise it SHOULD leave or decline join for the channel with Max Streams Exceeded.
 * ACK Bundle Size: Present when Has Ack Bundle Size is set in the Content Field.  The minimum number of ACKs a client should send in a single QUIC packet. If the max_ack_delay would force a client to send a packet that only consists of MC_CHANNEL_ACK frames, it SHOULD instead wait with sending until at least the specified number of acknowledgements have been collected. However, the Client MUST send any pending acknowledgements at least once per Max Idle Time to prevent the Server from perceiving the channel as interrupted.

## MC_CHANNEL_JOIN {#channel-join-frame}

An MC_CHANNEL_JOIN frame (type TBD-02) is sent from server to client and requests that a client join the given transport addresses and ports and process packets with the given Channel ID according to the corresponding MC_CHANNEL_PROPERTIES.

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

A client MAY perform the join if it has the sequence number of the corresponding sesssion properties and the client's limits will not be exceeded, even if the client sequence numbers are not up to date.
If the client does not join, it MUST send a MC_CLIENT_CHANNEL_STATE with "Declined Join" and a reason.

## MC_CHANNEL_LEAVE {#channel-leave-frame}

An MC_CHANNEL_LEAVE frame (type=TBD-03) is sent from server to client
Server-to-client in unicast connection or inside channel.

MC_CHANNEL_LEAVE frames are formatted as shown in {{fig-mc-channel-leave-format}}.

~~~
MC_CHANNEL_LEAVE Frame {
  Type (i) = TBD-03 (experiments use 0xff3e803),
  MC_CLIENT_CHANNEL_STATE Sequence Number (i),
  ID Length (8),
  Channel ID (8..160),
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

## MC_CHANNEL_STREAM_BOUNDARY_OFFSET {#channel-stream-boundary-offset-frame}

MC_CHANNEL_STREAM_BOUNDARY_OFFSET frames are formatted as shown in {{fig-mc-channel-stream-boundary-offset-format}}.

~~~
MC_CHANNEL_STREAM_BOUNDARY_OFFSET Frame {
  Type (i) = TBD-06 (experiments use 0xff3e806),
  ID Length (8),
  Channel ID (8..160),
  Stream ID (i),
  Stream Offset (i)
}
~~~
{: #fig-mc-channel-stream-boundary-offset-format title="MC_CHANNEL_STREAM_BOUNDARY_OFFSET Frame Format"}

Client must discard data before Stream Offset, and should start accepting stream data at Stream Offset as though it's a new stream with offset 0.

A server must ensure that data beginning at the given stream offsets could equivalently begin a new stream, and are safe for clients to start processing in order to use this.  (Well-suited for boundaries of http server push objects, for example, which otherwise would need to start a new stream per object in order to be usable by late joiners.)

## MC_CHANNEL_ACK {#channel-ack-frame}

Client->Server on unicast connection.

(TODO: Is it possible to reuse the multiple packet number space version of ACK_MP from Section 12.2 of {{I-D.draft-ietf-quic-multipath}}, defining channel id as the packet number space?  at 2022-04-12 they're identical.)

MC_CHANNEL_ACK frames are formatted as shown in {{fig-mc-channel-ack-format}}.

~~~
MC_CHANNEL_ACK Frame {
  Type (i) = TBD-07 (experiments use 0xff3e807),
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

MC_PATH_RESPONSE frames are sent from client to server in a unicast connection in response to an PATH_CHALLENGE received in a channel.  Like PATH_RESPONSE but includes a channel id.

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

Limit Support Flags is a bit field computed as follows:

 - 0x1 is set if IPv4 channels are permitted
 - 0x2 is set if IPv6 channels are permitted
 - 0x4 is set if SSM channels are permitted
 - 0x8 is set if ASM channels are permitted

For example, a Limit Support Flags value of 6 (0x110) indicates that only SSM IPv6 channels are supported.  Other kinds of channels SHOULD NOT have properties sent to this client.

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
 * 0x1000000-0x3fffffff: Application-specific Reason

A client might receive multicast packets that it can not associate with any channel ID. If these are addressed to an (S,G) that is used for reception in one or more known channels, it MAY leave these channels with reason "Spurious traffic".

(TODO: Or should we try to reuse PATH_ABANDON and/or PATH_STATUS?  I don’t think they’re sufficient, but maybe?):
  - {{I-D.draft-ietf-quic-multipath}}
  - <https://datatracker.ietf.org/doc/html/draft-liu-multipath-quic-04#section-9.1>

The things server needs to know for state changes *could* maybe be inferred from ack responses but explicit seems better, allowing for a more proactive response under strain?

# Frames Carried in Channel Packets

MC Channels will contain normal quic 1-rtt data packets (see Section 17.3.1 of {{RFC9000}}) except using the Channel ID instead of a Connection ID.  The packets are protected with the keys from MC_CHANNEL_PROPERTIES for the corresponding channel.

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
 - MC_STREAM_BOUNDARY_OFFSET
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

Mostly incorporate {{I-D.draft-krose-multicast-security}}.  Anything else?

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
(Note this doesn't prevent unauthorized receipt of multicast
data packts, but does prevent a quic server from lying when
claiming a multicast data channel belongs to it, preventing
legit receivers from consuming it.)

(alternatively, can the multicast channel just periodically say
what domain name is expected for the quic connection and get the
same crypto guarantee of a proper sender via the domain's cert,
which was already checked on the unicast channel?)

# IANA Considerations

TODO: lots

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
