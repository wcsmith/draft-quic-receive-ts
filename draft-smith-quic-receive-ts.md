---
title: "QUIC Extended Acknowledgement for Reporting Packet Receive Timestamps"
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
    org: NVIDIA
    email: connorsmith.ietf@gmail.com
 -
    ins: I. Swett
    name: Ian Swett
    org: Google LLC
    email: ianswett@google.com
 -
    ins: J. Beshay
    name: Joseph Beshay
    org: Meta Platforms, Inc.
    email: jbeshay@meta.com
 -
    ins: S. Jaiswal
    name: Sharad Jaiswal
    org: Meta Platforms, Inc.
    email: sj77@meta.com
 -
    ins: I. Purushothaman
    name: Ilango Purushothaman
    org: Meta Platforms, Inc.
    email: ipurush@meta.com
 -
    ins: B. Schlinker
    name: Brandon Schlinker
    org: Meta Platforms, Inc.
    email: bschlinker@meta.com


normative:

informative:

  RRBNC:
    title: "pathChirp: Efficient Available Bandwidth Estimation for Network
Paths"
    date: 2003
    author:
      name: "Ribeiro, V., Riedi, R., Baraniuk, R., Navratil, J., and L. Cottrel"
    publication: "Passive and Active Monitoring Workshop"


--- abstract

This document defines an extension to the QUIC transport protocol which supports
reporting multiple packet receive timestamps using a new ACK_EXTENDED
frame type which can carry multiple optional fields.


--- middle

# Introduction

The QUIC Transport Protocol {{!RFC9000}} provides a secure, multiplexed
connection for transmitting reliable streams of application data.

This document defines an extension to the QUIC transport protocol which supports
reporting multiple packet receive timestamps using a new ACK_EXTENDED
frame type.


# Motivation

QUIC congestion control ({{?RFC9002}}) supports sampling round-trip time (RTT)
by measuring the time from when a packet was sent to when it is acknowledged.
However, more precise delay signals measured via packet receive timestamps have
the potential to improve the accuracy of network bandwidth measurements and the
effectiveness of congestion control, especially for latency-critical
applications such as real-time video conferencing or game streaming.

Numerous existing algorithms and techniques leverage receive receive timestamps
to improve transport performance. Examples include:

- The WebRTC congestion control algorithm described in {{?I-D.ietf-rmcat-gcc}}
  uses the difference between packet inter-departure and packet inter-arrival
  times as the input to its delay-based controller.

- The pathChirp ({{RRBNC}}) technique estimates available bandwidth by measuring
  inter-arrival time of multiple packets.

