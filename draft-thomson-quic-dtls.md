---
title: Porting QUIC to Datagram Transport Layer Security (DTLS)
abbrev: QUIC over DTLS
docname: draft-thomson-quic-dtls-latest
date: 2016
category: std
ipr: trust200902

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
 -
    ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: martin.thomson@gmail.com


normative:
  RFC2119:
  I-D.ietf-tls-tls13:
  I-D.tsvwg-quic-protocol:
  RFC7301:

informative:
  RFC7540:
  RFC7258:
  RFC7230:
  RFC6347:
  RFC5764:
  RFC5389:
  RFC3711:
  RFC0793:

--- abstract

The QUIC experiment defines a custom security protocol.  This was necessary to
gain handshake latency improvements.  This document describes how that security
protocol might be replaced with DTLS.


--- middle

# Introduction

QUIC [I-D.tsvwg-quic-protocol] provides a multiplexed transport for HTTP
[RFC7230] semantics that provides several key advantages over HTTP/1.1 [RFC7230]
or HTTP/2 [RFC7540] over TCP [RFC0793].

The custom security protocol designed for QUIC provides critical latency
improvements for connection establishment.  Absent packet loss, most new
connections can be established with a single round trip; on subsequent
connections between the same client and server, the client can often send
application data immediately, that is, zero round trip setup.  DTLS 1.3 uses a
similar design and aims to provide the same set of improvements.

This document describes how the standardized DTLS 1.3 might serve as a security
layer for QUIC.  The same design could work for DTLS 1.2, though few of the
benefits QUIC provides would be realized due to the handshake latency in
versions of TLS prior to 1.3.

Alternative Designs:

: There are other designs that are possible; and many of these alternative
  designs are likely to be equally good.  The point of this document is to
  articulate a coherent single design.  Notes like this throughout the document
  are used describe points where alternatives were considered.


## Notational Conventions

The words "MUST", "MUST NOT", "SHOULD", and "MAY" are used in this document.
It's not shouting; when they are capitalized, they have the special meaning
defined in [RFC2119].


# Protocol Overview

QUIC [I-D.tsvwg-quic-protocol] can be separated into several modules:

1. The basic frame envelope describes the common packet layout.  This layer
   includes connection identification, version negotiation, and includes the
   indicators that allow the framing, public reset, and FEC modules to be
   identified.

2. The public reset is an unprotected frame that allows an intermediary (an
   entity that is not part of the security context) to request the termination
   of a QUIC connection.

3. The forward error correction (FEC) module provides redundant entropy that
   allows for frames to be repaired in event of loss.

4. Framing comprises most of the QUIC protocol.  Framing provides a number of
   different types of frame, each with a specific purpose.  Framing supports
   frames for both congestion management and stream multiplexing.  Framing
   additionally provides a liveness testing capability (the PING frame).

5. Crypto provides confidentiality and integrity protection for frames.  All
   frames are protected after the handshake completes on stream 1.  Prior to
   this, data is protected with the 0-RTT keys.

6. Multiplexed streams are the primary payload of QUIC.  These provide reliable,
   in-order delivery of data and are used to carry the encryption handshake and
   transport parameters (stream 1), HTTP header fields (stream 3), and HTTP
   requests and responses.  Frames for managing multiplexing include those for
   creating and destroying streams as well as flow control and priority frames.

7. Congestion management includes packet acknowledgment and other signal
   required to ensure effective use of available link capacity.

8. HTTP mapping provides an adaptation to HTTP that is based on HTTP/2.

The relative relationship of these components are pictorally represented in
{{quic-structure}}.

~~~
   +----+------+
   | HS | HTTP |
   +----+------+------------+
   |  Streams  | Congestion |
   +-----------+------------+
   |        Frames          |
   +  +---------------------+
   |  |      FEC            +--------+
   +  +---------------------+ Public |
   |  |     Crypto          | Reset  |
   +--+---------------------+--------+
   |              Envelope           |
   +---------------------------------+
   |                UDP              |
   +---------------------------------+

                             *HS = Crypto Handshake
