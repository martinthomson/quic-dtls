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
  RFC7230:
  RFC6347:
  RFC5764:
  RFC5389:
  RFC3711:

--- abstract

The QUIC experiment defines a custom security protocol.  This was necessary to
gain handshake latency improvements.  This document describes how that security
protocol might be replaced with DTLS.


--- middle

# Introduction

QUIC [I-D.tsvwg-quic-protocol] provides a multiplexed transport for HTTP
[RFC7230] semantics that provides several key advantages over TCP-based
transports like HTTP/1.1 [RFC7230] and HTTP/2 [RFC7540].

The custom security protocol designed for QUIC provides critical latency
improvements for connection establishment.  Absent packet loss, most new
connections can be established with a single round trip; on subseuqent
connections between the same client and server, the client can often send
application data immediately, that is, zero round trip setup.  DTLS 1.3 uses a
similar design and aims to provide the same set of improvements.

This document describes how the standardized DTLS 1.3 might serve as a security
layer for QUIC.  The same design could work for DTLS 1.2, though few of the
benefits QUIC provides would be realized due to the handshake latency in
versions of TLS prior to 1.3.

Alternative designs:

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
   frames are protected after the handshake completes on stream 1.

6. Multiplexed streams are the primary payload of QUIC.  These provide reliable,
   in-order delivery of data and are used to carry the encryption handshake
   (stream 1), HTTP header fields (stream 3), and HTTP requests and responses.
   Frames for managing multiplexing include those for creating and destroying
   streams as well as flow control and priority frames.

7. Congestion management includes packet acknowledgment and other signal
   required to ensure effective use of available link capacity.

8. HTTP mapping provides an adaptation to HTTP that is based on HTTP/2.

The relative relationship of these components are pictorally represented in
{{quic-structure}}.

~~~
   +----+------+
   | HS | HTTP |
   +----+------+------------+-------+--------+
   |  Streams  | Congestion |       |        |
   +-----------+------------+       |        |
   |        Frames          |  FEC  | Public |
   +  +---------------------+       | Reset  |
   |  |     Crypto          |       |        |
   +--+---------------------+-------+--------+
   |              Envelope                   |
   +-----------------------------------------+
   |                UDP                      |
   +-----------------------------------------+

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
                             <--------              {Finished}
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
   key operations.  (FFS: 0-RTT for PSK seems feasible, but it isn't clearly
   defined right now).


# QUIC over DTLS Structure

QUIC completes its cryptographic handshake on stream 1, which means that the
negotiation of keying material happens within the QUIC protocol.  In contrast,
QUIC over DTLS uses a layered approach, where QUIC frames are exchanged as DTLS
application data.

~~~
   +-----------+
   |   HTTP    |
   +-----------+------------+-------+--------+
   |  Streams  | Congestion |       |        |
   +-----------+------------+       |        |
   |        Frames          |  FEC  | Public |
   +------------------------+       | Reset  |
   |         DTLS           |       |        |
   +------------------------+-------+--------+
   |                  UDP                    |
   +-----------------------------------------+
