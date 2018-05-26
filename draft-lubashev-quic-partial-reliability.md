---
title: Partially Reliable Message Streams for QUIC
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

This memo introduces a new EXPIRED_STREAM_DATA frame to enable partial
reliability for QUIC streams.  The EXPIRED_STREAM_DATA frame allows a sender to
give up on retransmitting older parts of a stream and to notify the receiver
about this decision.  The content of this draft is intended for merging into
QUIC transport, recovery, and applicability drafts as a negotiable extension
and/or QUIC Version 2 transport feature.


--- middle

Notational Conventions    {#conventions}
======================

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.


Introduction     {#introduction}
============

Some applications, especially applications with near real-time requirements,
need transport that supports partially reliable streams -- streams that deliver
bytes in order but allow for applicaiton-controlled gaps.  These applications
communicate using application-specific messages that are serialized over QUIC
streams.  Applications desire partially reliable streams when their messages
expire and lose their usefulness due to later events (time passing, newer
messages, etc).

Examples of applications that can benefit from partially reliable streams are
real time video (all prior data is to be expired when a new key frame is
available) and data replication (expire previous updates, when a new update
overwrites the data).

The content of this draft is intended for {{!I-D.ietf-quic-transport}},
{{!I-D.ietf-quic-recovery}} and, {{!I-D.ietf-quic-applicability}} as a QUIC
extension and/or QUIC Version 2.

Stream-per-Message Alternative
------------------------------

It is possible to avoid the need for partially reliable streams by encoding one
message per QUIC stream.  When a message expires, the sender can reset the
stream, causing RST_STREAM frame to be transmitted, unless all data in the
stream has already been fully acknowledged.  Likewise, the receiver can send
STOP_SENDING frame to indicate its disinterest in the message.  The problem with
this approach is that messages transmitted by the application typically belong
to a message stream, and applications may need to support multiple concurrent
message streams.  Hence, a message-per-stream approach requires each message to
contain an extra header portion to associate the message with a logical
application stream.  In case of short messages, this approach introduces a
significant overhead due to STREAM frames and message headers. It also places
the burden on the application to reorder data arriving on multiple QUIC streams.
Furthermore, splitting each application stream into multiple QUIC streams
renders QUIC's per-stream flow control ineffective and requires an application
to build its own.


Partially Reliable Message Streams
----------------------------------

The proposed single-stream mechanism keeps aplication messages arriving in order
on a single stream, while allowing the application to control message
expiration.

The key to partially reliabile message streams is notifying the receiver about
data that will not be retransmitted and ensuring that the receiver can identify
the beginning of each new message.

It is important to note that the proposed protocol does not guarantee that data
is read by the receiver application at the stream offsets written to by the
sender application.


Minimum retransmittable offset and current receive offset    {#offsets}
---------------------------------------------------------

For fully reliable streams, the smallest unacknowledged data offset is treated
by the sender to be the minimum retransmittable offset.  Likewise, the current
receive offset for a stream is the smallest data offset that has not been
received by the receiver.  Due to loss and reordering, the current receive
offset may be smaller than the largest received offset.

Partially reliable streams allow the sender to advance its minimum
retransmittable offset and notify the receiver to advance its current receive
offset.


EXPIRED_STREAM_DATA Frame     {#frame-expired-stream-data}
=========================

The EXPIRED_STREAM_DATA frame (type=0x??) is used by a sender to inform a
receiver of the minimum retransmittable offset ({{offsets}}) for a stream.

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
TODO: Reconcile this with sender accounting !!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

Sending EXPIRED_STREAM_DATA frame may advance the stream's current send offset,
which MUST satisfy stream and connection flow control.

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

If the current receive offset becomes larger than the largest received offset
for the stream, the receiver MUST advance the stream's largest received offset
to be the current receive offset.

The sender MUST NOT reduce the minimum retransmittable offset for a stream, but
loss and reordering can cause EXPIRED_STREAM_DATA frames to be received out of
order.  EXPIRED_STREAM_DATA frames that do not advance the current receive
offset for the stream MUST be ignored.

An endpoint that receives an EXPIRED_STREAM_DATA frame for a send-only stream
MUST terminate the connection with error PROTOCOL_VIOLATION.


Sender Interface and Behavior    {#sender-interface}
=============================

QUIC library interface needs provide a way for a sender to expire data
previously written to the transport by updating the minimum retransmittable
offset ({{offsets}}) for a stream.  A typical sender would call this API
function whenever data previously enqueued for transmission expires, per
application semantics.  The sender would keep track of the message boundaries
and request expiration of data on a message boundary.

When an application instructs its QUIC transport to advance the minimum
retransmittable offset for a stream, and there is any unacknowledged data
(including unsent data) at an offset smaller than the new minimum
retransmittable offset, the sender SHOULD transmit an EXPIRED_STREAM_DATA frame
({{frame-expired-stream-data}}).

* When the new minimum retransmittable offset is less than or equal to the
  current send offset, the Minimum Stream Offset field in the
  EXPIRED_STREAM_DATA frame is set to the new minimum retransmittable offset.

* When the new minimum retransmittable offset is larger the current send offset,
  the Minimum Stream Offset field in the EXPIRED_STREAM_DATA frame is set to the
  current send offset plus 1, and stream data starting at the new minimum
  retransmittable offset is henceforth sent starting at the current send offset
  plus 1 (which becomes the new minimum retransmittable offset).

  It is recommended to not advance the current send offset until a STREAM frame
  is sent and to avoid sending identical EXPIRED_STREAM_DATA frames except due
  to retransmission.

  Note that an EXPIRED_STREAM_DATA frame is sent evern if the very first message
  has expired before any of its octets have been transmitted.  This allows the
  receiver to always ascertain whether there are any gaps in messages it is
  receiving.


Translating Application Offsets to QUIC Offsets
-----------------------------------------------

To allow a sender to expire a large amount of stream data never sent to the
receiver, the sender needs to create a gap between the data previously sent on
the stream and the data to be sent after the expiration point.  A single octet
gap is used for this purpose.  Sender's EXPIRED_STREAM_DATA frame extends the
minimum stream offset past that gap.  The gap ensures that the receiver does not
deliver subsequent octets to the application until the receipt of the
EXPIRED_STREAM_DATA frame, in case packets containing the EXPIRED_STREAM_DATA
frame and subsequent STREAM frame are lost or reordered.  Upon receipt of the
EXPIRED_STREAM_DATA frame, the receiver will be able to notify the application
of a gap, which will allow the application to identify the beginning of a new
message.

For example, an application wrote four 10-octet messages (A, B, C, D) to the
transport, and the current send offset (the next offset to be sent) is 12.

~~~
 0                   1   s               2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|A A A A A A A A A A B B B B B B B B B B C C C C C C C C C C D D ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Then, the application would like to expire the first two messages and requests
minimum retransmittable offset to be 20.  The transport will then send an
EXPIRED_STREAM_DATA frame with Minimum Stream Offset field set to 13 and a
subsequent STREAM frame will send the C message starting at stream offset 13.

~~~
 0                   1   s m             2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|A A A A A A A A A A B B   C C C C C C C C C C D D D D D D D D D D
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

However, if before any subsequent STREAM frames could be sent, the application
write another messages to the transport and requested to expire octets
corresponding to message C, no new EXPIRED_STREAM_DATA frame is sent, and a
subsequent STREAM frame will send the D message starting at stream offset 13.

~~~
 0                   1   s m             2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|A A A A A A A A A A B B   D D D D D D D D D D
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~


Since the QUIC library and the application need to communicate data offsets (for
example, for the purpose of updating the minimum retransmittable stream offset),
the QUIC library needs to translate appliction offsets to QUIC offsets.
Depending on the richness of the APIs exposed to the application, a single
difference between the current application and QUIC offsets is likely to be
sufficient.


Receiver Interface and Behavior   {#receiver-interface}
===============================

Upon receipt of an EXPIRED_STREAM_DATA frame ({{frame-expired-stream-data}}),
the receiver SHOULD assume that none of the data before the new current receive
offset ({{offsets}}) will be retransmitted.  A receiver SHOULD discard any
stream data received for an offset smaller than the new current receive offset.
Discarding such data ensures that when the application observes a gap in the
data stream, what follows the gap is a beginning of a new message.

It is recommended that a QUIC library API provides a way for a receiver
application to learn of the presence of a gap in the data stream, indicating
that the data that follows the gap is a beginning of a new message.


Retransmission of EXPIRED_STREAM_DATA      {#retransmission}
=====================================

The most recent EXPIRED_STREAM_DATA frame ({{frame-expired-stream-data}}) for a
stream MUST be retransmitted if it is declared lost, until the sender is certain
that the receiver is not expecting retransmission of any expired data.  I.e. the
frame MUST be retransmitted until the stream enters "half-closed (local)" state,
or all data between the largest Minimum Stream Offset field in an acknowledged
EXPIRED_STREAM_DATA frame and the current minimum retransmittable offset
({{offsets}}) has been acknowledged.


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
  reliability of QUIC streams.

- Added Exempt Stream Bytes value and updated connection flow control
  calculation to use Exempt Stream Bytes value.

- Replaced the Min Stream Offset value with the existing values: "min
  retransmittable offset" (for sender) and "current receive offset" (for
  receiver).  ({{offsets}})

- Changed MIN_STREAM_DATA frame to be a receiver-transmitted frame.

- Added sender-transmitted EXPIRED_STREAM_DATA frame.
  ({{frame-expired-stream-data}})

Since version 01
----------------

- Significantly simplifed the proposal by treating the stream as a message
  stream, allowing for data offsets not to be preserved between the sender and
  the receiver.

- Reverted to sender-only transport-level control of message expiration.

- Removed the need for Exempt Stream Bytes and changes to connection flow
  control accounting.

- Removed MIN_STREAM_DATA frame.


Acknowledgments
===============

Many thanks to Mike Bishop, Ian Swett, and Subodh Iyengar for their reviews,
feedback, and ideas.  Thus draft would not happen without their input.  Kudos to
the QUIC working group for a mountain of feedback on this draft and for
diligently plowing through hard problems, making thousands of big and small
decisions, to make the Internet better for everyone.


--- back