~~~
{: #quic-structure title="QUIC Structure"}

This document describes a replacement of the cryptographic parts of QUIC.  This
includes the handshake messages that are exchanged on stream 1, plus the record
protection that is used to encrypt and authenticate all other frames.


## Handshake Overview

TLS 1.3 provides two basic handshake modes of interest to QUIC:

 * A full handshake in which the client is able to send application data after
   one round trip and the server immediately after receiving the first message
   from the client.

 * A 0-RTT handshake in which the client uses information about the server to
   send immediately.  This data can be replayed by an attacker so it MUST NOT
   carry a self-contained trigger for any non-idempotent action.

A simplified TLS 1.3 handshake with 0-RTT application data is shown in
{{tls-full}}, see [I-D.ietf-tls-tls13] for more options.

~~~
    Client                                             Server

    ClientHello
   (Finished)
   (0-RTT Application Data)
   (end_of_early_data)        -------->
                                                  ServerHello
                                         {EncryptedExtensions}
                                         {ServerConfiguration}
                                                 {Certificate}
                                           {CertificateVerify}
                                                    {Finished}
                             <--------      [Application Data]
   {Finished}                -------->

   [Application Data]        <------->      [Application Data]
~~~
{: #tls-full title="TLS Handshake with 0-RTT"}

Two additional variations on this basic handshake exchange are relevant to this
document:

 * The server can respond to a ClientHello with a HelloRetryRequest, which adds
   an additional round trip prior to the basic exchange.  This is needed if the
   server wishes to request a different key exchange key from the client.
   HelloRetryRequest might also be used to verify that the client is correctly
   able to receive packets on the address it claims to have (see
   {{source-address}}).

 * A pre-shared key mode can be used for subsequent handshakes to avoid public
   key operations.  This might be the basis for 0-RTT, even if the remainder of
   the connection is protected by a new Diffie-Hellman exchange.


# QUIC over DTLS Structure

QUIC completes its cryptographic handshake on stream 1, which means that the
negotiation of keying material happens within the QUIC protocol.  In contrast,
QUIC over DTLS uses a layered approach, where QUIC frames are exchanged as DTLS
application data.

~~~
   +-----------+
   |   HTTP    |
   +-----------+------------+
   |  Streams  | Congestion |
   +-----------+------------+
   |        Frames          |
   |           +------------+
   |           |    FEC     +--------+
   +-----------+------------+ Public |
   |         DTLS           | Reset  |
   +------------------------+--------+
   |                  UDP            |
   +---------------------------------+
~~~
{: #dtls-quic-stack title="QUIC over DTLS"}

In this design QUIC frames are exchanged as DTLS application data.  FEC and
public reset are provided as minimal protocols that are multiplexed with DTLS,
using a different value for the first octet of the UDP payload (see {{packet}}
for a sample design).

The DTLS handshake is the first data that is exchanged on a connection, though
additional QUIC-specific data might be included, see {{additions}}.

Alternative Design:

: DTLS could be used as a drop-in replacement for the handshake protocol used by
  QUIC crypto.  This would require less restructuring of QUIC.  However, that
  suggests that record protection is ultimately managed by QUIC, negating much
  of the advantage provided by choosing a layered design.  For instance,
  improvements are made to DTLS would not be immediately available to QUIC.

  A more serious problem here is the synchronization of frames with the traffic
  keys used to protect them.  Within DTLS, the transition from cleartext to
  handshake traffic keys and application traffic keys is carefully synchronized
  with the handshake.  Coordinating multiple key derivations for use in a
  drop-in design would require careful synchronization.


# Mapping of QUIC to QUIC over DTLS

Several changes to the structure of QUIC are necessary to make a layered design
practical.

These changes produce the handshake shown in {{quic-dtls-handshake}}, where QUIC
frames are exchanged as DTLS application data.

~~~
    Client                                             Server

    ClientHello
     + QUIC Setup Parameters
     + ALPN ("quic")
   (Finished)
   (Replayable QUIC Frames)
   (end_of_early_data)        -------->
                                                  ServerHello
                                         {EncryptedExtensions}
                                         {ServerConfiguration}
                                                 {Certificate}
                                           {CertificateVerify}
                                                    {Finished}
                             <--------           [QUIC Frames]
   {Finished}                -------->

   [QUIC Frames]             <------->           [QUIC Frames]
~~~
{: #quic-dtls-handshake title="QUIC over DTLS Handshake"}


The remainder of this section describes how QUIC features are
modified to fit into a layered design.


# Handshake Additions {#additions}

[I-D.tsvwg-quic-protocol] does not describe any connection-level state that
might be included by a client in a standard 1-RTT handshake to aid in connection
setup.  However, the following additions are either implemented, or might be.


## Source Address Validation {#source-address}

QUIC implementations describe a source address token.  This is an opaque blob
that a server provides to clients when they first use a given source address.
The client returns this token in subsequent messages as a return routeability
check.  That is, the client returns this token to prove that it is able to
receive packets at the source address that it claims.

Since this token is opaque and consumed only by the server, it can be included
in the TLS 1.3 configuration identifier for 0-RTT handshakes.  Servers that use
0-RTT are advised to provide new configuration identifiers after every handshake
to avoid passive linkability of connections from the same client.

A server that is under load might include the same information in the cookie
extension/field of a HelloRetryRequest. (Note: the current version of TLS 1.3
does not include this information.)


## Protocol and Version Negotiation

Application Layer Protocol Negotiation (ALPN) [RFC7301] will be used to
negotiate the version of QUIC that is used.  This is a more verbose mechanism
than the scheme used in QUIC, but it is also more capable. For example, specific
QUIC versions could be separately identified: "quic-draft-01", "quic-draft-02",
and "quic-underdamped-cc".


## QUIC-Specific Extensions {#quic-extensions}

A client describes characteristics of the transport protocol it intends to
conduct with the server in a new QUIC-specific extensions in its ClientHello.
The server uses this information to determine whether it wants to continue the
connection, request source address validation, or reject the connection.  Having
this information unencrypted permits this check to occur prior to committing the
resources needed to complete the initial key exchange.

If the server decides to complete the connection, it generates a corresponding
response and includes it in the EncryptedExtensions message.

These parameters are not confidentiality-protected when sent by the client, but
the server response is protected by the handshake traffic keys.  The entire
exchange is integrity protected once the handshake completes.

This information is not used by DTLS, but can be passed to the QUIC protocol as
initialization parmeters.


### The quic_transport_parameters Extension {#quic_transport_parameters}

The `quic_transport_parameters` extension contains a declarative set of
parameters that constrain the behaviour of a peer.  This includes the size of
the stream- and connection-level flow control windows, plus a set of optional
parameters such as the receive buffer size.

~~~
   enum {
       receive_buffer(0),
       (65535)
   } QuicTransportParameterType;

   struct {
       QuicTransportParameterType type;
       uint32 value;
   } QuicTransportParameter;

   struct {
       uint32 connection_initial_window;
       uint32 stream_initial_window;
       uint32 implicit_shutdown_timeout;
       QuicTransportParameter parameters<0..2^16-1>;
   } QuicTransportParametersExtension;
~~~

These values can be updated once the connection has started by sending an
authenticated -SOMETHING- frame on stream -SOMETHING-.

Editor's Note:

: It would appear that these settings are encapsulated in QUIC crypto messages,
  though the QUIC documents are unclear on whether a SCFG message can be sent as
  a top-level message.

The QuicTransportParameterType identifies parameters.  This is taken from a
single space that is shared by all QUIC versions (and options, see
{{quic_options}}).

This extension MUST be included if a QUIC version is negotiated.  A server MUST
NOT negotiate QUIC if this extension is not present.  This could mean that a
server might consequently send a fatal `no_application_protocol` alert.

Based on the values offered by a client a server MAY use the values in this
extension to determine whether it wants to continue the connection, request
source address validation, or reject the connection.  Since this extension is
initially unencrypted (along with ALPN), the server can use the information
prior to committing the resources needed to complete a key exchange.

If the server decides to use QUIC, this extension MUST be included in the
EncryptedExtensions message.


### The quic_options Extension {#quic_options}

The `quic_options` extension includes a list of options that can be negotiated
for a given connection.  These are set during the initial handshake and are
fixed thereafter.  These options are used to enable or disable optional features
in the protocol.

~~~
   enum {
       (65535)
   } QuicOption;

   struct {
       QuicOption options<0..2^8-2>;
   } QuicOptionsExtension;
~~~

The set of features that are supported across different versions might vary.  A
client SHOULD include all options that it is willing to use.  The server MAY
select any subset of those options that apply to the version of QUIC that it
selects.  Only those options selected by the server are available for use.

Note:

: This sort of optional behaviour seems like it could be accommodated adequately
  by defining new versions of QUIC for each experiment.  However, as an evolving
  protocol, multiple experiments need to be conducted concurrently and
  continuously, which would overload the ALPN space.  This extension provides a
  flexible way to regulate which experiments are enabled on a per-connection
  basis.

If the server decides to use any QUIC options, includes this extension in the
EncryptedExtensions message.


## Congestion Management Before Handshake Completion

In addition to the parameters for the transport, congestion feedback might need
be exchanged prior to completing a key exchange is complete.

A new QUIC congestion extension might be included by clients to include any
information.  Servers might include this extension if there is a need to use the
HelloRetryRequest also.  Once the handshake is complete, this information can be
exchanged as QUIC frames.

The use of extensions for the purpose has some potential drawbacks:

* Extensions are vulnerable to modification prior to the successful completion
  of the handshake.  (The key schedule is structured so that the handshake
  encryption keys will likely differ if extensions are modified, though this is
  not a strong guarantee.)

* Extensions are authenticated, and therefore cannot contain updated information
  if they need to be retransmitted.

* Extensions that appear prior to the EncryptedExtensions do not have any
  confidentiality protection.

{{dtls-feedback}} identifies the related issue of loss feedback mechanisms
during the handshaking phase.


# Record Protection

No major changes are required to transport frames as DTLS application data.  A
small modification permits the use of a single contiguous sequence number space,
so that QUIC can reuse DTLS sequence numbers rather than provide redundant
numbering (see {{seqno}}).


## Entropy Bit

This can be removed, and is (apparently) already have been removed from
current implementations.


## Forward Error Control (FEC)

FEC can be covered by encryption and forms part of the framing layer.  The
current protocol has some external parts, but these can be encrypted.


# Modifications to DTLS

DTLS 1.3 does not need substantial modifications in order to support the more
critical features of QUIC.  However, some additional changes might be made to
optimize DTLS for use with QUIC.

Since the work on TLS 1.3 is not yet complete, these changes might be integrated
before TLS 1.3 is completed.  Failing that, new extensions to TLS might be
considered to negotiate their use.


## Sequence Numbers {#seqno}

QUIC relies on a single sequence number space for identifying frames.  DTLS does
not provide a contiguous sequence number space as each change of traffic keys
causes sequence numbers to be reset.  This change occurs at the transition from
0-RTT application data to full application data, plus when traffic keys are
updated.

The only reason for the separate sequence number space is to support the
guarantees required by TLS.  For TLS, an attacker with access to record
protection keys might be able to force loss of a number of packets that are
protected by the next key.

DTLS does not rely on the same property, since it does not aim to secure a
contiguous stream of data.  Attacker-induced loss is not distinguishable from
other forms of loss.  For this reason, a single sequence number space is
acceptable.


## Handshake Message Loss Recovery {#dtls-feedback}

DTLS has a somewhat simplistic retransmission scheme for its handshake
datagrams.  An entire flight of handshake messages is sent repeatedly by an
endpoint until it receives a response from its peer.  In this regard, the QUIC
scheme of acknowledgments is more precise in identifying and repairing missing
datagrams.  Providing a more complete and accurate feedback mechanism would be
valuable in reducing the performance of the handshake.


## DTLS Datagram Header {#packet}

DTLS 1.2 [RFC6347] has a rather large per-packet overhead.  This overhead
includes a content type (1 octet), protocol version (2 octets), an epoch (2
octets), sequence number (6 octets) and length (2 octets), totalling 13 octets
on every record.  Of these, only the sequence number and 14 bits of the record
length are critical to the correct functioning of the protocol and both might be
reduced in size.  A single bit of the epoch is useful in allowing an endpoint to
more easily identify when keys change.

Note that a change to the packet format needs to be carefully designed if DTLS
is to remain compatible with existing use cases.  DTLS-SRTP [RFC5764] relies on
the value of the first octet of the DTLS packet not overlapping with valid
values for SRTP [RFC3711] and STUN [RFC5389].  Only values 20 through 63
inclusive are guaranteed to be available to DTLS in that context.

One possible design has a four octet header, comprising three fixed bits (001),
1 epoch bit, 12 sequence number bits, 2 zero bits, and 14 length bits.  This
format would only apply to encrypted datagrams, since the ClientHello and
ServerHello would need to be backward compatible.

~~~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0 0 1|e|sequence number (13bit)|0 0|   length (14bit)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #packet-header title="Packet Header"}

The second set of zero bits could be used for an expanded sequence number space.
However, if there is any expectation of being able to use datagrams that are
more than 4096 packets out of sequence, then adding another octet might be a
better option, even if that results in destroying word alignment.

This design allows for unprotected messages such as unprotected handshake
messages and the QUIC public reset to use values between 20 and 31 in the first
octet.

Alternative Design:

: If no multiplexing is necessary, more bits can be spent on the sequence
  number.  This might be possible if DTLS-SRTP 1.3 were modified to include a
  multiplexing header (a shim, as it were).


### No Length Option

If each UDP datagram is restricted to containing a single DTLS record, the
length field could be omitted entirely, reducing the unencrypted overhead to as
little as 2 octets.  The drawback of this approach is that it could require some
additional UDP datagrams during the handshake, since changes to keying material
can only happen when a new record is sent.

In TLS 1.3, these transitions are infrequent aside from during the initial
handshake:

* the client's first flight is only split if 0-RTT is used, in which case 4
  packets are required (this might be fewer with some proposed key schedule
  changes)

* the server's first flight contains handshake records with two different
  traffic keys and optionally application data records and therefore three
  datagrams

* the client's second flight contains two different packets


## Connection ID

The QUIC connection identifier serves to identify a connection and to allow a
server to resume an existing connection from a new client address in case of
mobility events.  However, this creates an identifier that a passive observer
[RFC7258] can use to correlate connections.

TLS 1.3 offers connection resumption using pre-shared keys, which also allows a
client to send 0-RTT application data.  This mode could be used to continue a
connection rather than rely on a publicly visible correlator.  This only
requires that servers produce a new ticket on every connection and that clients
do not resume from the same ticket more than once.

The advantage of relying on 0-RTT modes for mobility events is that this is also
more robust.  If the new point of attachment results in contacting a new server
instance - one that lacks the session state - then a fallback is easy.

The main drawback with a clean restart or anything resembling a restart is that
accumulated state can be lost.  In particular, the state of the HPACK header
compression table can be quite valuable.  Note that some QUIC implementations
use part of the connection ID to identify the server that is handling the
connection, enabling routing to that server and avoiding this sort of problem.

A lightweight state resurrection extension might be used to avoid having to
recreate any expensive state.

Editor's Note:

: It's not clear how mobility and public reset interact.  If the goal is to
  allow public reset messages to be sent by on-path entities, then using a
  connection ID to move a connection to a new path results in any entities on
  the new path not seeing the start of the connection and the nonce they need to
  generate the public reset.  A connection restart would avoid this issue.


# Security Considerations

Including data outside of the DTLS protection layer exposes endpoints to all
sorts of intermediary-initiated shenanigans.

This includes transport parameter negotiation, which might ultimately have
integrity protection.  If transport configuration is grounds for rejecting a
connection, and a client adjusts its proposed configuration in response to a
rejection, and that rejection is not authenticated (it won't be), then an
attacker can force a client to adjust their configuration.

Clients are therefore encouraged to not alter their ClientHello or its contents
in response to unauthenticated rejections or network errors.


# IANA Considerations

This document has no IANA actions.  Yet.

-- back

# Acknowledgments

Christian Huitema's knowledge of QUIC is far better than my own.  This would be
even more inaccurate and useless if not for his assistance.  This document has
variously benefited from a long series of discussions with Ryan Hamilton, Jana
Iyengar, Adam Langley, Roberto Peon, Ian Swett, and likely many others who are
merely forgotten by a faulty meat computer.
