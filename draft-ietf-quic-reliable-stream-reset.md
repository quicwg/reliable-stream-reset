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
This document defines a new QUIC frame, the CLOSE_STREAM frame, that allows
closing and resetting of a stream, while guaranteeing reliable delivery of
stream data up to a certain byte offset.

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
CLOSE_STREAM frame. This frame allows an endpoint to mark a portion at
the beginning of the stream which will then be guaranteed to be delivered to
receiver's application, even if the stream was reset.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the close_stream (0x17f7586d2cb570) transport parameter
(Section 7.4 of {{!RFC9000}}) with an empty value. An implementation that
understands this transport parameter MUST treat the receipt of a non-empty
value as a connection error of type TRANSPORT_PARAMETER_ERROR.

In order to allow reliable stream resets in 0-RTT packets, both endpoints MUST
remember the value of this transport parameter.  If 0-RTT data is
accepted by the server, the server MUST NOT disable this extension on the
resumed connection.

# CLOSE_STREAM Frame

Conceptually, the CLOSE_STREAM frame is a RESET_STREAM frame with an
added Reliable Size field.

~~~
CLOSE_STREAM Frame {
  Type (i) = 0x20..0x21,
  Stream ID (i),
  [Application Protocol Error Code (i)],
  Final Size (i),
  Reliable Size (i),
}
~~~

The CLOSE_STREAM frames contain the following fields:

Stream ID:  A variable-length integer encoding of the stream ID of
      the stream being terminated.

Application Protocol Error Code:  A variable-length integer
    containing the application protocol error code (see Section 20.2)
    that indicates why the stream is being closed. If the Type is 0x20,
    this field is not included.

Final Size:  A variable-length integer indicating the final size of
    the stream by the RESET_STREAM sender, in units of bytes; see
    (Section 4.5 of {{!RFC9000}}).

Reliable Size:  A variable-length integer indicating the amount of
    data that needs to be delivered to the application even though
    the stream is reset.

If the Reliable Size is larger than the Final Size, the receiver MUST close the
connection with a connection error of type FRAME_ENCODING_ERROR.

CLOSE_STREAM frames are ack-eliciting. When lost, they MUST be retransmitted,
unless the stream state has transitioned to "Data Recvd" or "Reset Recvd" due
to transmission and acknowledgement of other frames (see {{multiple-frames}}).

# Closing Streams

## Without an Error Code

When closing a stream without an error code, the node has the choice between a
STREAM frame that carries the FIN bit and a CLOSE_STREAM frame of type 0x20.

The CLOSE_STREAM frame can be used to reduce the reliable offset after a
STREAM frame with a FIN bit has been sent. If STREAM frames containing data
up to that byte offset are lost, the initiator MUST retransmit this data, as
described in (Section 13.3 of {{!RFC9000}}). Data sent beyond that byte offset
SHOULD NOT be retransmitted.

## Using an Error Code

When resetting a stream, the node has the choice between using a RESET_STREAM
frame and a CLOSE_STREAM frame of type 0x21. When using a RESET_STREAM frame,
the behavior is unchanged from the behavior described in ({{!RFC9000}}).

When using a CLOSE_STREAM frame, the initiator MUST guarantee reliable delivery
of stream data of at least Reliable Size bytes. If STREAM frames containing data
up to that byte offset are lost, the initiator MUST retransmit this data, as
described in (Section 13.3 of {{!RFC9000}}). Data sent beyond that byte offset
SHOULD NOT be retransmitted.

As described in (Section 3.2 of {{RFC9000}}), it MAY deliver data beyond that
offset to the application.

## Multiple CLOSE_STREAM / RESET_STREAM frames {#multiple-frames}

The initiator MAY send multiple CLOSE_STREAM frames for the same
stream in order to reduce the Reliable Size.  It MAY also send a RESET_STREAM
frame, which is equivalent to sending a CLOSE_STREAM frame with a
Reliable Size of 0.

When sending multiple frames for the same stream, the initiator MUST NOT increase
the Reliable Size.  When receiving a CLOSE_STREAM frame with a lower
Reliable Size, the receiver only needs to deliver data up the lower Reliable
Size to the application. It MUST NOT expect the delivery of any data beyond that
byte offset.

Reordering of packets might lead to a CLOSE_STREAM frame with a higher
Reliable Size being received after a CLOSE_STREAM frame with a lower
Reliable Size.  The receiver MUST ignore any CLOSE_STREAM frame that
increases the Reliable Size.

When sending another CLOSE_STREAM, RESET_STREAM or STREAM frame carrying a FIN
bit for the same stream, the initiator MUST NOT change the Application Error
Code or the Final Size.  This also means that sending CLOSE_STREAM frames of
different types is not permitted. If the receiver detects a change in those
fields, it MUST close the connection with a connection error of type
STREAM_STATE_ERROR.

# Security Considerations

TODO Security


# IANA Considerations

## QUIC Transport Parameter

This document registers the close_stream transport parameter in the "QUIC
Transport Parameters" registry established in {{Section 22.3 of RFC9000}}. The
following fields are registered:

Value:
: 0x17f7586d2cb570

Parameter Name:
: close_stream

Status:
: Permanent

Specification:
: This document

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: QUIC Working Group (quic@ietf.org)

## QUIC Frame Types

This document registers two new values in the "QUIC Frame Types" registry
established in {{Section 22.4 of RFC9000}}. The following fields are
registered:

Value:
: 0x20-0x21

Frame Type Name:
: CLOSE_STREAM

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
