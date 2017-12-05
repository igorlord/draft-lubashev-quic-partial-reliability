---
title: Partially Reliable Streams for QUIC
abbrev: quic-pr
docname: draft-lubashev-quic-partial-reliability-latest
date: {DATE}
category: info

ipr: trust200902
area: General
workgroup: QUIC
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  compact: yes
  inline: yes
  text-list-symbols: -o*+


author:
 -
    ins: I. Lubashev
    name: Igor Lubashev
    org: Akamai Technologies
    email: igorlord@alum.mit.edu

normative:

informative:

--- abstract

This memo introduces a MIN_STREAM_DATA frame to enable partial
reliability for QUIC streams.  The MIN_STREAM_DATA frame allows a
sender to give up on retransmitting older parts of a stream and to
notify the receiver about this decision. The content of this draft is
intended for merging into QUIC transport, recovery, and applicability
drafts.


--- middle

Notational Conventions    {#conventions}
======================

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}.


Introduction     {#introduction}
============

Some applications, especially applications with real-time requirments,
need a partially reliable transport.  These applications typically
communicate data in application-specific messages that are serialized
over QUIC streams.  Applications desire partially reliable transport,
when their messages expire and lose their usefulness upon timer
expirations, becoming superseeded by newer messages, or other events
external to the transport.

The content of this draft is intended for
{{!I-D.ietf-quic-transport}}, {{!I-D.ietf-quic-recovery}} and,
{{!I-D.ietf-quic-applicability}}.

The key to partial reliablity is notifying the transport and the peer
when data, previously enqueued for transmission, no longer needs to be
transmitted.

Multi-Stream Approach
---------------------

It is possible to provide partial reliablity without any changes to
QUIC transport by using QUIC streams, encoding one message per QUIC
stream.  When the message expires, the sender can resent the stream,
causing RST_STREAM frame to be transmitted, unless all data in the
stream has already been fully acknowledged.  The problem with this
approach is that messages transmitted by the application are typically
not independent but a part of a message stream, and applications may
need to support multiple concurrent message streams.  Hence, a
message-per-stream approach requires each message to contain an extra
header portion to associate the message with a logical application
stream.  In case of short messages, this approach introduces a
significant overhead.

Single-Stream Approach
----------------------

An alternative is the proposed single-stream mechanism that keeps
messages arriving in order on a single stream.


Min Stream Offset     {#min-stream-offset}
=================

This proposal introduces a new QUIC stream variable "Min Stream
Offset" that indicates the smallest retransmittable data offset.  The
receiver SHOULD NOT wait for any data at offsets smaller than Min
Stream Offset to be retransmitted by the sender.  Initially, Min
Stream Offset is 0 for all streams.


MIN_STREAM_DATA Frame     {#min_stream_data}
=====================

The MIN_STREAM_DATA frame (types 0x?? and 0x??) is used in flow
control to inform the peer of the minimum (re-)transmittable data
offset on a stream.  If the least significant bit is set, Unsent Bytes
field is present in the frame.

The frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sent Data (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Unsent Bytes (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields in the MIN_STREAM_DATA frame are as follows:

Stream ID:

: The stream ID of the stream that is affected encoded as a variable-length
  integer.

Sent Data:

: A variable-length integer indicating the number of data octets
  written to the stream (since the beginning of the stream) that have
  expired and will not be retransmitted.

Unsent Bytes:

: A variable-length integer indicating the number of data octets past
  Sent Data that have expired but have never been sent and will not be
  transmitted.  If Unsent Bytes field is absent, it is presumed to be
  0.

Since Stream 0 MUST be reliable, Stream ID MUST NOT be 0.  If Unsent
Bytes field is present in the frame, it MUST NOT be 0 (reserved for
the future use).

Min Stream Offset ({{min-stream-offset}}) for Stream ID is determined
by the formula:

~~~
  Min Stream Offset = MAX(Min Stream Offset, Sent Data + Unsent Bytes)
~~~


Effect of MIN_STREAM_DATA on Flow Control   {#flow-control}
=========================================

The value of Sent Data field MUST be used in flow control accounting.
A receipt of a MIN_STREAM_DATA frame MUST advance the stream and
connection flow control credits, if the maximum data offset received
on Stream ID is less than Sent Data.

Also, a receipt of MIN_STREAM_DATA with Min Stream Offset beyond the
stream's MAX_STREAM_DATA SHOULD be treated as a receipt of a
STREAM_BLOCKED frame.


Sender Interface and Behavior    {#sender}
=============================

It is recommended that a QUIC library API provides a way for a sender
to update the minimum retransmittable offset for a stream.  A typical
sender would call an API function providing this functionality
whenever any data previously enqueued for transmission expires, per
application semantics.  The sender would keep track of the message
boundaries of such data.  If all data between the current Min Stream
Offset and the new Min Stream Offset has been acknowledged, no action
is performed by the sender's QUIC implementation.  Otherwise, if there
is unacknowledged data, a MIN_STREAM_DATA frame is transmitted.

An application may decide to conditionally expire messages based on
the delivery status of prior messages.  For example, an application
may wish to ensure that its large messages are delivered at least at a
given minimum rate before expiring a partially-delivered message just
because their is a newer message to deliver.  That is, if the rate of
data the application wishes to write exceeds the network's throughput,
the application may want to ensure that at least some messages are
delivered in their entirety.  To support this use case, it is
recommended that a QUIC library API provides a way for the sender to
minitor the smallest unacknowledged stream offset greater than Min
Stream Offset ({{min-stream-offset}}).


Receiver Interface and Behavior   {#receiver}
===============================

A receiver of MIN_STREAM_DATA MUST use Sent Data for flow control
accounting (see {{flow-control}}).  The receiver SHOULD assume that
none of the data up to Min Stream Offset ({{min-stream-offset}}) will
be retransmitted.

When computing window updates for MAX_STREAM_DATA and MAX_DATA, the
receiver SHOULD use the larger of "Min Stream Offset" and "the largest
data offset" received for the stream.

It is recommended that a QUIC library API provides a way for a
receiver to obtain the length of a gap corresponding to the expired
data instead of (or in addition to) data octets that follow the gap.

A receiver MAY discard any stream data received for an offset smaller
than Min Stream Offset.


IANA Considerations   {#iana}
===================

This document has no actions for IANA.


Security Considerations   {#security}
=======================

This document has no new security considerations.


Acknowledgments
===============

Many thanks to Mike Bishop for his feedback on flow control issues and
proofreading this draft.  Kudos to the QUIC working group for
dilligently plowing through hard problems and making thousands of big
and small decisions to make the Internet better for everyone.


--- back
