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
When support for multicast is negotiated, the server can optionally advertise existence of one or more multicst channels that contain unidirectional data streams from server to client, and the client can optionally join the multicast channel and verify from integrity data the server provides that correct data is being received, then acknowledge the data from the multicast channel(s) over the unicast connection.

Enabling this can provide large scalability benefits for popular traffic over multicast-capable networks.

This document does not define any multicast transport except server to client.

## Conventions and Definitions

{::boilerplate bcp14}

# Multicast Session

A QUIC multicast session (or just session) is a one-way network path that a server can use as an alternate path to send QUIC connection data to a client.

Multicast sessions are designed to leverage multicast IP and to be shared by many different connections simultaneously for unidirectional server-initiated data.
One or more servers can use the same QUIC multicast session to send the same data to many clients, as a supplement to the individual QUIC connections between those servers and clients.

QUIC connections are defined in Section 5 of {{RFC9000}} and are not changed in this document; each connection is a shared state between a client and a server.

Sessions carry only 1-RTT packets.
Packets associated with a session contain a Session ID in place of a Destination Connection ID.
(A Session ID cannot be zero length.)
This adds a layer of indirection to the process described in Section 5.2 of {{RFC9000}}} for matching packets to connections upon receipt.
Incoming packets received on the network path associated with a session use the Session ID to associate the packet with a joined session.

A client with a matching joined session always has at least one connection associated with the session.
If a client has no matching joined session, the packet is discarded.

Since the network path for a session is unidirectional, packets associated with a session are acknowledged with MC_SESSION_ACK frames {{session-ack-frame}} instead of with ACK frames.
Each session has an independent sequence number space.

The use of any particular session is OPTIONAL for both the server and the client.
It is recommended that applications designed to leverage the multicast capabilities of this extension also provide graceeful degradation for endpoints that do not or cannot make use of the multicast functionality.

The server has access to all data transmitted on any multicast session it uses, and could optionally send this data with unicast instead.

No special handling of the data is required in a client application that has enabled multicast.
A datagram or any particular bytes from a server-initiated unidirectional stream can be delivered over the unicast connection or a multicast session transparently to the client.

Client applications should have a mechanism that disables the use of multicast on connections with enhanced privacy requirements for the privacy-related reasons covered in {{I-D.draft-krose-multicast-security}}.

# Transport Parameter {#transport-parameter}

Support for multicast extesnsions in a client is advertised by means of a QUIC transport parameter:

 * name: multicast_client_params (TBD - experiments use 0xff3e800)

If a multicast_client_params transport parameter is not included, servers MUST NOT send any frames defined in this document.  (Given that a server never sends any MC_SESSION_JOIN frames, the clients also will never send any frames in this document so only the client-to-server advertisement is necessary.)

The multicast_client_params parameter has the structure below:

multicast_client_params parameters are formatted as shown in {{fig-transport-parameter-format}}.

