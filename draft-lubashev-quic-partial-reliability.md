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

This memo introduces MIN_STREAM_DATA and EXPIRED_STREAM_DATA frames to enable
partial reliability for QUIC streams.  The EXPIRED_STREAM_DATA frame allows a
sender to give up on retransmitting older parts of a stream and to notify the
receiver about this decision. The MIN_STREAM_DATA frame allows a receiver to
express its disinterest in older parts of a stream.  The content of this draft
is intended for merging into QUIC transport, recovery, and applicability drafts
as a negotiable extension and/or QUIC Version 2 transport feature.


--- middle

Notational Conventions    {#conventions}
======================

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.


Introduction     {#introduction}
============

Some applications, especially applications with near real-time requirements,
need a partially reliable transport.  These applications typically communicate
data in application-specific messages that are serialized over QUIC streams.
Applications desire partially reliable transport when their messages expire and
lose their usefulness due to later events (time passing, newer messages, etc).

The content of this draft is intended for {{!I-D.ietf-quic-transport}},
{{!I-D.ietf-quic-recovery}} and, {{!I-D.ietf-quic-applicability}} as a QUIC
extension and/or QUIC Version 2.

The key to partial reliability is notifying the peer about data that will not or
should not be retransmitted and managing flow control for the connection.


Partially Reliable Streams
==========================

It is possible to provide partial reliability without any changes to QUIC
transport by using QUIC streams, encoding one message per QUIC stream.  When a
message expires, the sender can reset the stream, causing RST_STREAM frame to be
transmitted, unless all data in the stream has already been fully acknowledged.
Likewise, the receiver can send STOP_SENDING frame to indicate its disinterest
in the message.  The problem with this approach is that messages transmitted by
the application typically belong to a message stream, and applications may need
to support multiple concurrent message streams.  Hence, a message-per-stream
approach requires each message to contain an extra header portion to associate
the message with a logical application stream.  In case of short messages, this
approach introduces a significant overhead due to STREAM frames and message
headers. It also places the burden on the application to reorder data arriving
on multiple QUIC streams.  Furthermore, splitting each application stream into
multiple QUIC streams renders QUIC's per-stream flow control ineffective and
requires an application to build its own.

An alternative is the proposed single-stream mechanism that keeps messages
arriving in order on a single stream.

In this proposal, both the sender and the receiver are able to control
expiration of messages in a stream.

To facilitate flow control, this proposal introduces a new QUIC per-stream
value: Exempt Stream Bytes ({{exempt-stream-bytes}}).


Exempt Stream Bytes      {#exempt-stream-bytes}
-------------------

Exempt Stream Bytes is the number of bytes sent on the stream that do
not count toward connection flow control limit.  Initially, Exempt
Stream Bytes is 0 for all streams.


Minimum retransmittable offset and current receive offset    {#offsets}
---------------------------------------------------------

For fully reliable streams, the smallest unacknowledged data offset is treated
by the sender to be the minimum (re-)transmittable offset.  Likewise, the
current receive offset for a stream is the smallest data offset that has not
been received by the receiver.  Note that due to loss and reordering, the
current receive offset may be smaller than the largest received offset.

Partially reliable streams allow the sender to advance its minimum
retransmittable offset and notify the receiver to advance its current receive
offset.  The receiver can also advance its current receive offset and notify the
sender to advance its minimum retransmittable offset.


New Frames
==========

This introduces new MIN_STREAM_DATA ({{frame-min-stream-data}}) and
EXPIRED_STREAM_DATA ({{frame-expired-stream-data}}) frames.


## MIN_STREAM_DATA Frame     {#frame-min-stream-data}

The MIN_STREAM_DATA frame (type=0x??) is used in flow control by a receiver to
inform a sender of the maximum amount of data that can be sent on a stream (like
MAX_STREAM_DATA frame) and to request an update to the minimum retransmittable
offset ({{offsets}}) and Exempt Stream Bytes value ({{exempt-stream-bytes}}) for
this stream.

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

The fields in the MIN_STREAM_DATA frame are as follows:

Stream ID:

: The stream ID of the stream that is affected encoded as a variable-length
  integer.

Maximum Stream Data:

: A variable-length integer indicating the maximum amount of data that can be
  sent on the identified stream, in units of octets.

Minimum Stream Offset:

: A variable-length integer indicating the minimum offset of the stream data
  that the receiver is expecting to receive on the identified stream, in units
  of octets.

Exempt Stream Bytes:

: A variable-length integer indicating the amount of data on the identified
  stream exempt from connection flow control, in units of octets.

The semantics of Maximum Stream Data field is identical to that of Maximum
Stream Data field in MAX_STREAM_DATA frame.

Since Stream 0 MUST be reliable, Stream ID MUST NOT be 0.

Upon receipt of a MIN_STREAM_DATA frame, the sender advances the maximum amount
of data that can be sent on the stream, the minimum retransmittable offset, and
the Exempt Stream Bytes value to the corresponding values of Maximum Stream
Data, Minimum Stream Offset, and Exempt Stream Bytes fields.

If the current send offset becomes smaller than the minimum retransmittable
offset for a stream, the current send offset is advanced to the minimum
retransmittable offset.

The receiver MUST NOT reduce the maximum stream data value, minimum
retransmittable offset, and Exempt Stream Bytes value for the stream, but loss
and reordering can cause MIN_STREAM_DATA frames to be received out of order.  If
Maximum Stream Data field does not advance the maximum amount of data that can
be sent on the stream, or Minimum Stream Offset field does not advance the
minimum retransmittable offset, or Exempt Stream Bytes field does not advance
Exempt Stream Bytes value, the corresponding stream parameter is not updated.

A MIN_STREAM_DATA referencing a closed or a "half-closed (local)" stream SHOULD
be ignored.

An endpoint that receives a MIN_STREAM_DATA frame for a receive-only stream MUST
terminate the connection with error PROTOCOL_VIOLATION.

An endpoint that receives a MIN_STREAM_DATA frame for a send-only stream it has
not opened MUST terminate the connection with error PROTOCOL_VIOLATION.

Note that an endpoint may legally receive a MIN_STREAM_DATA frame on a
bidirectional stream it has not opened.

An endpoint MUST terminate a connection with a MIN_STREAM_DATA_ERROR error, if
one of the three fields is advancing its stream parameter, while another field
is trying to retard its stream parameter.  An endpoint MUST terminate a
connection with a MIN_STREAM_DATA_ERROR error, if Maximum Stream Data field is
smaller than Minimum Stream Offset field or Minimum Stream Offset field is
smaller than Exempt Stream Bytes field.


EXPIRED_STREAM_DATA Frame     {#frame-expired-stream-data}
-------------------------

The EXPIRED_STREAM_DATA frame (type=0x??) is used in flow control by a sender to
inform a receiver of the minimum retransmittable offset ({{offsets}}) for a
stream.

Sending EXPIRED_STREAM_DATA frame does not change the stream's current send
offset.

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

The fields in the EXPIRED_STREAM_DATA frame are as follows:

Stream ID:

: The stream ID of the stream that is affected encoded as a variable-length
  integer.

Minimum Stream Offset:

: A variable-length integer indicating the minimum offset of the stream data
  that will sent (or re-transmitted) on the identified stream, in units of
  octets.

Since Stream 0 MUST be reliable, Stream ID MUST NOT be 0.

Upon receipt of an EXPIRED_STREAM_DATA frame, the receiver advances the current
receive offset for the stream to be Minimum Stream Offset value.

The sender MUST NOT reduce the minimum retransmittable offset for a stream, but
loss and reordering can cause EXPIRED_STREAM_DATA frames to be received out of
order.  EXPIRED_STREAM_DATA frames that do not advance the current receive
offset for the stream MUST be ignored.

If the largest received offset for the stream becomes smaller than the current
receive offset, the receiver MUST advance the stream's Exempt Stream Bytes value
by the difference between the current and the largest received offsets.  The
largest received offset is then set to match the current receive offset.

Upon receipt of an EXPIRED_STREAM_DATA frame that advances the Exempt Stream
Bytes value for the stream, the receiver SHOULD send a MIN_STREAM_DATA frame
({{frame-min-stream-data}}).

Note that receipt of an EXPIRED_STREAM_DATA frame may cause the current receive
offset (and hence the largest received offset) to exceed a previously advertised
maximum stream data value for the stream.

An endpoint that receives an EXPIRED_STREAM_DATA frame for a send-only stream
MUST terminate the connection with error PROTOCOL_VIOLATION.


Flow Control Update      {#flow-control}
===================

Flow control changes are designed to allow a sender that desires to expire a
large number of bytes that have never been transmitted to do so efficiently and
without closing down the connection flow control window (thereby blocking other
streams).  That must be done in a way that does not open up the connection flow
control window, allowing a different stream to use connection credits not
designed for it.


Connection Flow Control    {#flow-control-connection}
-----------------------

The connection flow control calculation is redefined to be the sum of the
current stream offsets (current send offset for the sender and the largest
received offset for the receiver) minus the sum of Exempt Stream Bytes values
({{exempt-stream-bytes}}) for all streams, including closed streams but
excluding stream 0.


Stream Final Offset
-------------------

If a STREAM-with-FIN or an RST_STREAM frame is received with the final stream
offset smaller than largest received offset for a stream, it is only an error,
if the final receive offset for the stream is smaller than largest offset
learned from a STREAM or RST_STREAM frames.  If the final stream offset is
smaller than the largest received offset, the final stream offset is advanced to
be the largest received offset.


QUIC Interface and Behavior      {#interface}
=============================

QUIC library interface needs to expose additional APIs to allow applications to
take advantage of partially reliable streams.


Sender Interface and Behavior    {#sender-interface}
-----------------------------

It is recommended that a QUIC library provides a way for a sender to update the
minimum retransmittable offset ({{offsets}}) for a stream.  A typical sender
would call this API function whenever data previously enqueued for transmission
expires, per application semantics.  The sender would keep track of the message
boundaries and request expiration of data on a message boundary.

When an application instructs its QUIC transport to advance the minimum
retransmittable offset for a stream, and there there is any unacknowledged data
(including unsent data) at an offset smaller than the new minimum
retransmittable offset, the sender SHOULD transmit a EXPIRED_STREAM_DATA frame
({{frame-expired-stream-data}}).

An application may decide to conditionally expire messages based on the delivery
status of prior messages.  For example, an application sending large messages
may wish to ensure that its messages are delivered at least at a given minimum
rate before expiring a partially-delivered message just because there is a newer
message to deliver.  That is, if the rate of data the application wishes to
write exceeds the network's throughput, the application may want to ensure that
at least some messages are delivered in their entirety.  To support this use
case, it is recommended that a QUIC library API provides a way for the sender to
monitor the smallest unacknowledged stream offset greater than the current
minimum retransmittable offset.


Receiver Interface and Behavior   {#receiver-interface}
-------------------------------

The receiver SHOULD assume that none of the data before the new current receive
offset ({{offsets}}) will be retransmitted.  A receiver MAY discard any stream
data received for an offset smaller than the new current receive offset.

It is recommended that a QUIC library API provides a way for a receiver
application to obtain the length of a gap corresponding to the expired data in
addition to data octets that follow the gap.

It is recommended that a QUIC library API provide a way for a receiver
application to skip data octets past the current point in the stream.  Such a
request from the application should be treated by QUIC as a receipt of an
EXPIRED_STREAM_DATA frame ({{frame-expired-stream-data}}) with the Minimum
Stream Offset field set of the offset to which the application wished to skip.
If the current receive offset is advanced as a result of this application
request, QUIC should transmit a MIN_STREAM_DATA frame.


Retransmission      {#retransmission}
==============

Both MIN_STREAM_DATA ({{frame-min-stream-data}}) and EXPIRED_STREAM_DATA
({{frame-expired-stream-data}}) frames MUST be retransmitted if declared lost.

Retransmission of MIN_STREAM_DATA    {#retransmission-min-stream-data}
---------------------------------

The most recent MIN_STREAM_DATA frame MUST be retransmitted until the receiver
is certain that the sender is not going to transmit any skipped data.  I.e. the
frame MUST be retransmitted until the stream enters "half-closed (remote)"
state, or all data between the largest Minimum Stream Offset field in an
acknowledged MIN_STREAM_DATA or received EXPIRED_STREAM_DATA frames and the
current receive offset ({{offsets}}) has been received.


Retransmission of EXPIRED_STREAM_DATA    {#retransmission-expired-stream-data}
---------------------------------

The most recent EXPIRED_STREAM_DATA frame for a stream MUST be retransmitted
until the sender is certain that the receiver is not expecting retransmission of
any expired data.  I.e. the frame MUST be retransmitted until the stream enters
"half-closed (local)" state, or all data between the largest Minimum Stream
Offset field in an acknowledged EXPIRED_STREAM_DATA or received MIN_STREAM_DATA
frames and the current minimum retransmittable offset ({{offsets}}) has been
acknowledged.


IANA Considerations   {#iana}
===================

This document has no actions for IANA.


Security Considerations   {#security}
=======================

This document has no new security considerations.


Change Log
==========

Since version 00
----------------

- Fixed flow control to disallow other streams to use connection credits
  designated for skipping expired bytes.


Since version 01
----------------

- Added an ability by the receiver as well as the sender to control partial
  reliability of QUIC streams.  ({{receiver-interface}})

- Added Exempt Stream Bytes value and updated connection flow control
  calculation to use Exempt Stream Bytes value.  ({{exempt-stream-bytes}})

- Replaced the Min Stream Offset value with the existing values: "min
  retransmittable offset" (for sender) and "current receive offset" (for
  receiver).  ({{offsets}})

- Changed MIN_STREAM_DATA frame to be a receiver-transmitted frame.
  ({{frame-min-stream-data}})

- Addded sender-transmitted EXPIRED_STREAM_DATA frame.
  ({{frame-expired-stream-data}})


Acknowledgments
===============

Many thanks to Mike Bishop and Ian Swett for their feedback on flow control
issues.  Thus draft could not happen without Subodh Iyengar's ideas for
receiver-controlled MIN_STREAM_DATA.  Kudos to the QUIC working group for a
mountain of feedback on this draft and for diligently plowing through hard
problems, making thousands of big and small decisions, to make the Internet
better for everyone.


--- back
