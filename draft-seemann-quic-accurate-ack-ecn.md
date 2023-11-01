---
title: "QUIC Accurate ECN Acknowledgements"
abbrev: "QUIC Accurate ECN Acknowledgements"
category: std

docname: draft-seemann-quic-accurate-ack-ecn-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - QUIC
 - ECN
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "marten-seemann/draft-seemann-quic-accurate-ack-ecn"
  latest: "https://marten-seemann.github.io/draft-seemann-quic-accurate-ack-ecn/draft-seemann-quic-accurate-ack-ecn.html"

author:
 -
    fullname: "Marten Seemann"
    organization: Protocol Labs
    email: "martenseemann@gmail.com"

normative:

informative:


--- abstract

QUIC defines a variant of the ACK frame that carries three counters, one for the
ECT(0), one for the ECT(1) and one for the CE codepoint. From this information,
the recipient of the ACK frame cannot deduce which packet was ECN-marked, if the
ACK frame acknowledges multiple packets at once.

This document defines an ACK frame that encodes enough information to encode
what ECN marks each individual packet carried.

--- middle

# Introduction

Some advanced congestion control algorithms would benefit from not only knowing
that some packets were ECN-marked, but exactly which ones. This is in general
not possible with the standard {{!RFC9000}} ACK frame, since it only contains
cumulative ECN counts.

This document defines an ACK frame that encodes the ECN codepoint alongside the
ACK range. This encoding doesn't come for free: In the presence of ECN markings,
this will lead to ACK frames containing more ACK ranges. However, it is expected
that this doesn't inflate the size of ACK frames too much. For example, in the
steady state, L4S {{!RFC9331}} leads to two packets being CE-marked per
roundtrip. In that case, two of the ACK frames sent during that RTT would
contain two instead of one ACK range.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# ACCURATATE_ACK_ECN Frame

The ACCURATE_ACK_ECN frame looks similar to an {{!RFC9000}} ACK frame. It uses a
different encoding for ACK ranges (see below).

~~~
ACCURATE_ACK_ECN Frame {
  Type (i) = 0x2051a5fa,
  Largest Acknowledged (i),
  ACK Delay (i),
  ACK Range Count (i),
  First ACK Range (..),
  ACK Range (..) ...,
}
~~~

Except for the two ACK Range fields, all the fields are the same as defined in
{{Section 19.3 of RFC9000}}.

Similar to regular ACK frames, ACCURATE_ACK_ECN frames are not ack-eliciting
(see {{Section 13.2 of RFC9000}}), nor are they congestion-controlled.

## First ACK Range {#first-ack-range}

~~~
First ACK Range {
  ACK Range Length (i),
  ECN marking (8),
}
~~~

ACK Range Length:

: A variable-length integer indicating the number of contiguous packets
preceding the Largest Acknowledged that are being acknowledged. That is, the
smallest packet acknowledged in the range is determined by subtracting the First
ACK Range value from the Largest Acknowledged field.

ECN marking

: The ECN code point all packets in this range were received with: Non-ECT is
encoded as 0, ECT(1) as 1, ECT(0) as 2 and CE as 3. Values larger or equal than
4 are invalid, and the receiver MUST close the connection with a
FRAME_ENCODING_ERROR if it receives an ACK range with an invalid value.

## ACK Ranges {#ack-ranges}

Each ACK Range consists of alternating Gap, ACK Range Length and ECN marking
values in descending packet number order. ACK Ranges can be repeated. The number
of Gap and ACK Range Length values is determined by the ACK Range Count field;
one of each value is present for each value in the ACK Range Count field.

ACK Ranges are structured as shown in {{ack-range-format}}.

~~~
ACK Range {
  Gap (i),
  ACK Range Length (i),
  ECN marking (8),
}
~~~
{: #ack-range-format title="ACK Ranges"}

The fields that form each ACK Range are:

Gap:

: A variable-length integer indicating the number of contiguous unacknowledged
  packets preceding the packet number than the smallest in the preceding ACK
  Range. Note that this definition differes by one from the Gap definition of
  the standard QUIC ACK frame in {{Section 19.3.1 of RFC9000}}. This is
  necessary to allow encoding of contiguous ranges of packet numbers that were
  received with different ECN markings.

ACK Range Length:

: A variable-length integer indicating the number of contiguous acknowledged
  packets preceding the largest packet number, as determined by the
  preceding Gap.

ECN marking:

: The ECN code point all packets in this range were received with, as defined in
  {{first-ack-range}}.

# Negotiating Extension Use {#negotiate-extension}

Endpoints advertise their support of the extension by sending the
address_discovery (0x2051a5fa8648af) transport parameter ({{Section 7.4 of
RFC9000}}) with an empty value. Implementations that understand this transport
parameter MUST treat the receipt of a non-empty value as a connection error of
type TRANSPORT_PARAMETER_ERROR.

When using 0-RTT, both endpoints MUST remember the value of this transport
parameter. This allows sending the frames defined by this extension in 0-RTT
packets. If 0-RTT data is accepted by the server, the server MUST NOT disable
this extension on the resumed connection.

# Security Considerations

The sender of an ACK frame might be able to make its peer do by encoding a large
number of ACK ranges. With the ACK frame defined in {{RFC9000}} it is not
possible to split a contiguous sequence of packet numbers into multiple ranges,
which is possible when using the ACCURATE_ACK_ECN frame. The number of ACK
ranges is implicitely by the requirement that each frame fits into a QUIC
packet. Receivers SHOULD make sure that they can process an ACCURATE_ACK_ECN
frame containing a few hundred ACK ranges efficiently.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
