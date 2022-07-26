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

This document describes a QUIC extension to allow the reset of streams which
guarantees the reliable delivery of parts of the data sent on the stream.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Motivation

QUIC allows resetting of streams. When a stream is reset, the sender doesn't retransmit stream data for the respective stream. On the receiver side, the QUIC stack is free to surface the stream reset to the application, even if stream data is received.

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the following transport parameter ({{Section 7.2 of QUIC-TRANSPORT}}):

reliable_reset_stream (TBD):

: A variable-length integer encoding a 1.

TODO: 0-RTT interaction.

# RELIABLE_RESET_STREAM Frame

The RELIABLE_RESET_STREAM frame adds an additional Reliable Size field to the RESET_STREAM frame.

```
~~~
RELIABLE_RESET_STREAM Frame {
  Type (i) = TBD,
	Stream ID (i),
	Application Protocol Error Code (i),
	Final Size (i),
	Reliable Size (i)
}
~~~
```

Semantically, a RESET_STREAM frame is equivalent to a RELIABLE_RESET_STREAM frame with the Reliable Size set to 0.

# Resetting Streams

When resetting a stream, the node has the choice between using a RESET_STREAM frame and a RELIABLE_RESET_STREAM frame. When using a RESET_STREAM frame, the behavior is unchanged from RFC 9000.

When using the RELIALBE_RESET_STREAM frame:

* The initiator guarantees reliable delivery of stream data of at least Reliable Size bytes. If STREAM frames containing data up to that byte offset are lost, the initiator MUST retransmit this data,  as described in ({{Section 13.3 of QUIC-TRANSPORT}}). Data sent beyond that byte offset SHOULD NOT be retransmitted.
* The receiver MUST deliver at least Reliable Size bytes to the application before surfacing the stream reset error. As described in TODO, it MAY deliver data beyond that offset to the application.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
