---
title: "Reliable QUIC Stream Resets"
abbrev: "QUIC Reliable Stream Reset"
docname: draft-seemann-quic-reliable-stream-reset-latest
category: info

ipr: trust200902
area: General
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
    email: marten@protocol.ai

normative:

  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: 2021-05
    seriesinfo:
      RFC: 9000
      DOI: 10.17487/RFC9000
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

informative:


--- abstract

TODO Abstract


--- middle

# Introduction

QUIC v1 ({{QUIC-TRANSPORT}}) allows streams to be reset.  When a stream is
reset, the sender doesn't retransmit stream data for the respective stream.
On the receiver side, the QUIC stack is free to surface the stream reset to the
application immediately, even if it has already received stream data for that
stream.

Application running on top of QUIC might need to send an identifier at the
beginning of the stream in order to associate that stream with a specific
subpart of the application.  For example, WebTransport (TODO: cite) uses a
QUIC varint to encode the ID of the WebTransport session.

It is desirable that the receiver is able to associate incoming streams with
their respective subpart of the application, even if the QUIC stream is reset
before the identifier at the beginning of the stream was read.

This document describes a QUIC extension that allows an endpoint to mark a
portion at the beginning of the stream, which will then be guaranteed to be
delivered to receiver's application, even if the stream was reset.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the reliable_reset_stream (TBD) transport parameter
({{Section 7.2 of QUIC-TRANSPORT}}) with an empty value. An implementation that
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
  Type (i) = TBD,
  Stream ID (i),
  Application Protocol Error Code (i),
  Final Size (i),
  Reliable Size (i)
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
    ({{Section 4.5 of QUIC-TRANSPORT}}).

Reliable Size:  A variable-length integer indicating the amount of
    data that needs to be delivered to the application before the
    error code can be surfaced, in units of bytes.

If the Reliable Size is larger than the Final Size, the receiver MUST close the
connection with a connection error of type FRAME_ENCODING_ERROR.

Semantically, a RESET_STREAM frame is equivalent to a RELIABLE_RESET_STREAM
frame with the Reliable Size set to 0.

RELIABLE_RESET_STREAM frames are ack-eliciting. When lost,
RELIABLE_RESET_STREAM frames MUST be retransmitted, unless another
RELIABLE_RESET_STREAM was sent for the same stream (see {{multiple-frames}}).

# Resetting Streams

When resetting a stream, the node has the choice between using a RESET_STREAM
frame and a RELIABLE_RESET_STREAM frame. When using a RESET_STREAM frame, the
behavior is unchanged the behavior described in ({{QUIC-TRANSPORT}}).

When using the RELIABLE_RESET_STREAM frame:

- The initiator guarantees reliable delivery of stream data of at least
  Reliable Size bytes. If STREAM frames containing data up to that byte offset
  are lost, the initiator MUST retransmit this data,  as described in
  ({{Section 13.3 of QUIC-TRANSPORT}}). Data sent beyond that byte offset
  SHOULD NOT be retransmitted.
- The receiver MUST deliver at least Reliable Size bytes to the application
  before surfacing the stream reset error. As described in TODO, it MAY deliver
  data beyond that offset to the application.

## Multiple RELIABLE_RESET_STREAM frames {#multiple-frames}

The initiator MAY send multiple RELIABLE_RESET_STREAM frames for the same
stream in order to reduce the Reliable Size.  It MUST NOT increase the Reliable
Size.  When receiving a RELIABLE_RESET_STREAM frame with a lower Reliable Size,
the receiver only needs to deliver data up the lower Reliable Size to the
application before surfacing the stream reset error.  It MUST NOT expect the
delivery of any data beyond that byte offset.

When sending another RELIABLE_RESET_STREAM frame for the same stream, the
initiator MUST NOT change the Application Error Code and the Final Size. The
receiver MUST close the connection with a connection error of type
STREAM_STATE_ERROR.

Reordering of packets might lead to a RELIABLE_RESET_STREAM frame with a higher
Reliable Size to be received after a RELIABLE_RESET_STREAM frame with a lower
Reliable Size.  The receiver MUST ignore any RELIABLE_RESET_STREAM frame that
increases the Reliable Size.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