~~~
{: #dtls-quic-stack title="QUIC over DTLS"}

In this design QUIC frames are exchanged as DTLS application data.  FEC and
public reset are provided as minimal protocols that are multiplexed with DTLS,
using a different value for the first octet of the UDP payload (see {{packet}}
for a sample design).

The DTLS handshake is the first data that is exchanged on a connection, though
additional QUIC-specific data might be included, see {{additions}}.

Issue:

: FEC and its role in this protocol in this are very poorly understood by this
  author.  This assumes that FEC packets are not protected so that an
  intermediary might repair an opaque QUIC exchange, but that might be
  completely inappropriate in practice.  A more principled approach might be to
  avoid designing FEC here and instead to examine the use of generic FEC for
  encrypted protocols separately.  That would potentially yield improvements for
  other protocols that build on unreliable lower layers.

Alternative Design:

: DTLS could be used as a drop-in replacement for the handshake protocol used by
  QUIC crypto.  This would require less restructuring of QUIC.  However, that
  suggests that record protection is ultimately managed by QUIC.  Such a design
  would require the definition of a mapping from the DTLS record protection to
  the QUIC one, which could ultimately not include new improvements to DTLS that
  alter the record protection.


# Mapping of QUIC to QUIC over DTLS

Several changes to the structure of QUIC are necessary to make a layered design
practical.  The remainder of this section describes how QUIC features are
modified to fit into a layered design.


## Handshake Additions {#additions}

[I-D.tsvwg-quic-protocol] does not describe any connection-level state that
might be included by a client in a standard 1-RTT handshake to aid in connection
setup.  However, the following additions are either implemented, or might be.

### Source Address Validation {#source-address}

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


### Congestion Management Before Handshake Completion

While many congestion management parameters will be exchanged in encrypted
packets, this is not possible until the key exchange is complete.  For 1-RTT
handshakes, it is highly desirable to start congestion management as soon as
possible.

A new QUIC congestion extension might be included by clients to include any
information.  Servers do not require this extension unless there is a need to
use the HelloRetryRequest, in which case the server can include the same
extension in that message.

Since little information is available on what might be needed here, the only
recommendation this document can make is to say that encoding a QUIC frame into
the extension might be advisable.

The initial message from the client will not have confidentiality protection,
and nor will subsequent messages if the server uses HelloRetryRequest for any
reason.  These messages will also be subject to modification, only being
authenticated if the handshake completes successfully.  This could greatly limit
what information can be included in these early parts of the handshake.


## Record Protection

No changes are required to transport frames as DTLS application data.


## Protocol and Version Negotiation

Application Layer Protocol Negotiation (ALPN) [RFC7301] will be used to
negotiate the version of QUIC that is used.  This is a more verbose mechanism
than the scheme used in QUIC, but it is also more capable.


## Connection ID

The connection identifier serves to.  TLS 1.3 offers connection resumption using
pre-shared keys, which might offer a 0-RTT option (open issue).  This mode is
preferred in this case; failing that, a complete 0-RTT handshake could be used.


## Entropy Bit

It's not clear what value this provides, or even how an implementation might
decide to set it.


## Forward Error Control (FEC)

Not enough is known about the QUIC FEC scheme to propose a solution.


# Modifications to DTLS

DTLS 1.3 does not need substantial modifications in order to support the more
critical features of QUIC.  However, some additional changes might be made to
optimize DTLS for use with QUIC.

Since the work on TLS 1.3 is not yet complete, these changes might be integrated
before TLS 1.3 is completed.  Failing that, new extensions to TLS might be
considered to negotiate their use.


## DTLS Datagram Header {#packet}

DTLS 1.2 [RFC6347] has a rather large per-packet overhead.  This overhead
includes a content type (1 octet), protocol version (2 octets), an epoch (2
octets), sequence number (6 octets) and length (2 octets), totalling 13 octets
on every record.  Of these, only the sequence number and 14 bits of the record
length are critical to the correct functioning of the protocol and both might be
reduced in size.

Note that a change to the packet format needs to be carefully designed if DTLS
is to remain compatible with existing use cases.  DTLS-SRTP [RFC5764] relies on
the value of the first octet of the DTLS packet not overlapping with valid
values for SRTP [RFC3711] and STUN [RFC5389].  Only values 20 through 63
inclusive are guaranteed to be available to DTLS in that context.

One possible design has a four octet header, comprising three fixed bits (001),
13 sequence number bits, 2 zero bits, and 14 length bits.  This format would
only apply to encrypted datagrams, since the ClientHello and ServerHello would
need to be backward compatible.

~~~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0 0 1| sequence number (13bit) |0 0|   length (14bit)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #packet-header title="Packet Header"}

Note that the second set of zero bits could be used for an expanded sequence
number space.  However, if there is any expectation of being able to use
datagrams that are more than 8192 packets out of sequence, then adding another
octet might be a better option, even if that results in destroying word
alignment.

This design allows for unprotected messages such as unprotected handshake
messages and the QUIC public reset to use values between 20 and 31 in the first
octet.  If FEC is used to repair protected packets (a detail that
[I-D.tsvwg-quic-protocol] is unclear on), then FEC packets can use a first byte
in this range as well.

Alternative Design:

: If a single DTLS record is contained in each UDP datagram, then the length
  field might be omitted entirely.


# Security Considerations

Including data outside of the DTLS protection layer exposes endpoints to all
sorts of intermediary-initiated shenanigans.


# IANA Considerations

This document has no IANA actions.