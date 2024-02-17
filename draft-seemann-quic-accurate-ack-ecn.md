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
    email: "martenseemann@gmail.com"
 -
    fullname: "Vidhi Goel"
    organization: Apple Inc.
    email: "vidhi_goel@apple.com"

normative:

informative:


--- abstract

QUIC defines a variant of the ACK frame that carries cumulative count for
each of the three ECN codepoints (ECT(1), ECT(0) and CE). From this information,
the recipient of the ACK frame cannot deduce which ECN marking the individual
packets were received with.

This document defines an alternative ACK frame that encodes enough information
to indicate which ECN mark each individual packet was received with.

--- middle

# Introduction

Some congestion control algorithms would benefit from not only knowing
that some packets were marked with Congestion Experienced (CE) bit,
but exactly which ones. In the general case,
this is not possible with the standard {{!RFC9000}} ACK frame, since it only
contains cumulative ECN counts.

This document defines an alternative ACK frame, the ACCURATE_ACK_ECN frame,
which encodes the corresponding ECN codepoint alongside the ACK range.
This encoding comes at a cost: In the presence of ECN markings, this will lead
to ACCURATE_ACK_ECN frames containing more ACK ranges compared to a regular
ACK frame. However, this is not expected to significantly inflate the size of
ACCURATE_ACK_ECN frames.
For example, in the steady state, L4S {{!RFC9331}} applies the CE marking to two
packets per roundtrip. In the absence of packet loss, two of the
ACCURATE_ACK_ECN frames sent during that RTT would contain two ACK ranges
instead of one.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# ACCURATE_ACK_ECN Frame

The ACCURATE_ACK_ECN frame looks similar to an {{!RFC9000}} ACK frame. It uses a
different encoding for ACK ranges (see {{first-ack-range}} and {{ack-ranges}}).

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

All packets within an ACK range MUST have been received with the same ECN code point.
If a range of packets with contiguous packet numbers, but different ECN markings
is received, this MUST be reported using multiple ACK ranges.

Similar to regular ACK frames, ACCURATE_ACK_ECN frames are not ack-eliciting
(see {{Section 13.2 of RFC9000}}), nor are they congestion-controlled.

## First ACK Range {#first-ack-range}

~~~
First ACK Range {
  ACK Range Length (i),
  ECN Marking (8),
}
~~~

ACK Range Length:

: A variable-length integer indicating the number of contiguous packets
preceding the Largest Acknowledged that are being acknowledged with the same
ECN code point. That is, the smallest packet acknowledged in the range
with the same ECN code point is determined by subtracting the First ACK Range
Length value from the Largest Acknowledged field.

ECN Marking:

: The ECN code point all packets in this range were received with: Non-ECT is
encoded as 0, ECT(1) as 1, ECT(0) as 2 and CE as 3. Values larger than or equal
to 4 are invalid, and the receiver MUST close the connection with a
FRAME_ENCODING_ERROR if it receives an ACK range with an invalid ECN marking value.

## ACK Ranges {#ack-ranges}

Each ACK Range consists of alternating Gap, ACK Range Length and ECN Marking
values in descending packet number order. ACK Ranges can be repeated. The number
of ranges is determined by the ACK Range Count field; one of each value is
present for each value in the ACK Range Count field.

ACK Ranges are structured as shown in {{ack-range-format}}.

~~~
ACK Range {
  Gap (i),
  ACK Range Length (i),
  ECN Marking (8),
}
~~~
{: #ack-range-format title="ACK Ranges"}

The fields that form each ACK Range are:

Gap:

: A variable-length integer indicating the number of contiguous unacknowledged
  packets preceding the packet number given by the smallest in the preceding ACK
  Range. Note that this definition differs by one from the Gap definition of
  the standard QUIC ACK frame in {{Section 19.3.1 of RFC9000}}. This is
  necessary to allow encoding of contiguous ranges of packet numbers that were
  received with different ECN markings.

ACK Range Length:

: A variable-length integer indicating the number of contiguous acknowledged
  packets preceding the largest packet number, as determined by the
  preceding Gap.

ECN Marking:

: The ECN code point all packets in this range were received with, as defined in
  {{first-ack-range}}.

As described in {{Section 19.3.1 of RFC9000}}, given a largest packet number for
an ACK range, the smallest value is determined by:

~~~
smallest = largest - ack_range
~~~

To calculate the largest value for a subsequent ACK Range, the formula differs
from the standard QUIC ACK frame which can be calculated using:

~~~
largest = previous_smallest - gap - 1
~~~

If any computed packet number is negative, an endpoint MUST generate a connection
error of type FRAME_ENCODING_ERROR.

## Example

Consider a scenario where 10 packets (from packet number 1 to 10) were sent with
ECT(1) but receiver received a total of 9 packets where packet number 8 was lost
and packet number 6 and 9 were CE marked. The ACCURATE_ACK_ECN frame would look
like below.

~~~
ACCURATE_ACK_ECN Frame {
  Type (i) = 0x2051a5fa,
  Largest Acknowledged (10),
  ACK Delay (i),
  ACK Range Count (4),
  First ACK Range {
    ACK Range Length (0),
    ECN Marking (1),
  },
  ACK Range {
    Gap (0),
    ACK Range Length (0),
    ECN Marking (3),
  }
  ACK Range {
    Gap (1),
    ACK Range Length (0),
    ECN Marking (1),
  }
  ACK Range {
    Gap (0),
    ACK Range Length (0),
    ECN Marking (3),
  }
  ACK Range {
    Gap (0),
    ACK Range Length (4),
    ECN Marking (1),
  }
}
~~~

# Negotiating Extension Use {#negotiate-extension}

Endpoints advertise their support of the extension by sending the
accurate_ack_ecn (0x2051a5fa8648af) transport parameter ({{Section 7.4 of
RFC9000}}) with an empty value. Implementations that understand this transport
parameter MUST treat the receipt of a non-empty value as a connection error of
type TRANSPORT_PARAMETER_ERROR.

After negotiating this extension, endpoints MUST report received packets using
the ACCURATE_ACK_ECN frame. This only applies to the application data packet
number space. Initial and Handshake packets are acknowledged using the regular
ACK frame.

It is invalid to send regular ACK frames in the application data packet number
space after negotiating this extension. Endpoints MUST close the connection
using a PROTOCOL_VIOLATION error when they receive an ACK frame in the
application data packet number space after this extension was negotiated.

When using 0-RTT, both endpoints MUST remember the value of this transport
parameter. This allows acknowledging 0-RTT packets using ACCURATE_ACK_ECN
frames. If 0-RTT data is accepted by the server, the server MUST NOT disable
this extension on the resumed connection.

# Security Considerations

The sender of an ACCURATE_ACK_ECN frame might be able to burden its peer by
encoding a large number of ACK ranges. With the ACK frame defined in {{RFC9000}}
it is not possible to split a contiguous sequence of packet numbers into
multiple ranges, which becomes possible when using the ACCURATE_ACK_ECN frame.
The number of ACK ranges is implicitely by the requirement that each frame fits
into a QUIC packet. Receivers SHOULD make sure that they can process an
ACCURATE_ACK_ECN frame containing a few hundred ACK ranges efficiently.

# IANA Considerations

TODO consider IANA

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
