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
  WEBTRANSPORT: I-D.ietf-webtrans-http3

informative:


--- abstract

QUIC defines a RESET_STREAM frame to abort sending on a stream. When a sender
resets a stream, it also stops retransmitting STREAM frames for this stream in
the of packet loss. On the receiving side, there is no guarantee that any data
sent on that stream is delivered.

This document defines a new QUIC frame, the RESET_STREAM_AT frame, that allows
resetting a stream, while guaranteeing delivery of stream data up to a certain
byte offset.

--- middle

# Introduction

QUIC version 1 ({{!RFC9000}}) allows streams to be reset.  When a stream is
reset, the sender doesn't retransmit stream data for the respective stream. On
the receiving side, the QUIC stack is free to surface the stream reset to the
application immediately, without providing any stream data it has received for
that stream.

Some applications running on top of QUIC send an identifier at the beginning of
the stream in order to associate that stream with a specific subcomponent of the
application. For example, WebTransport ({{!WEBTRANSPORT}}) uses a
variable-length encoded integer to associate a stream with a particular
WebTransport session. It is desirable that the receiver is able to associate
incoming streams with their respective subcomponent of the application, even if
the QUIC stream is reset before the identifier at the beginning of the stream
was read by the application.

Another use case is relaying data from an external data source. When a relay is
sending data being read from an external source and encounters an error, it
might want to use a stream reset to signal that error, while at the same time
guaranteeing that all data received from the source is delivered to the peer.

This document describes a QUIC extension defining a new frame type, the
RESET_STREAM_AT frame. This frame allows an endpoint to mark a portion at the
beginning of the stream which will then be reliably delivered, even if the
stream was reset.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the reliable_stream_reset (0x17f7586d2cb570) transport parameter
({{Section 7.4 of RFC9000}}) with an empty value. An implementation that
understands this transport parameter MUST treat the receipt of a non-empty value
as a connection error of type TRANSPORT_PARAMETER_ERROR.

