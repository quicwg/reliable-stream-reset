---
title: "Reliable QUIC Stream Resets"
abbrev: "QUIC Reliable Stream Reset"
docname: draft-ietf-quic-reliable-stream-reset-latest
category: std

ipr: trust200902
area: "Transport"
workgroup: "QUIC"
keyword: Internet-Draft
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "quicwg/reliable-stream-reset"
  latest: "https://quicwg.github.io/reliable-stream-reset/draft-ietf-quic-reliable-stream-reset.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Seemann
    name: Marten Seemann
    organization: Protocol Labs
    email: martenseemann@gmail.com
 -
    fullname:
      :: 奥一穂
      ascii: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com

normative:

informative:


--- abstract

QUIC (RFC9000) defines a RESET_STREAM frame to reset a stream. When a
sender resets a stream, it stops retransmitting STREAM frames for this stream.
On the receiver side, there is no guarantee that any of the data sent on that
stream is delivered to the application.
This document defines a new QUIC frame, the RESET_STREAM_AT frame, that allows
resetting of a stream, while guaranteeing reliable delivery of stream data
up to a certain byte offset.

--- middle

# Introduction

QUIC version 1 ({{!RFC9000}}) allows streams to be reset.  When a stream is
reset, the sender doesn't retransmit stream data for the respective stream.
On the receiver side, the QUIC stack is free to surface the stream reset to the
application immediately, even if it has already received stream data for that
stream.

Applications running on top of QUIC might need to send an identifier at the
beginning of the stream in order to associate that stream with a specific
subpart of the application. For example, WebTransport
({{!WEBTRANSPORT=I-D.ietf-webtrans-http3}}) uses a variable-length-encoded
integer (as defined in QUIC version 1) to transmit the ID of the WebTransport session to
the receiver. It is desirable that the receiver is able to associate incoming
streams with their respective subpart of the application, even if the QUIC stream
is reset before the identifier at the beginning of the stream was read.

Another use-case is relaying data from an external data source. When a relay
is sending data being read from an external source and encounters an error, it
might want to use a stream reset to signal that error, at the same time making
sure that all data being read previously is delivered to the peer.

This document describes a QUIC extension defining a new frame type, the
RESET_STREAM_AT frame. This frame allows an endpoint to mark a portion at
the beginning of the stream which will then be guaranteed to be delivered to
receiver's application, even if the stream was reset.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the RESET_STREAM_AT (0x17f7586d2cb570) transport parameter
({{Section 7.4 of RFC9000}}) with an empty value. An implementation that
understands this transport parameter MUST treat the receipt of a non-empty
value as a connection error of type TRANSPORT_PARAMETER_ERROR.

In order to allow reliable stream resets in 0-RTT packets, both endpoints MUST
remember the value of this transport parameter.  If 0-RTT data is
accepted by the server, the server MUST NOT disable this extension on the
resumed connection.

# RESET_STREAM_AT Frame

Conceptually, the RESET_STREAM_AT frame is a RESET_STREAM frame with an
added Reliable Size field.

~~~
RESET_STREAM_AT Frame {
  Type (i) = 0x20,
  Stream ID (i),
  Application Protocol Error Code (i),
  Final Size (i),
  Reliable Size (i),
}
~~~

The RESET_STREAM_AT frames contain the following fields:

Stream ID:  A variable-length integer encoding of the stream ID of
      the stream being terminated.

Application Protocol Error Code:  A variable-length integer
    containing the application protocol error code
    ({{Section 20.2 of RFC9000}}) that indicates why the stream is being
    closed.

Final Size:  A variable-length integer indicating the final size of
    the stream by the RESET_STREAM sender, in units of bytes; see
    {{Section 4.5 of RFC9000}}.

Reliable Size:  A variable-length integer indicating the amount of
    data that needs to be delivered to the application even though
    the stream is reset.

If the Reliable Size is larger than the Final Size, the receiver MUST close the
connection with a connection error of type FRAME_ENCODING_ERROR.

RESET_STREAM_AT frames are ack-eliciting. When lost, they MUST be retransmitted,
unless the stream state has transitioned to "Data Recvd" or "Reset Recvd" due
to transmission and acknowledgement of other frames (see {{multiple-frames}}).