Notably, these techniques require receive timestamps for more than one packet
per round-trip in order to best measure the network.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Enabling Extensibility in the ACK frame {#extensibility}

The QUIC transport protocol defines two different frame types for acknowledgements
{{Section 19.3 of !RFC9000}}. The endpoint sending acknowledgements
decides which type to use depending on whether it wants to report ECN counts.
This approach works well with one set of optional fields, but grows exponentially
with more sets of optional fields.

This document defines a new set of optional fields to report receive timestamps.
Using a new frame type for each variant of the ACK frame would require adding 2 new
frame types, for a total of 4. Instead, this document defines one new frame type
(ACK_EXTENDED), that can carry multiple optional sections, reducing
the number of new frame types from 2 to 1, and avoids futre extensions causing
an exponential growth in frame types.



# ACK_EXTENDED Frame {#frame}

Endpoints send ACK_EXTENDED (type=0xB1 temporary value) frames in place of--and
in the
same manner as--regular ACK frames as described in {{Section 13.2 of !RFC9000}}.
ACK_EXTENDED frames contain additional fields to enable reporting two optional
sets of fields:
- ECN Counts (from {{Section 19.3 of !RFC9000}})
- Receive Timestamps (defined in this document)

ACK_EXTENDED frames are formatted as shown in {{fig-frame}}.

~~~
ACK_EXTENDED Frame {
  Type (i) = 0xB1
  // Fields of the existing ACK (type=0x02) frame:
  Largest Acknowledged (i),
  ACK Delay (i),
  ACK Range Count (i),
  First ACK Range (i),
  ACK Range (..) ...,
  Extended Ack Features (i),
  // Optional ECN counts (if bit 0 is set in Features)
  [ECN Counts (..)],
  // Optional Receive Timestamps (if bit 1 is set in Features)
  [Receive Timestamps (..)]
}
~~~
{: #fig-frame title="ACK_EXTENDED Frame Format"}

The fields Largest Acknowledged, ACK Delay, ACK Range Count, First ACK Range,
and ACK Range are the same as for ACK (type=0x02) frames specified in {{Section
19.3 of !RFC9000}}.

Extended Ack Features: A variable-length integer whose bit-wise
value indicates which optional fields are included in the ACK. This document
defines two sets of fields corresponding to the two least significant bits of
the value:
- Bit 0 indicates whether ECN count fields are included in the frame
- Bit 1 indicates whether Receive Timestamps are included in the frame

When Extended Ack Features has bit 0 set, the frame will include ECN Counts as
defined in {{Section 19.3.2 of !RFC9000}}.

When Extended Ack Feature has bit 1 set, the frame will include the following
additional fields for the receive timestmaps:

Timestamp Range Count:

: A variable-length integer specifying the number of Timestamp Range fields in
  the frame.

Timestamp Ranges:

: Ranges of receive timestamps for contiguous packets in descending packet
  number order; see {{ts-ranges}}.


## Timestamp Ranges {#ts-ranges}

Each Timestamp Range describes a series of contiguous packet receive timestamps
in descending sequential packet number (and descending timestamp) order.
Timestamp Ranges consist of a Gap indicating the largest packet number in the
range, followed by a list of Timestamp Deltas describing the relative receive
timestamps for each contiguous packet in the Timestamp Range (descending).

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

  The sum of Timestamp Delta Counts for all Timestamp Ranges in the frame MUST
  NOT exceed max_receive_timestamps_per_ack as specified in {{negotiation}}.

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
  received by the sender of the ACK_EXTENDED frame (see
  {{negotiation}}):

# Extension Negotiation {#negotiation}

The use of the ACK_EXTENDED frame is negotiated using the following
three transport parameters ({{Section 7.2 of !RFC9000}}):

extended_ack_features (0xff0a004 temporary value for draft use):

: A variable-length integer indicating the sending endpoint would like to
  receive ACK_EXTENDED frames with the specified set of features. The bit-wise
  value of the integer indicates the features the endpoint can accept in the
  ACK_EXTENDED frame. The value of this paramter is interpreted in the same way
  as the Extended Ack Features field in the ACK_EXTENDED frame, i.e.,
    - Bit 0 indicates whether ECN count fields are included in the frame
    - Bit 1 indicates whether Receive Timestamps are included in the frame

  Upon receiving this parameter, the receiving peer SHOULD send ACK_EXTENDED
  frames with the features supported by the endpoint which sent the parameter,
  and set the Extended Ack Features field accordingly. The receiving peer MAY
  choose to send ACK_EXTENDED frames with less features (and set the equivalent
  Extended Ack Features field value) if the requested features are not available
  or there is no data to send (e.g. if no timing information is available for
  receive timestamps). The receiving peer MAY send regular ACK and ACK_ECN
  frames, in which case the endpoint MUST still support processing regular ACK
  frames as defined by {{Section 19.3 of !RFC9000}}.

  ACK_EXTENDED frames written by the peer receiving this parameter MUST not have
  bits set in the Extended Ack Features that were not set by the sending
  endpoint
  in the transport parameter. An endpoint receiving an ACK_EXTENDED frame with
  features it did not include in the transport parameter should terminate the
  connection with a PROTOCOL_VIOLATION violation error.

  The receiving peer SHOULD NOT send ACK_EXTENDED frames with 0 features.

max_receive_timestamps_per_ack (0xff0a002 temporary value for draft use):

: A variable-length integer indicating that the maximum number of receive
  timestamps the sending endpoint would like to receive in an ACK_EXTENDED
  frame.

  The sending endpoint MUST sent this transport parameter if sends the
  extended_ack_features transport parameter with bit 1 set in its value. A peer
  that receives this transport parameter but not an extended_ack_features
  transport paramter with bit 1 set MUST ignore this transport parameter.

  Each ACK_EXTENDED frame sent MUST NOT contain more than the
  specified maximum number of receive timestamps, but MAY contain fewer or none.

receive_timestamps_exponent (0xff0a003 temporary value for draft use):

: A variable-length integer indicating the exponent to be used when encoding and
  decoding timestamp delta fields in ACK_EXTENDED frames sent by the
  peer (see {{ts-ranges}}). If this value is absent, a default value of 0 is
  assumed (indicating microsecond precision). Values above 20 are invalid.


## Receive Timestamp Basis {#ts-basis}

Endpoints which send ACK_EXTENDED frames must determine a value,
receive_timestamp_basis, relative to which all receive timestamps for the
session will be reported (see {{ts-ranges}}).

The value of receive_timestamp_basis MUST be less than the smallest receive
timestamp reported, and MUST remain constant for the entire duration of the
session. The receive_timestamp_basis is a local value that is not communicated
to the peer.

Receive timestamps are reported relative to the basis, rather than in absolute
time to avoid requiring clock synchronization between endpoints and to make
the frame more compact.


# Discussion

## Best-Effort Behavior

Receive timestamps are sent on a best-effort basis and endpoints MUST gracefully
handle scenarios where receive timestamp information for sent packets is not
received. Examples of such scenarios are:

- The packet containing the ACK_EXTENDED frame is lost.

- The sender truncates the number of timestamps sent in order to (a) avoid
  sending more than max_receive_timestamps_per_ack ({{negotiation}}); or (b) fit
  the ACK frame into a packet.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
