---
title: "QUIC Extension for Reporting Packet Receive Timestamps"
abbrev: "QUIC Receive Timestamps"
docname: draft-smith-quic-receive-ts-latest
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
    ins: C. Smith
    name: Connor Smith
    org: Magic Leap, Inc.
    email: connorsmith.ietf@gmail.com
 -
    ins: I. Swett
    name: Ian Swett
    org: Google LLC
    email: ianswett@google.com

normative:

informative:


--- abstract

This document defines an extension to the QUIC transport protocol which supports
reporting multiple packet receive timestamps using a new ACK_RECEIVE_TIMESTAMPS
frame type.


--- middle

# Introduction

The QUIC Transport Protocol {{!RFC9000}} provides a secure, multiplexed
connection for transmitting reliable streams of application data.

This document defines an extension to the QUIC transport protocol which supports
reporting multiple packet receive timestamps using a new ACK_RECEIVE_TIMESTAMPS
frame type.


# Motivation

TODO Describe motivation for receive timestamps


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Extension Negotiation {#negotiation}

The use of the ACK_RECEIVE_TIMESTAMPS frame is negotiated using the following
two transport parameters ({{Section 7.2 of !RFC9000}}):

max_receive_timestamps_per_ack (TBD):

: A variable-length integer indicating that the sending endpoint would like to
  receive ACK_RECEIVE_TIMESTAMPS frames from the peer containing no more than
  the given maximum number of receive timestamps.

  Upon receiving this parameter with a non-zero value, the peer SHOULD send
  ACK_RECEIVE_TIMESTAMPS frames instead of ACK frames if new receive timestamp
  information is available. The peer MAY still send regular ACK frames (e.g. if
  no timing information is available), in which case the endpoint MUST still
  support processing regular ACK frames as defined by {{Section 19.3 of
  !RFC9000}}.

  Each ACK_RECEIVE_TIMESTAMPS frame sent MUST NOT contain more than the
  specified maximum number of receive timestamps, but MAY contain fewer or none.

receive_timestamps_exponent (TBD):

: A variable-length integer indicating the exponent to be used when encoding and
  decoding timestamp delta fields in ACK_RECEIVE_TIMESTAMPS frames sent by the
  peer (see {{ts-ranges}}). If this value is absent, a default value of 0 is
  assumed (indicating microsecond precision). Values above 20 are invalid.


## Receive Timestamp Basis {#ts-basis}

Endpoints which send ACK_RECEIVE_TIMESTAMPS frames must determine a value,
receive_timestamp_basis, relative to which all receive timestamps for the
session will be reported (see {{ts-ranges}}).

The value of receive_timestamp_basis MUST be less than the smallest receive
timestamp reported, and MUST remain constant for the entire duration of the
session.

TODO: Discuss (here or below) why receive timestamps are reported relative to
the basis, rather than in absolute time to avoid clock synchronization between
endpoints.


# ACK_RECEIVE_TIMESTAMPS Frame {#frame}

Receivers send ACK_RECEIVE_TIMESTAMPS (type=TBD) frames in place of--and in the
same manner as--regular ACK frames as described in {{Section 13.2 of !RFC9000}}.
However, ACK_RECEIVE_TIMESTAMPS frames contain additional fields to report
packet receive timestamps.

ACK_RECEIVE_TIMESTAMPS frames are formatted as shown in {{fig-frame}}.

~~~
ACK_RECEIVE_TIMESTAMPS Frame {
  Type (i) = TBD
  // Fields of the existing ACK (type=0x02) frame:
  Largest Acknowledged (i),
  ACK Delay (i),
  ACK Range Count (i),
  First ACK Range (i),
  ACK Range (..) ...,
  // Additional fields for ACK_RECEIVE_TIMESTAMPS:
  Timestamp Range Count (i),
  Timestamp Ranges (..) ...,
}
~~~
{: #fig-frame title="ACK_RECEIVE_TIMESTAMPS Frame Format"}

The fields Largest Acknowledged, ACK Delay, ACK Range Count, First ACK Range,
and ACK Range are the same as for ACK (type=0x02) frames specified in {{Section
19.3 of !RFC9000}}.

ACK_RECEIVE_TIMESTAMPS frames contain the following additional fields:

Timestamp Range Count:

: A variable-length integer specifying the number of Timestamp Range fields in
  the frame.

Timestamp Ranges:

: Ranges of receive timestamps for contiguous packets in descending packet
  number order; see {{ts-ranges}}.


## Timestamp Ranges {#ts-ranges}

Each Timestamp Range describes a series of contiguous packet receive timestamps
in descending sequential packet number (and descending timestamp) order. Timestamp Ranges
consist of a Gap indicating the largest packet number in the range, followed by
a list of Timestamp Deltas describing the relative receive timestamps for each
contiguous packet in the Timestamp Range (descending).

Note that reporting receive timestamps for packets received out of order is not
supported. Specifically: for any packet number P for which a receive timestamp T
is reported, all reports for packet numbers less than P must have timestamps
less than or equal to T; and all reports for packet numbers greater than P must
have timestamps greater than or equal to T.

Timestamp Ranges are structured as shown in {{fig-ts-range}}.

~~~
Timestamp Range {
  Gap (i),
  Timestamp Delta Count (i),
  Timestamp Delta (i) ...,
}
~~~
{: #fig-ts-range title="Timestamp Range Format"}

The fields that form each Timestamp Range are:

Gap:

: A variable-length integer indicating the largest packet number in the
  Timestamp Range as follows:

  For the first Timestamp Range: Gap is the difference between (a) the Largest
  Acknowledged packet number in the frame and (b) the largest packet in the
  current (first) Timestamp Range.

  For subsequent Timestamp Ranges: Gap is the difference between (a) the packet
  number two lower than the smallest packet number in the previous Timestamp
  Range and (b) the largest packet in the current Timestamp Range.

Timestamp Delta Count:

: A variable-length integer indicating the number of Timestamp Deltas in the
  current Timestamp Range.

Timestamp Deltas:

: Variable-length integers encoding the receive timestamp for contiguous packets
  in the Timestamp Range in descending packet number order as follows:

  For the first Timestamp Delta of the first Timestamp Range in the frame: the
  value is the difference between (a) the receive timestamp of the largest
  packet in the Timestamp Range (indicated by Gap) and (b) the session
  receive_timestamp_basis (see {{ts-basis}}), decoded as described below.

  For all other Timestamp Deltas: the value is the difference between (a) the
  receive timestamp specified by the previous Timestamp Delta and (b) the
  receive timestamp of the current packet in the Timestamp Range, decoded as
  described below.

  All Timestamp Delta values are decoded by mulitplying the value in the field
  by 2 to the power of the receive_timestamps_exponent transport parameter
  received by the sender of the ACK_RECEIVE_TIMESTAMPS frame (see
  {{negotiation}}):

# Discussion

TODO Discussion


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
