---
title: "Reliable QUIC Stream Resets"
abbrev: "QUIC Reliable Stream Reset"
docname: draft-seemann-quic-reliable-stream-reset-latest
category: std

ipr: trust200902
area: Transport
workgroup: QUIC
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Seemann
    name: Marten Seemann
    organization: Protocol Labs
    email: martenseemann@gmail.com

normative:

informative:


--- abstract

QUIC ({{!RFC9000}}) defines a RESET_STREAM frame to reset a stream. When a
sender resets a stream, it stops retransmitting STREAM frames for this stream.
On the receiver side, there is no guarantee that any of the data sent on that
stream is delivered to the application.
This document defines a new QUIC frame, the RELIABLE_RESET_STREAM frame, that
resets a stream, while guaranteeing reliable delivery of stream data up to a
certain byte offset.

--- middle

# Introduction

QUIC v1 ({{!RFC9000}}) allows streams to be reset.  When a stream is
reset, the sender doesn't retransmit stream data for the respective stream.
On the receiver side, the QUIC stack is free to surface the stream reset to the
application immediately, even if it has already received stream data for that
stream.

Applications running on top of QUIC might need to send an identifier at the
beginning of the stream in order to associate that stream with a specific
subpart of the application. For example, WebTransport
({{!WEBTRANSPORT=I-D.ietf-webtrans-http3}}) uses a variable-length-encoded
integer (as defined in QUIC v1) to transmit the ID of the WebTransport session to
the receiver.

It is desirable that the receiver is able to associate incoming streams with
their respective subpart of the application, even if the QUIC stream is reset
before the identifier at the beginning of the stream was read.

This document describes a QUIC extension defining a new frame type, the
RELIABLE_RESET_STREAM frame. This frame allows an endpoint to mark a portion at
the beginning of the stream which will then be guaranteed to be delivered to
receiver's application, even if the stream was reset.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the reliable_reset_stream (0x727273) transport parameter
(Section 7.4 of {{!RFC9000}}) with an empty value. An implementation that
understands this transport parameter MUST treat the receipt of a non-empty
value as a connection error of type TRANSPORT_PARAMETER_ERROR.

In order to allow reliable stream resets in 0-RTT packets, the client MUST
remember the value of this transport parameter.  If 0-RTT data is accepted by
the server, the server MUST not disable this extension on the resumed
connection.

# RELIABLE_RESET_STREAM Frame

Conceptually, the RELIABLE_RESET_STREAM frame is a RESET_STREAM frame with an
added Reliable Size field.

~~~
RELIABLE_RESET_STREAM Frame {
  Type (i) = 0x72,
  Stream ID (i),
  Application Protocol Error Code (i),
  Final Size (i),
  Reliable Size (i),
}
~~~

RELIABLE_RESET_STREAM frames contain the following fields:

Stream ID:  A variable-length integer encoding of the stream ID of
      the stream being terminated.

Application Protocol Error Code:  A variable-length integer
    containing the application protocol error code (see Section 20.2)
    that indicates why the stream is being closed.

Final Size:  A variable-length integer indicating the final size of
    the stream by the RESET_STREAM sender, in units of bytes; see
    (Section 4.5 of {{!RFC9000}}).

Reliable Size:  A variable-length integer indicating the amount of
    data that needs to be delivered to the application even though
    the stream is reset.

If the Reliable Size is larger than the Final Size, the receiver MUST close the
connection with a connection error of type FRAME_ENCODING_ERROR.

Semantically, a RESET_STREAM frame is equivalent to a RELIABLE_RESET_STREAM
frame with the Reliable Size set to 0.

RELIABLE_RESET_STREAM frames are ack-eliciting. When lost, they MUST be
retransmitted, unless a RESET_STREAM frame or another RELIABLE_RESET_STREAM
frame was sent for the same stream (see {{multiple-frames}}).

When retransmitting RELIABLE_RESET_STREAM frames for the same stream, the
seder MUST NOT change the Application Error Code, the Final Size, or the
Reliable Size.

If the receiver detects a change in those fields, it MUST close the
connection with a connection error of type STREAM_STATE_ERROR.

# Resetting Streams

When resetting a stream, the node has the choice between using a RESET_STREAM
frame and a RELIABLE_RESET_STREAM frame. When using a RESET_STREAM frame, the
behavior is unchanged from the behavior described in ({{!RFC9000}}).

The initiator MUST guarantee reliable delivery of stream data of at least
Reliable Size bytes.  If STREAM frames containing data up to that byte offset
are lost, the initiator MUST retransmit this data,  as described in
(Section 13.3 of {{!RFC9000}}). Data sent beyond that byte offset SHOULD NOT be
retransmitted.

As described in (Section 3.2 of {{RFC9000}}), it MAY deliver data beyond that
offset to the application.

## Multiple RELIABLE_RESET_STREAM / RESET_STREAM frames {#multiple-frames}

The initiator MAY send RESET_STREAM frames on a stream that it has already sent
RELIABLE_RESET_STREAM frames.

When a receiver observes a RELIABLE_RESET_STREAM frame on a stream then
receives a RESET_STREAM frame on the same stream, the receiver cannot expect
delivery of any additional data on that stream.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