When using 0-RTT, both endpoints MUST remember the value of this transport
parameter. This allows use of this extension in 0-RTT packets. When the server
accepts 0-RTT data, the server MUST NOT disable this extension on the resumed
connection.

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
{: #reset-stream-at-format title="RESET_STREAM_AT Frame Format"}

The RESET_STREAM_AT frame contains the following fields:

Stream ID:

: A variable-length integer encoding of the stream ID of the stream being
  terminated.

Application Protocol Error Code:

: A variable-length integer containing the application protocol error code
  ({{Section 20.2 of RFC9000}}) that indicates why the stream is being closed.

Final Size:

: A variable-length integer indicating the final size of the stream by the
  sender, in units of bytes; see {{Section 4.5 of RFC9000}}.

Reliable Size:

: A variable-length integer indicating the amount of data that needs to be
  delivered to the application even though the stream is reset.

If the Reliable Size is larger than the Final Size, the receiver MUST close the
connection with a connection error of type FRAME_ENCODING_ERROR.

RESET_STREAM_AT frames are ack-eliciting, and MUST only be sent in the
application data packet number space. When lost, they MUST be retransmitted,
unless the stream state has transitioned to "Data Recvd" or "Reset Recvd" due to
transmission and acknowledgement of other frames (see {{multiple-frames}}).

# Resetting Streams

A sender that wants to reset a stream but also deliver some bytes to the
receiver sends a RESET_STREAM_AT frame with the Reliable Size field specifying
the amount of data to be delivered.

When using a RESET_STREAM_AT frame, the initiator MUST guarantee reliable
delivery of stream data of at least Reliable Size bytes. If STREAM frames
containing data up to that byte offset are lost, the initiator MUST retransmit
this data, as described in {{Section 13.3 of RFC9000}}. Data sent beyond that
byte offset SHOULD NOT be retransmitted.

As described in {{Section 3.2 of RFC9000}}, a stream reset signal might be
suppressed or withheld, and the same applies to a stream reset signal carried in
a RESET_STREAM_AT frame. Similary, the Reliable Size of the RESET_STREAM_AT
frame does not prevent a QUIC stack from delivering data beyond the specified
offset to the receiving application.

Note that a Reliable Size value of zero is valid. A RESET_STREAM_AT frame with
this value is logically equivalent to a RESET_STREAM frame ({{Section 3.2 of
RFC9000}}). When resetting a stream without the intent to deliver any data to
the receiver, the sender MAY use either RESET_STREAM or
RESET_STREAM_AT with a Reliable Size of zero.

## Multiple RESET_STREAM_AT / RESET_STREAM frames {#multiple-frames}

The initiator MAY send multiple RESET_STREAM_AT frames for the same stream in
order to reduce the Reliable Size.  It MAY also send a RESET_STREAM frame, which
is equivalent to sending a RESET_STREAM_AT frame with a Reliable Size of 0. When
reducing the Reliable Size, the sender MUST retransmit the RESET_STREAM_AT frame
carrying the smallest Reliable Size as well as stream data up to that size,
until all acknowledgements for the stream data and the RESET_STREAM_AT frame are
received.

When sending multiple RESET_STREAM_AT or RESET_STREAM frames for the same
stream, the initiator MUST NOT increase the Reliable Size.

When receiving a RESET_STREAM_AT frame with a lower Reliable Size, the receiver
only needs to deliver data up the lower Reliable Size to the application. It
MUST NOT expect the delivery of any data beyond that byte offset.

Reordering of packets might lead to a RESET_STREAM_AT frame with a higher
Reliable Size being received after a RESET_STREAM_AT frame with a lower
Reliable Size.  The receiver MUST ignore any RESET_STREAM_AT frame that
increases the Reliable Size.

When sending another RESET_STREAM_AT, RESET_STREAM or STREAM frame carrying a FIN
bit for the same stream, the initiator MUST NOT change the Application Error
Code or the Final Size. If the receiver detects a change in those fields, it
MUST close the connection with a connection error of type STREAM_STATE_ERROR.

## Stream States {#stream-states}

In terms of stream state transitions ({{Section 3 of RFC9000}}), the effect of a
RESET_STREAM_AT frame is equivalent to that of the FIN bit. Both the
RESET_STREAM_AT frame and the FIN bit on a STREAM frame serve the same role:
signaling the amount of data to be delivered.

On the sending side, when the first RESET_STREAM_AT frame is sent, the sending
part of the stream enters the "Data Sent" state. Once the RESET_STREAM_AT frame
carrying the smallest Reliable Size and all stream data up to that byte offset
have been acknowledged, the sending part of the stream enters the "Data Recvd"
state. The transition from "Data Sent" to "Data Recvd" happens immediately if
the application resets a stream and all bytes up to the specified Reliable Size
have already been sent and acknowledged. Conversely, the transition might take
multiple network roundtrips or require additional flow control credit issued by
the receiver.

On the receiving side, when a RESET_STREAM_AT frame is received, the receiving
part of the stream enters the "Size Known" state. Once all data up to the
smallest Reliable Size have been received, it enters the "Data Recvd" state.
Similarly to the sending side, transition from "Size Known" to "Data Recvd"
might happen immediately or involve issuance of additional flow control credit.

# Implementation Guidance

In terms of transport machinery, the RESET_STREAM_AT frame is more akin to the
FIN bit than to the RESET_STREAM frame (see {{stream-states}}). By sending a
RESET_STREAM_AT frame, the sender commits to delivering all bytes up to the
Reliable Size.

To the endpoints, the main differences from closing a stream by using the FIN
bit are:

- the offset up to which the sender commits to sending might be smaller than
  Final Size,
- this offset might get reduced by subsequent RESET_STREAM_AT frames, and
- the closure is accompanied by an error code.


# Security Considerations

TODO Security


# IANA Considerations

## QUIC Transport Parameter

This document registers the reliable_stream_reset transport parameter in the
"QUIC Transport Parameters" registry established in {{Section 22.3 of RFC9000}}.
The following fields are registered:

Value:
: 0x17f7586d2cb570

Parameter Name:
: reliable_stream_reset

Status:
: Permanent

Specification:
: This document

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: QUIC Working Group (quic@ietf.org)

## QUIC Frame Types

This document registers one new value in the "QUIC Frame Types" registry
established in {{Section 22.4 of RFC9000}}. The following fields are registered:

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