~~~
multicast_client_params {
  Permit IPv4 (1),
  Permit IPv6 (1),
  Reserved (6),
  Max Aggregate Rate (i),
  Max Session IDs (i),
  Hash Algorithms Supported (i),
  AEAD Algorithms Supported (i),
  Hash Algorithms List (16 * Hash Algorithms Supported),
  AEAD Algorithms List (16 * AEAD Algorithms Supported)
}
~~~
{: #fig-transport-parameter-format title="multicast_client_params Format"}

The Permit IPv4, Permit IPv6, Max Aggregate Rate, and Max Session IDs fields are the same as in MC_CLIENT_LIMITS frames ({{client-limits-frame}}) and provide the initial client values.

The AEAD Algorithms List field is in order of prefernce (most preferred occuring first) using values from the registry below. It lists the algorithms the client is willing to use to decrypt data in multicast sessions, and the server MUST NOT send a MC_SESSION_JOIN to this client for any sessions using unsupported algorithms:

  - <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>

The Hash Algorithms List field is in order of prefernce (most preferred occurring first) using values from this registry below. It lists the algorithms the client is willing to use to check integrity of data in multicast sessions, and the server MUST NOT send a MC_SESSION_JOIN to this client for any sessions using unsupported algorithms:

 - <https://www.iana.org/assignments/named-information/named-information.xhtml#hash-alg>

# Extension Overview

A client has the option of refusal and the power to impose upper bound maxima on several resources (see {{flow-control}}), but otherwise its join status for all multicast sessions is entirely managed by the server.

 * A client MUST NOT join a session without receiving instructions from a server to do so
 * A client MUST leave joined sessions when instructed by the server to do so
 * A client MAY leave sessions or refuse to join sessions, regardless of instructions from the server

## Session Management

The client tells its server about some restrictions on resources that it is capable of processing with the initial values in the multicast_client_params transport parameter ({{transport-parameter}}) and later can update these limits with MC_CLIENT_LIMITS {{client-limits-frame}} frames. Servers ensure the set of sessions the client is currently requested to join remains within these advertised client limits as covered in {{flow-control}}.

The server asks the client to join sessions with MC_SESSION_JOIN ({{session-join-frame}}) frames and to leave sessions with MC_SESSION_LEAVE ({{session-leave-frame}}) frames.

The server uses MC_SESSION_PROPERTIES ({{session-properties-frame}}) frames before any join or leave frames for the session to describe the session properties to the client, including values the client can use to ensure the server's requests remain within the limits it has sent to the server, as well as the keys necessary to decode packets in the session.

When the server has asked the client to join a session, it also sends MC_SESSION_INTEGRITY frames ({{session-integrity-frame}}) to enable the client to verify packet integrity before processing the packet.
A client MUST NOT decode packets for a session for which it has not received an applicable set of MC_SESSION_PROPERTIES ({{session-properties-frame}}) frames containing the full set of data required, or for which it has not received a matching packet hash in an MC_SESSION_INTEGRITY ({{session-integrity-frame}}) frame.

The server ensures that in aggregate, all sessions that the client has currently been asked to join and that the client has not left or declined to join fit within the limits indicated by the initial values in the transport parameter or last MC_CLIENT_LIMITS ({{client-limits-frame}}) frame the server received.

## Client Response

The client sends back information about how it has responded to the server's requests to join and leave sessions in MC_CLIENT_SESSION_STATE ({{client-session-state-frame}}) frames.
MC_CLIENT_SESSION_STATE frames are only sent for sessions after the server has requested the client to join the session, and are thereafter sent any time the state changes.

The client also sends back acknowledgements of data packets received from joined sessions with MC_SESSION_ACK ({{session-ack-frame}}) frames.

Clients that receive and decode data on a multicast session send acknowledgements for the data on a unicast session using MC_SESSION_ACK frames.
Sessions also will periodically contain PATH_CHALLENGE ({{RFC9000}} Section 19.17) frames, which cause clients to send MC_PATH_RESPONSE ({{path-response-frame}}) frames on the unicast connection in addition to their MC_SESSION_ACK frames.

## Data Carried in Sessions

Data transmitted in a multicast session is encrypted with symmetric keys so that on-path observers without access to these keys cannot decode the data.
However, since potentially many receivers receive identical packets and identical keys for the multicast session and some receivers might be malicious, the packets are also protected by MC_SESSION_INTEGRITY ({{session-integrity-frame}}) frames transmitted over a separate integrity-protected path.

A client MUST NOT decode packets on a multicast session for which it has not received a matching hash in an MC_SESSION_INTEGRITY frame over a different integrity-protected communication path.
The different path can be either the unicast connection or another multicast session with packets that were verified with an earlier MC_SESSION_INTEGRITY frame.

## Stream Processing

Stream IDs in sessions are restricted to unidirectional server initiated streams, or those with the least significant 2 bits of the stream ID equal to 3 (see {{RFC9000}} Section 2.1).

Since a server has access to all data in sessions it uses, a server can always avoid stream ID collisions with the stream IDs carried in sessions, and can usually (depending on the timing) avoid allowing sessions to exceed the client's max_streams_uni by requesting that clients leave sessions before their limits would be exceeded.

However, since clients can join later than a session began, clients supporting the multicast extensions to QUIC should be prepared to handle stream IDs that do not begin at early values, since by the time a client joins a session in progress the stream id count might have been increasing for a long time.
Clients should therefore begin with a high initial_max_streams_uni or send an early MAX_STREAMS type 0x13 value (see Section 19.11 of {{RFC9000}}) with a high limit.

MC_SESSION_PROPERTIES can provide a recommended value for max_streams_uni to allow for uninterrupted transport using the multicast session.

Servers also can send MC_SESSION_STREAM_BOUNDARY_OFFSET ({{session-stream-boundary-offset-frame}}) frames to indicate an application-layer boundary in a stream carried inside a session.
These frames enable new clients joining a session to start receiving application data from the indicated stream as though the stream data at that offset had an offset of 0.

# Flow Control {#flow-control}

The values used for unicast flow control cannot be used to limit the transmission rate of a multicast session because a single client with a low MAX_STREAM_DATA or MAX_DATA value that did not acknowledge receipt could block many other receivers if the servers had to ensure that sessions responded to each client's limits.

Instead, clients advertise resource limits that the server is responsible for staying within via MC_CLIENT_LIMITS ({{client-limits-frame}}) frames and their initial values from the transport parameter ({{transport-parameter}}).
The server advertises the expected maxima of the values that can contribute toward client resource limits within a session in MC_SESSION_PROPERTIES ({{session-properties-frame}}) frames.

If the server asks the client to join a session that would exceed the client's limits with an up-to-date Client Limit Sequence Number, the client shoud send back a MC_CLIENT_SESSION_STATE with "Declined Join" and reason "Protocol Violation".
If the server asks the client to join a session that would exceed the client's limits with an out-of-date Client Limit Sequence Number or a Session Property Sequence Number that the client has not yet seen, the client should instead send back a "Declined Join" with "Desynchronized Limit Violation".
If the actual contents sent in the session exceed the advertised limits from the MC_SESSION_PROPERTY, clients SHOULD leave the stream with a PROTOCOL_ERROR/Limit Violated state change.

# Congestion Control

The server maintains a full view of the traffic received by the client via the ACK frames coupled with the MC_SESSION_ACK ({{session-ack-frame}}) frames.

Under sustained persistent loss, the server SHOULD instruct the client such that the aggregate rate of joined sessions remains under the data rate successfully received by the client in the recent past.

# Data Integrity

TODO: import the {{I-D.draft-krose-multicast-security}} explanation for why extra integrity protection is necessary (many client have the shared key, so AEAD doesn't provide authentication against other valid clients on its own).

## Packet Hashes {#packet-hashes}

TODO: explanation and example for how to calculate the packet hash.
Note that the hash is on the unencrypted packet because it checks against a specific packet number, which is protected by AEAD.
(This approach also may help make better use of crypto hardware offload.)

# Packet Scheduling

# Implementation and Operational Considerations

# New Frames

## MC_SESSION_PROPERTIES {#session-properties-frame}

An MC_SESSION_PROPERTIES frame (type=TBD-01) is sent from server to client, either with the unicast connection or in an existing joined multicast session.

A server can send an update to a prior MC_SESSION_PROPERTIES with a new sequence number increased by one.
Any values omitted by not setting the corresponding Selector bit remain unchanged from their last value.
It is RECOMMENDED that servers set an Until Packet Number and send regular updates to the MC_SESSION_PROPERTIES before the packet numbers in the session exceed that value.

A client cannot join a multicast session without first receiving a set of MC_SESSION_PROPERTIES frames that set each of the values for the session.


MC_SESSION_PROPERTIES frames are formatted as shown in {{fig-mc-session-properties-format}}.

~~~
MC_SESSION_PROPERTIES Frame {
  Type (i) = TBD-01 (experiments use 0xff3e801),
  Session ID (i),
  Properties Sequence Number (i),
  From Packet Number (i),
  Selectors (9) {
    Has Until Packet Number (1),
    Has Addresses (1)
    Has SSM (1),
    Has Header Key (1),
    Has Key (1),
    Has Hash Algorithm (1),
    Has Max Rate (1),
    Has Max Idle Time (1),
    Has Max Streams (1),
  },
  Reserved (5),
  Key Phase (1),
  [IP Family] (1),
  Until Packet Number (0..i),
  [Source IP] (0..128),
  [Group IP] (0..128),
  [UDP Port] (0..16)
  [Header AEAD Algorithm] (0..16),
  [Header Key] (...),
  AEAD Algorithm (0..16),
  Key (...),
  Integrity Hash Algorithm (0..8),
  Max Rate (0..i),
  Max Idle Time (0..i),
  Max Streams (0..i),
}
[] = immutable values
~~~
{: #fig-mc-session-properties-format title="MC_SESSION_PROPERTIES Frame Format"}

The 'Has' fields in the Selector determine presence or absence of the corresponding values in the rest of the frame, as described below.
If a field is not included, it remains unchanged unless a different semantic is explained below.

 * From Packet Number, Until Packet Number: the mutable values in this MC_SESSION_PROPERTIES frame apply only to packets starting at From Packet Number and continuing for all packets up to and including Until Packet Number.  If Until Packet Number is omitted it indicates the current property values for this session have no expiration at (equivalent to the maximum value for packet numbers, or 2^62-1).  If a packet number is received outside of any prior (From,Until) range, it has no applicable session properties and MUST be dropped.
 * Properties Sequence Number: increases by 1 each time the properties for the session are changed by the server.  For mutable properties, the client tracks the sequence number of the MC_SESSION_PROPERTIES frame that set its current value, and only updates the value and the packet number range on which it's applicable if the Properties Sequnce Number is higher.

### Immutable Properties

These values cannot change during the lifetime of the session.  If a new value is received that is not the same as a prior value, the client MUST close the connection with a PROTOCOL_ERROR.

 * IP Family: Always present, but used only when Has Addresses=1 (ignored otherwise).  0 indicates IPv4, 1 indicates IPv6 for both Source IP (if present) and Group IP.
 * Source IP: Present if Has Addresses=1 and Has SSM=1.  The IP Address of the source of the (S,G) for the session's channel.  Either a 32-bit IPv4 address or a 128-bit IPv6 address, as indicated by IP Family.
 * Group IP: Present if Has Addresses=1.  The IP Address of the group of the (S,G) for the session's channel.  Either a 32-bit IPv4 address or a 128-bit IPv6 address, as indicated by IP Family.
 * UDP Port: Present if Has Addressess=1.  The 16-bit UDP Port of traffic for the session's channel.
 * Header AEAD Algorithm: Present when Has Header Key=1.  a value from <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>, used to protect the header fields in the session packets.  The value MUST match a value provided in the "AEAD Algorithms List" of the transport parameter (see {{transport-parameter}}).
 * Header Key: Present when Has Header Key=1.  A key with length and semantics determined by the Header AEAD Algorithm.
> **Author's Note:** I assume it’s not better to use a TLS CipherSuite because there is no KDF stage for deriving these keys (they are a strict server-to-client advertisement), so the Hash part would be unused? (<https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4>)

### Mutable Properties

These values MAY change during the lifetime of the session.
From Packet Number and Until Packet Number are used to indicate the packet number (Section 17.1 of {{RFC9000}}) the 1-RTT packets received) over which these values are applicable.
A From Packet Number without an Until Packet Number has an unspecified termination.
If new property values appear and are different from prior values, the From Packet Number implicitly sets the Until Packet Number of the prior property value equal to the new From Packet Number.

 * Key Phase: Always present, used only if Has Key is set (otherwise ignored).  The key phase value in the 1-RTT packet that will be in use for this key (see Section 6 of {{RFC9001}}).
 * AEAD Algorithm: Present if Has Key is set.  A value from <https://www.iana.org/assignments/aead-parameters/aead-parameters.xhtml>.  The value MUST match a value provided in the "AEAD Algorithms List" of the transport parameter (see {{transport-parameter}}).
 * Key: present if and only if Has Key is set, with length determined by the AEAD Algorith value.  Used to protect the packet contents of 1-RTT packets for the session as described in {{RFC9001}}.
 * Integrity Hash Algorithm: the hash algorithm used in integrity frames
> **Author's Note:** Several candidate iana registries, not sure which one to use?  Some have only text for some possibly useful values:
   - <https://www.iana.org/assignments/named-information/named-information.xhtml#hash-alg>
   - <https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-18>
   - (text-only): <https://www.iana.org/assignments/hash-function-text-names/hash-function-text-names.xhtml>
 * Max Rate: The maximum rate in Kibps of the payload data for this session.Session data MUST NOT exceed this rate over any 5s window, if it does clients SHOULD leave the session with reason Max Rate Exceeded.
 * Max Idle Time: The maximum expected idle time of the session.  If this amount of time passes in a joined session without data received, clients SHOULD leave the session with reason Max Idle Time Exceeded.
 * Max Streams: The maximum stream ID that might appear in the session.  If a client joined to this session can raise its Max Streams limit up to or above this value it SHOULD do so, otherwise it SHOULD leave or decline join for the session with Max Streams Exceeded.

## MC_SESSION_JOIN {#session-join-frame}

An MC_SESSION_JOIN frame (type TBD-02) is sent from server to client and requests that a client join the given transport addresses and ports and process packets with the given Session ID according to the corresponding MC_SESSION_PROPERTIES.

MC_SESSION_JOIN frames are formatted as shown in {{fig-mc-session-join-format}}.

~~~
MC_SESSION_JOIN Frame {
  Type (i) = TBD-02 (experiments use 0xff3e802),
  Session ID (i),
  MC_CLIENT_LIMITS Sequence Number (i),
  MC_CLIENT_SESSION_STATE Sequence Number (i),
  MC_SESSION_PROPERTIES Sequence Number (i)
~~~
{: #fig-mc-session-join-format title="MC_SESSION_JOIN Frame Format"}

The sequence numbers are present to allow the client to distinguish between a broken sender that has performed an illegal action and an instruction that's based on updates that are out of sync (either one or more missing updates to MC_SESSION_PROPERTIES not yet received by the client or one or more missing updates to MC_CLIENT_LIMITS or MC_CLIENT_SESSION_STATE not yet received by the server).

A client SHOULD perform the join if it has the sequence number of the corresponding sesssion properties and the client's limits will not be exceeded, even if the client sequence numbers are not up to date.
If the client does not join, it MUST send a MC_CLIENT_SESSION_STATE with "Declined Join" and a reason.

## MC_SESSION_LEAVE {#session-leave-frame}

An MC_SESSION_LEAVE frame (type=TBD-03) is sent from server to client
Server-to-client in unicast connection or inside session.

MC_SESSION_LEAVE frames are formatted as shown in {{fig-mc-session-leave-format}}.

~~~
MC_SESSION_LEAVE Frame {
  Type (i) = TBD-03 (experiments use 0xff3e803),
  Session ID (i),
  After Packet Number (i)
~~~
{: #fig-mc-session-leave-format title="MC_SESSION_LEAVE Frame Format"}

If After Packet Number is nonzero, wait until receiving that packet or a higher valued number before leaving.

## MC_SESSION_INTEGRITY {#session-integrity-frame}

MC_SESSION_INTEGRITY frames are formatted as shown in {{fig-mc-session-integrity-format}}.

~~~
MC_SESSION_INTEGRITY Frame {
  Type (i) = TBD-04..TBD-05 (experiments use 0xff3e804/0xff3e805),
  Session ID (i),
  Packet Number Start (i),
  [Length (i)],
  Packet Hashes (..)
}
~~~
{: #fig-mc-session-integrity-format title="MC_SESSION_INTEGRITY Frame Format"}

For type TBD-05, Length is present and is a count of packet hashes.  For TBD-04, Length is not present and the packet hashes extend to the end of the packet.

The first hash in the Packet Hashes list is a hash of a 1-RTT packet with the Session ID equal to the Session ID in the MC_SESSION_INTEGRITY frame and packet number equal to the Packet Number Start field.
Subsequent hashes refer to the packets for the session with packet numbers increasing by 1.

Packet hashes MUST have length with an integer multiple of the length indicated by the Hash Algorithm from the Session Properties.

See {{packet-hashes}} for a description of the packet hash calculation.

## MC_SESSION_STREAM_BOUNDARY_OFFSET {#session-stream-boundary-offset-frame}

MC_SESSION_STREAM_BOUNDARY_OFFSET frames are formatted as shown in {{fig-mc-session-stream-boundary-offset-format}}.

~~~
MC_SESSION_STREAM_BOUNDARY_OFFSET Frame {
  Type (i) = TBD-06 (experiments use 0xff3e806),
  Session ID (i),
  Stream ID (i),
  Stream Offset (i)
~~~
{: #fig-mc-session-stream-boundary-offset-format title="MC_SESSION_STREAM_BOUNDARY_OFFSET Frame Format"}

Client must discard data before Stream Offset, and should start accepting stream data at Stream Offset as though it's a new stream with offset 0.

A server must ensure that data beginning at the given stream offsets could equivalently begin a new stream, and are safe for clients to start processing in order to use this.  (Well-suited for boundaries of http server push objects, for example, which otherwise would need to start a new stream per object in order to be usable by late joiners.)

## MC_SESSION_ACK {#session-ack-frame}

Client->Server on unicast connection.

(TODO: Is it possible to reuse the multiple packet number space version of ACK_MP from Section 12.2 of {{I-D.draft-ietf-quic-multipath}}, defining session id as the packet number space?  at 2022-04-12 they're identical.)

MC_SESSION_ACK frames are formatted as shown in {{fig-mc-session-ack-format}}.

~~~
  MC_SESSION_ACK Frame {
    Type (i) = TBD-07 (experiments use 0xff3e807),
    Session ID (i),
    Largest Acknowledged (i),
    ACK Delay (i),
    ACK Range Count (i),
    First ACK Range (i),
    ACK Range (..) ...,
    [ECN Counts (..)],
  }
~~~
{: #fig-mc-session-ack-format title="MC_SESSION_ACK Frame Format"}


## MC_PATH_RESPONSE {#path-response-frame}

MC_PATH_RESPONSE frames are sent from client to server in a unicast connection in response to an PATH_CHALLENGE received in a session.  Like PATH_RESPONSE but includes a session id.

MC_PATH_RESPONSE frames are formatted as shown in {{fig-mc-path-response-format}}.

~~~
MC_PATH_RESPONSE Frame {
  Type (i) = TBD-08 (experiments use 0xffe38008)
  Session ID (i),
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
  Permit IPv4 (1),
  Permit IPv6 (1),
  Reserved (6),
  Max Aggregate Rate (i),
  Max Session IDs (i)
~~~
{: #fig-mc-client-limits-format title="MC_CLIENT_LIMITS Frame Format"}

The sequence number is implicitly 0 before the first MC_CLIENT_LIMITS frame from the client, and increases by 1 each new frame that's sent.
Newer frames override older ones.

Max Aggregate Rate allowed across all joined sessions is in Kibps.

## MC_SESSION_RETIRE {#session-retire-frame}

MC_SESSION_RETIRE frames are formatted as shown in {{fig-mc-session-retire-format}}.

~~~
MC_SESSION_RETIRE Frame {
  Type (i) = TBD-0a (experiments use 0xff3e80a),
  Session IDs (i)
~~~
{: #fig-mc-session-retire-format title="MC_SESSION_RETIRE Frame Format"}

Retires a session by id.  (We can't use RETIRE_CONNECTION_ID because we don't have a coherent sequence number.)

## MC_CLIENT_SESSION_STATE {#client-session-state-frame}

MC_CLIENT_SESSION_STATE frames are formatted as shown in {{fig-mc-client-session-state-format}}.

~~~
MC_CLIENT_SESSION_STATE Frame {
  Type (i) = TBD-0b (experiments use 0xff3e80b),
  Client Session State Sequence Number (i),
  State (i),
  Reason (0..i)
~~~
{: #fig-mc-client-session-state-format title="MC_CLIENT_SESSION_STATE Frame Format"}

State has possible values:

 * 0x0: Left
 * 0x1: Declined Join
 * 0x2: Joined

For Left, Reason must be set to one of:

 * 0x0: Left as requested by server
 * 0x1: Protocol Error
 * 0x2: Property Violation
 * 0x3: High Loss
 * 0x4: Exceeded max idle time
 * 0x5: Administrative change
 * 0x6: other

For Declined Join, Reason must be set to one of:

 * Protocol Violation (limits exceeded, unsupported ciphersuite, etc)
 * held-down (prior session failures, administrative block)

(TODO: Should we include a loggable string in this?)

(TODO: Or should we try to reuse PATH_ABANDON and/or PATH_STATUS?  I don’t think they’re sufficient, but maybe?):
  - {{I-D.draft-ietf-quic-multipath}}
  - <https://datatracker.ietf.org/doc/html/draft-liu-multipath-quic-04#section-9.1>

The things server needs to know for state changes *could* maybe be inferred from ack responses but explicit seems better, allowing for a more proactive response under strain?

# Frames Carried in Session Packets

MC Sessions will contain normal quic 1-rtt data packets (see Section 17.3.1 of {{RFC9000}}) except using the Session ID instead of a Connection ID.  The packets are protected with the keys from MC_SESSION_PROPERTIES for the corresponding session.

Data packet hashes will also be sent in MC_SESSION_INTEGRITY frames, as keys cannot be trusted for integrity due to giving them to too many receivers, as in {{I-D.draft-krose-multicast-security}}.

The 1-rtt packets in multicast sessions will have a restricted set of frames.
Since the session is strictly 1-way server to client, the general principle is that broadcastable shared server->client data frames can be sent, but frames that make sense only for individualized connections cannot.

Permitted:

 - PADDING Frames ({{RFC9000}} Section 19.1)
 - PING Frames ({{RFC9000}} Section 19.2)
 - RESET_STREAM Frames ({{RFC9000}} Section 19.4)
 - STREAM Frames ({{RFC9000}} Section 19.8)
 - DATAGRAM Frames (both types) ({{RFC9221}} Section 4)
 - PATH_CHALLENGE Frames ({{RFC9000}} Section 19.17)
 - MC_SESSION_PROPERTIES
 - MC_SESSION_LEAVE (however, join must come over unicast?)
 - MC_SESSION_INTEGRITY (not for this session, only for another)
 - MC_STREAM_BOUNDARY_OFFSET
 - MC_SESSION_RETIRE

Not permitted:

 - 19.3.  ACK Frames
 - 19.6.  CRYPTO Frames (crypto handshake does not happen on mc sessions)
 - 19.7.  NEW_TOKEN Frames
 - Flow control is different:
   - 19.5.  STOP_SENDING Frames
   - 19.9.  MAX_DATA Frames  (flow control for mc sessions is by rate)
   - 19.10. MAX_STREAM_DATA Frames
   - 19.11. MAX_STREAMS Frames
   - 19.12. DATA_BLOCKED Frames
   - 19.13. STREAM_DATA_BLOCKED Frames
   - 19.14. STREAMS_BLOCKED Frames
 - Session ID Migration can't use the "prior to" concept, not 0-starting
   - 19.15. NEW_CONNECTION_ID Frames
   - 19.16. RETIRE_CONNECTION_ID Frames
 - 19.18. PATH_RESPONSE Frames
 - 19.19. CONNECTION_CLOSE Frames
 - 19.20. HANDSHAKE_DONE Frames
 - MC_PATH_RESPONSE
 - MC_CLIENT_LIMITS
 - MC_CLIENT_SESSION_STATE
 - MC_SESSION_ACK

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
key that arrived on the multicast session.
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