# Resetting Streams

When a sender wants to reset a stream but also deliver some bytes to the receiver,
the sender sends a RESET_STREAM_AT frame with the Reliable Size field specifying
the amount of data to be delivered.

When resetting a stream without the intent to deliver any data to the receiver,
the sender uses a RESET_STREAM frame ({{Section 3.2 of RFC9000}}). The sender
MAY also use a RESET_STREAM_AT frame with a Reliable Size of zero in place of a
a RESET_STREAM frame. These two are identical and the behavior of
RESET_STREAM frame is unchanged from the behavior described in {{!RFC9000}}.

When using a RESET_STREAM_AT frame, the initiator MUST guarantee reliable delivery
of stream data of at least Reliable Size bytes. If STREAM frames containing data
up to that byte offset are lost, the initiator MUST retransmit this data, as
described in ({{Section 13.3 of RFC9000}}). Data sent beyond that byte offset
SHOULD NOT be retransmitted.

As described in {{Section 3.2 of RFC9000}}, a stream reset signal might be
suppressed or withheld, and the same applies to a stream reset signal carried in
a RESET_STREAM_AT frame. Similary, the Reliable Size of the RESET_STREAM_AT
frame does not prevent a QUIC stack from delivering data beyond the
specified offset to the receiving application.

## Multiple RESET_STREAM_AT / RESET_STREAM frames {#multiple-frames}

The initiator MAY send multiple RESET_STREAM_AT frames for the same
stream in order to reduce the Reliable Size.  It MAY also send a RESET_STREAM
frame, which is equivalent to sending a RESET_STREAM_AT frame with a
Reliable Size of 0.

When sending multiple frames for the same stream, the initiator MUST NOT increase
the Reliable Size.  When receiving a RESET_STREAM_AT frame with a lower
Reliable Size, the receiver only needs to deliver data up the lower Reliable
Size to the application. It MUST NOT expect the delivery of any data beyond that
byte offset.

Reordering of packets might lead to a RESET_STREAM_AT frame with a higher
Reliable Size being received after a RESET_STREAM_AT frame with a lower
Reliable Size.  The receiver MUST ignore any RESET_STREAM_AT frame that
increases the Reliable Size.

When sending another RESET_STREAM_AT, RESET_STREAM or STREAM frame carrying a FIN
bit for the same stream, the initiator MUST NOT change the Application Error
Code or the Final Size. If the receiver detects a change in those fields, it
MUST close the connection with a connection error of type STREAM_STATE_ERROR.

# Implementation Guidance

In terms of transport machinery, the RESET_STREAM_AT frame is more akin to the
FIN bit than to the RESET_STREAM frame.

By sending a RESET_STREAM_AT frame, the sender commits to delivering all bytes
up to the Reliable Size. The state transitions to "Data Sent" on the sender
side, or to "Size Known" on the receiver side. Note that the flow control limit
might prevent the sender from sending all bytes up to the Reliable Size at once.

To the endpoints, the only differences from closing a stream by using the FIN
bit are:
- the offset up to which the sender commits to sending might be smaller than
  Final Size,
- this offset might get reduced by subsequent RESET_STREAM_AT frames,
- the closure is accompanied by an error code, and
- the RESET_STREAM_AT frame does not contain any payload like the STREAM frame
  with the FIN bit does.

Therefore, QUIC stacks might implement support for the RESET_STREAM_AT frame by
extending their code paths that deal with the FIN bit.

# Security Considerations

TODO Security


# IANA Considerations

## QUIC Transport Parameter

This document registers the RESET_STREAM_AT transport parameter in the "QUIC
Transport Parameters" registry established in {{Section 22.3 of RFC9000}}. The
following fields are registered:

Value:
: 0x17f7586d2cb570

Parameter Name:
: RESET_STREAM_AT

Status:
: Permanent

Specification:
: This document

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: QUIC Working Group (quic@ietf.org)

## QUIC Frame Types

This document register one new value in the "QUIC Frame Types" registry
established in {{Section 22.4 of RFC9000}}. The following fields are
registered:

Value:
: 0x20

Frame Type Name:
: RESET_STREAM_AT

Status:
: Permanent

Specification:
: This document

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: QUIC Working Group (quic@ietf.org)

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
