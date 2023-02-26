---
title: "QUIC Stream Reset Payload"
abbrev: "QUIC Stream Reset Payload"
docname: draft-seemann-quic-reliable-stream-reset-latest
category: info

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
stream is delivered to the application. The only application protocol data
contained in RESET_STREAM is a 62-bit error code.
This document defines a new QUIC frame, RESET_STREAM_WITH_PAYLOAD, that
acts in the same way as RESET_STREAM, but contains additional application
protocol payload.

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
RESET_STREAM_WITH_PAYLOAD frame. This frame allows an endpoint to communicate
additional application protocol data while resetting a stream; such data can be
used to maintain stream associations, or to communicate additional error
information on top of the 62-bit error code.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the reliable_reset_stream (0x727274) transport parameter
(Section 7.4 of {{!RFC9000}}) with an empty value. An implementation that
understands this transport parameter MUST treat the receipt of a non-empty
value as a connection error of type TRANSPORT_PARAMETER_ERROR.

In order to allow reliable stream resets in 0-RTT packets, the client MUST
remember the value of this transport parameter.  If 0-RTT data is accepted by
the server, the server MUST not disable this extension on the resumed
connection.

# RESET_STREAM_WITH_PAYLOAD Frame

Conceptually, the RESET_STREAM_WITH_PAYLOAD frame is a RESET_STREAM frame with
an added Application Protocol Payload field.

~~~
RESET_STREAM_WITH_PAYLOAD Frame {
  Type (i) = 0x73,
  Stream ID (i),
  Application Protocol Error Code (i),
  Final Size (i),
  Application Protocol Payload Size (i),
  Application Protocol Payload (..),
}
~~~

RESET_STREAM_WITH_PAYLOAD frames contain the following fields:

Stream ID:  A variable-length integer encoding of the stream ID of
      the stream being terminated.

Application Protocol Error Code:  A variable-length integer
    containing the application protocol error code (see Section 20.2)
    that indicates why the stream is being closed.

Final Size:  A variable-length integer indicating the final size of
    the stream by the RESET_STREAM sender, in units of bytes; see
    (Section 4.5 of {{!RFC9000}}).

Application Protocol Payload Size:  A variable-length integer specifying
    the length of the application payload in bytes. Because
    a RESET_STREAM_WITH_PAYLOAD frame cannot be split between packets,
    any limits on packet size will also limit the space available for
    the application payload.

Application Protocol Payload:  An application-specific payload accompanying
    the reset.  An application protocol MUST define the semantics of this field
    in order for the RESET_STREAM_WITH_PAYLOAD frame to be used.

Semantically, a RESET_STREAM frame is equivalent to a RESET_STREAM_WITH_PAYLOAD
frame with the Application Protocol Payload being empty.

RESET_STREAM_WITH_PAYLOAD frames are ack-eliciting. When lost, they MUST be
retransmitted.

TODO If we are using RESET_STREAM_WITH_PAYLOAD to communicate error codes, do
we need to define a similar frame for STOP_SENDING?

# Security Considerations

TODO Security


# IANA Considerations

TODO Register the new transport parameter and frame type.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
