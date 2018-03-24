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
{{!I-D.ietf-quic-applicability}} for QUIC V2.

The key to partial reliability is notifying the peer about data that
will not be retransmitted and managing flow control for the
connection.


Partially Reliable Streams
==========================

It is possible to provide partial reliability without any changes to
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

In this proposal, both the sender and the receiver are able to control
expiration of messages in a stream.

This proposal introduces two new QUIC per-stream variables: Min Stream
Offset ({{min-stream-offset}}) and Exempt Stream Bytes
({{exempt-stream-bytes}}).


Min Stream Offset     {#min-stream-offset}
-----------------

Min Stream Offset indicates the smallest retransmittable data offset
for the stream.  The receiver SHOULD NOT wait for any data at offsets
smaller than Min Stream Offset to be (re-)transmitted by the sender.
The sender SHOULD NOT send any data at offsets smaller than Min Stream
Offset.  Initially, Min Stream Offset is 0 for all streams.


Exempt Stream Bytes      {#exempt-stream-bytes}
-------------------

Exempt Stream Bytes is the number of bytes sent on the stream that do
not count toward connection flow control limit.  Initially, Exempt
Stream Bytes is 0 for all streams.


New and Updated Frames
======================

This proposal updates MAX_STREAM_DATA ({{frame-max-stream-data}})
frame and introduces a new MIN_STREAM_DATA frame
({{frame-min-stream-data}}).


## MAX_STREAM_DATA Frame     {#frame-max-stream-data}

The text with *emphasis* is added to the description of MAX_STREAM_DATA
frame.  Also, optional Minimum Stream Offset and Exempt Stream Bytes
field sare added to MAX_STREAM_DATA frame.

The MAX_STREAM_DATA frame (type=0x05 or type=0xXY) is used in flow
control to inform a peer of the maximum amount of data that can be
sent on a stream *and, optionally, update Min Stream Offset
({{min-stream-offset}}) and Exempt Stream Bytes
({{exempt-stream-bytes}}) for this stream*.

*If type=0xXY, Minimum Stream Offset and Exempt Stream Bytes fields
are also present in the frame.*

An endpoint that receives a MAX_STREAM_DATA frame for a receive-only stream
MUST terminate the connection with error PROTOCOL_VIOLATION.

An endpoint that receives a MAX_STREAM_DATA frame for a send-only stream
it has not opened MUST terminate the connection with error PROTOCOL_VIOLATION.

Note that an endpoint may legally receive a MAX_STREAM_DATA frame on a
bidirectional stream it has not opened.

The frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Maximum Stream Data (i)                  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Minimum Stream Offset (i)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Exempt Stream Bytes (i)                  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields in the MAX_STREAM_DATA frame are as follows:

Stream ID:

: The stream ID of the stream that is affected encoded as a variable-length
  integer.

Maximum Stream Data:

: A variable-length integer indicating the maximum amount of data that can be
  sent on the identified stream, in units of octets.

Minimum Stream Offset:

: *A variable-length integer indicating the minimum offset of the stream data
  that can be sent (or re-transmitted) on the identified stream, in units of
  octets.*

Exempt Stream Bytes:

: *A variable-length integer indicating the amount of data on the identified
  stream exempt from connection flow control, in units of octets.*

When counting data toward this limit, an endpoint accounts for the largest
received offset of data that is sent or received on the stream.  Loss or
reordering can mean that the largest received offset on a stream can be greater
than the total size of data received on that stream.  Receiving STREAM frames
might not increase the largest received offset.

The data sent on a stream MUST NOT exceed the largest maximum stream data value
advertised by the receiver.  An endpoint MUST terminate a connection with a
FLOW_CONTROL_ERROR error if it receives more data than the largest maximum
stream data that it has sent for the affected stream, unless this is a result of
a change in the initial limits (see ((zerortt-parameters))).

*If Minimum Stream Offset and Exempt Stream Bytes fields are present:*

1. *Since Stream 0 MUST be reliable, Stream ID MUST NOT be 0.*

2. *They update the stream's Min Stream Offset and Exempt Stream Bytes
   upon receipt.*

3. *If current send offset for the stream is less than the new Min Stream Offset,
   the current send offset for the stream is set to be the new Min Stream Offset
   upon receipt.*

*The receiver MUST NOT reduce the Maximum Stream Data, Min Stream Offset, and
Exempt Stream Bytes, but loss and reordering can cause MAX_STREAM_DATA frames to
be received out of order.  If Maximum Stream Data field does not advance the
maximum amount of data that can be sent on the stream, or Minimum Stream Offset
field does not advance Min Stream Offset, or Exempt Stream Bytes field does not
advance Exempt Stream Bytes, the corresponding stream parameter is not updated.
An endpoint MUST terminate a connection with a MAX_STREAM_DATA_ERROR error, if
one of the three fields is advancing its stream parameter, while another field
is trying to retard its stream parameter.  An endpoint MUST terminate a
connection with a MAX_STREAM_DATA_ERROR error, if Maximum Stream Data is less
than Minimum Stream Offset.*


MIN_STREAM_DATA Frame     {#frame-min-stream-data}
---------------------

The MIN_STREAM_DATA frame (type=0xXY) is used in flow control by the
sender to inform the receiver of the minimum (re-)transmittable data
offset on a stream.

The frame is as follows:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Minimum Stream Offset (i)                  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The fields in the MIN_STREAM_DATA frame are as follows:

Stream ID:

: The stream ID of the stream that is affected encoded as a variable-length
  integer.

Minimum Stream Offset:

: A variable-length integer indicating the new value for the stream's Min Stream
  Offset ({{min-stream-offset}})

Since Stream 0 MUST be reliable, Stream ID MUST NOT be 0.

The sender MUST NOT reduce the Minimum Stream Offset for a stream, but loss and
reordering can cause MIN_STREAM_DATA frames to be received out of order.
MIN_STREAM_DATA frames that do not increase the stream's Min Stream Offset MUST
be ignored.

Minimum Stream Offset can exceed the stream's maximum data offset.

Sending MIN_STREAM_DATA frame does not change the stream's current send offset.

Upon receipt, if the current receive offset for the stream is less the Minimum
Stream Offset field, the receiver MUST advance the stream's Exempt Stream Bytes
and the current receive offset by the difference between Minimum Stream Offset
field and the current receive offset for this stream.


Flow Control Update      {#flow-control}
===================

Flow control changes are designed to make sure that a sender that
desires to expire a large number of bytes that have never been
transmitted can do so efficiently and without closing down the
connection flow control window (thereby blocking other streams).  That
must be done in a way that does not open up connection flow control
window, allowing a different stream to use connection credits not
designed for it.


Connection Flow Control    {#flow-control-connection}
-----------------------

The connection flow control calculation is redefined as the sum of the
current stream offsets minus the sum of Exempt Stream Bytes for all
streams, including closed streams but excluding stream 0.


Sender Flow Control    {#flow-control-sender}
-------------------

When an application notifies QUIC transport of the minimum
retransmittable offset for a stream beyond the current Min Stream Data,
sender SHOULD advance Min Stream Data for the stream. If there is any
unacknowledged (including unsent) data at an offset less than the new
Min Stream Data, the sender SHOULD transmit a MIN_STREAM_DATA frame
({{frame-min-stream-data}}).

If the last sent MIN_STREAM_DATA frame for a stream is declared lost,
it MUST be retransmitted.

The sender behavior upon receipt of MAX_STREAM_DATA is described in
{{frame-max-stream-data}}.  It is possible that MAX_STREAM_DATA will
update current send stream offset and Exempt Stream Bytes for a
"half-closed (local)" stream.


Receiver Flow Control    {#flow-control-receiver}
---------------------

Upon receipt of a MIN_STREAM_DATA frame ({{frame-min-stream-data}}) that
advances Min Stream Offset past the current receive offset for a stream,
the receiver SHOULD send a MAX_STREAM_DATA frame
({{frame-max-stream-data}}).

If a STREAM-with-FIN or an RESET_STREAM frame is received with the final
stream offset less than current receive offset for a stream, it is only
an error, if the final receive offset for the stream is less than
largest offset learned from a STREAM or RESET_STREAM frames.


Sender Interface and Behavior    {#sender}
=============================

It is recommended that a QUIC library API provides a way for a sender
to update the minimum retransmittable offset for a stream.  A typical
sender would call this API function whenever data previously enqueued
for transmission expires, per application semantics.  The sender would
keep track of the message boundaries and request expiration of data on
a message boundary.

If all data between the current Min Stream Offset and the new Min
Stream Offset has been acknowledged, no action is performed by the
sender's QUIC transport.  Otherwise, if there is unacknowledged data,
a MIN_STREAM_DATA frame ({{frame-min-stream-data}}) is transmitted.

An application may decide to conditionally expire messages based on
the delivery status of prior messages.  For example, an application
may wish to ensure that its large messages are delivered at least at a
given minimum rate before expiring a partially-delivered message just
because there is a newer message to deliver.  That is, if the rate of
data the application wishes to write exceeds the network's throughput,
the application may want to ensure that at least some messages are
delivered in their entirety.  To support this use case, it is
recommended that a QUIC library API provides a way for the sender to
monitor the smallest unacknowledged stream offset greater than Min
Stream Offset ({{min-stream-offset}}).


Receiver Interface and Behavior   {#receiver}
===============================

The receiver SHOULD assume that none of the data up to Min Stream Offset
({{min-stream-offset}}) will be retransmitted.  A receiver MAY discard
any stream data received for an offset smaller than Min Stream Offset.

It is recommended that a QUIC library API provides a way for a
receiver application to obtain the length of a gap corresponding to
the expired data in addition to data octets that follow the gap.

It is also recommended that a QUIC library API provide a way for a
receiver application to skip some octets past the current point in the
stream.  When some of the skipped octets have not, yet, been received,
the receiver SHOULD advance Min Stream Data for the stream. If the
current receive offset for the stream is less the Minimum Stream Offset
field, the receiver MUST advance the stream's Exempt Stream Bytes and
the current receive offset by the difference between Minimum Stream
Offset field and the current receive offset for this stream. If the
stream's Min Stream Data has been advanced, the receiver SHOULD send a
MAX_STREAM_DATA frame ({{frame-max-stream-data}}).


Retransmission      {#retransmission}
==============

Both MAX_STREAM_DATA and MIN_STREAM_DATA frames MUST be retransmitted if
declared lost.

Retransmission of MAX_STREAM_DATA    {#retransmission-max-stream-data}
---------------------------------

The most recent MIN_STREAM_DATA frame that contains Minimum Stream
Offset and Exempt Stream Bytes MUST be retransmitted until the receiver
is certain that the sender is not going to transmit any skipped data.
I.e. the frame MUST be retransmitted until either the stream enters
"half-closed (remote)" state, or all data between the largest Minimum
Stream Offset in an acknowledged MAX_STREAM_DATA frame and the current
Min Stream Offset has been received, or all data between the largest
Minimum Stream Offset in a received MIN_STREAM_DATA frame and the
current Min Stream Offset has been received.


Retransmission of MIN_STREAM_DATA    {#retransmission-min-stream-data}
---------------------------------

The most recent MIN_STREAM_DATA frame for a stream MUST be retransmitted
until the sender is certain that the receiver is not expecting
retransmission of any expired data.  I.e. the frame MUST be
retransmitted until either the stream enters "half-closed (local)"
state, or all data between the largest Minimum Stream Offset in an
acknowledged MIN_STREAM_DATA frame and the current Min Stream Offset has
been acknowledged, or all data between the largest Minimum Stream Offset
in a received MAX_STREAM_DATA frame and the current Min Stream Offset
has been acknowledged.


IANA Considerations   {#iana}
===================

This document has no actions for IANA.


Security Considerations   {#security}
=======================

This document has no new security considerations.


Acknowledgments
===============

Many thanks to Mike Bishop and Ian Swett for their feedback on flow
control issues.  Kudos to the QUIC working group for a mountain of
feedback on this draft and for diligently plowing through hard
problems, making thousands of big and small decisions, to make the
Internet better for everyone.


--- back
