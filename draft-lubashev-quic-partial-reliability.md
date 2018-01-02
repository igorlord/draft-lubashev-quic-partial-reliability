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

Some applications, especially applications with real-time
requirements, need a partially reliable transport.  These applications
typically communicate data in application-specific messages that are
serialized over QUIC streams.  Applications desire partially reliable
transport, when their messages expire and lose their usefulness due to
later events (time passing, newer messages, etc).

The content of this draft is intended for
{{!I-D.ietf-quic-transport}}, {{!I-D.ietf-quic-recovery}} and,
{{!I-D.ietf-quic-applicability}}.

The key to partial reliablity is notifying the peer about data that
will not be retransmitted and managing flow control for the
connection.


Partially Reliabile Streams
===========================

It is possible to provide partial reliablity without any changes to
QUIC transport by using QUIC streams, encoding one message per QUIC
stream.  When the message expires, the sender can reset the stream,
causing RST_STREAM frame to be transmitted, unless all data in the
stream has already been fully acknowledged.  The problem with this
approach is that messages transmitted by the application typically
belong to a message stream, and applications may need to support
multiple concurrent message streams.  Hence, a message-per-stream
approach requires each message to contain an extra header portion to
associate the message with a logical application stream.  In case of
short messages, this approach introduces a significant overhead due to
STREAM frames and message headers. It also places the burden on the
application to reorder data arriving on multiple QUIC streams.
Furthermore, splitting each application stream into multiple QUIC
streams renders QUIC per-stream flow control ineffective and requires
an application to build its own.

An alternative is the proposed single-stream mechanism that keeps
messages arriving in order on a single stream.

Min Stream Offset     {#min-stream-offset}
-----------------

This proposal introduces a new QUIC stream variable "Min Stream
Offset" that indicates the smallest retransmittable data offset.  The
receiver SHOULD NOT wait for any data at offsets smaller than Min
Stream Offset to be retransmitted by the sender.  Initially, Min
Stream Offset is 0 for all streams.


MIN_STREAM_DATA Frame     {#min_stream_data}
---------------------

The MIN_STREAM_DATA frame (types 0x?? (type) and 0x?? (type+1)) is
used in flow control to inform the peer of the minimum
(re-)transmittable data offset on a stream.  If the least significant
bit is set, Unsent Bytes field is present in the frame.

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

If Unsent Bytes field is present in the frame, it implies that all
data previously sent to the receiver on the stream has expired.
Hence, Sent Data is implicitly the Largest Sent Data on the stream.

Min Stream Offset ({{min-stream-offset}}) for Stream ID is determined
by the formula:

~~~
  Min Stream Offset = Sent Data + Unsent Bytes
~~~

The Min Stream Offset for a stream MUST NOT be reduced by the sender
in a subsequent MIN_STREAM_DATA frame, but loss and reordering can
cause MIN_STREAM_DATA frames to be received out of order.
MIN_STREAM_DATA frames that do not increase the stream's Min Stream
Offset MUST be ignored.


Effect of MIN_STREAM_DATA on Flow Control   {#flow-control}
=========================================

Specifying Unsent Bytes separately from Send Data in MIN_STREAM_DATA
frame is done to avoid using its stream and connection flow control
credits to notify the other endpoint of bytes that have never been and
will never be transmitted.  Hence, a sender that desires to expire a
large number of bytes that have never been transmitted can do so in a
single frame without closing down the connection flow control window,
affecting other streams, and without a risk of exceeding its stream or
connection flow control credits.


Sender Flow Control    {#flow-control-sender}
-------------------

When an ACK frame is received for a packet containing a
MIN_STREAM_OFFSET frame, and the current Largest Data Sent of the
stream is smaller than Min Stream Offset of the acknowledged
MIN_STREAM_OFFSET frame, the Largest Data Sent is advanced to the Min
Stream Offset.  Note that this can only happen, when Unsent Bytes is
non-zero in the MIN_STREAM_OFFSET frame.

If the Largest Data Sent for at least one stream was advanced due to
the receipt of a packet with an ACK frame, and, after processing that
entire packet, the sum of the Largest Data Sent on all streams -
including streams in terminal states but excluding stream 0 - exceeds
MAX_DATA, the sender MUST terminate a connection with a
QUIC_FLOW_CONTROL_SENT_TOO_MUCH_DATA error.

It is possible that the Largest Data Sent for at least one stream will
be advanced past MAX_STREAM_DATA for that stream.  In that case, no
more data octets can be sent on the stream until a MAX_STREAM_DATA
frame advancing the maximum offset is received.  Note that this does
not prohibit using the Largest Data Sent beyond MAX_STREAM_DATA in an
RST_STREAM frame, a MIN_STREAM_DATA frame, or a STREAM frame with no
data and FIN bit set.


TODO: Work on this!

Receiver Flow Control    {#flow-control-receiver}
---------------------

The value of the Largest Received Offset on the stream is immediately
advanced, if it is smaller than Sent Data.

When computing window updates for MAX_STREAM_DATA and MAX_DATA frames,
the receiver SHOULD use the larger of Min Stream Offset and Largest
Received Offset for the stream.

ACK of MIN_STREAM_DATA    {#ack-min-stream-data}
----------------------

Sending an ACK frame for a packet containing MIN_STREAM_DATA frame that advanced Min Stream Offset past Largest

Received Offset for the stream causes Largest
Received Offset to be


ACK of MIN_STREAM_DATA and MAX_DATA    {#ack-max-data}
-----------------------------------



FIXME START: Must ensure that connection window can only be used by
the gap in the stream!

  When computing window updates for MAX_STREAM_DATA and MAX_DATA, the
  receiver SHOULD use the larger of "Min Stream Offset" and "the
  largest data offset" received for the stream.

FIXME END


Sender Interface and Behavior    {#sender}
=============================

It is recommended that a QUIC library API provides a way for a sender
to update the minimum retransmittable offset for a stream.  A typical
sender would call an API function providing this functionality
whenever any data previously enqueued for transmission expires, per
application semantics.  The sender would keep track of the message
boundaries and request expiration of data on a message boundary.

If all data between the current Min Stream Offset and the new Min
Stream Offset has been acknowledged, no action is performed by the
sender's QUIC implementation.  Otherwise, if there is unacknowledged
data, a MIN_STREAM_DATA frame is transmitted.

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

The receiver SHOULD assume that none of the data up to Min Stream
Offset ({{min-stream-offset}}) will be retransmitted.

It is recommended that a QUIC library API provides a way for a
receiver to obtain the length of a gap corresponding to the expired
data in addition to data octets that follow the gap.

A receiver MAY discard any stream data received for an offset smaller
than Min Stream Offset.


Retransmission of MIN_STREAM_DATA    {#retransmission}
=================================

The most recent MIN_STREAM_DATA frame for a stream MUST be
retransmitted until the sender is certain that the receiver is not
expecting retransmission of any expired data.  I.e. the frame MUST be
retransmitted until either the stream enters "half-closed (local)"
state or all data between the largest acknowledged Min Stream Offset
and the current Min Stream Offset has been acknowledged.  Note that
the later condition includes the trivial case of receiving an
acknowledgment for the latest MIN_STREAM_DATA frame.


IANA Considerations   {#iana}
===================

This document has no actions for IANA.


Security Considerations   {#security}
=======================

This document has no new security considerations.


Acknowledgments
===============

Many thanks to Mike Bishop for his feedback on flow control issues and
proofreading the first draft.  Kudos to the QUIC working group for a
mountain of feedback on this draft and for diligently plowing through
hard problems and making thousands of big and small decisions to make
the Internet better for everyone.


--- back
