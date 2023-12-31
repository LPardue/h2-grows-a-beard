---
title: "HTTP/2 Plus Plus"
abbrev: "H2++"
category: info

docname: draft-pardue-httpbis-h2-plus-plus-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword:
 - next generation
 - riker
 - beard
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "LPardue/h2-grows-a-beard"
  latest: "https://LPardue.github.io/h2-grows-a-beard/draft-pardue-httpbis-h2-plus-plus.html"

author:
 -
    fullname: "Lucas Pardue"
    organization: Cloudflare
    email: "lucapardue.24.7@gmail.com"

normative:
  RFC9110:
    display: HTTP
  RFC9112:
    display: HTTP/1.1
  RFC9113:
    display: HTTP/2

informative:



--- abstract

This document defines HTTP/2 Plus Plus, an HTTP mapping to TCP that is inspired
by HTTP/2. It shares many concepts but has a slightly different feature set.
Differences make the wire image incompatible but translation to HTTP/2 is
possible.


--- middle

# Introduction

HTTP/2 {{RFC9113}} is an optimzed expression of the semantics of HTTP {{RFC9110}} for TCP {{?TCP=RFC793}}.

Several features defined in HTTP/2 have not been widely deployed. The second
revision of {{RFC9113}} deprecated these features in a way that would avoid
interoperabilty issues. However, some of the complexity for dealing with this,
especially the possibility of speaking with a peer that tries to use deprecated
features, is pushed to implementations. Furthermore, HTTP/2 extensions that
replace features (whether deprecated or not) require careful writing to cover
all scenarios of usage.

HTTP/2 Plus Plus takes the best bits of HTTP/2 and addresses issues that are
hard to fix in a backwards-compitible and easily-deployable way. It is not wire
compatible with HTTP/2 and so uses a different APLN ID. However, the concepts
shared between HTTP/2 and HTTP/2 Plus Plus mean that it is possible to translate
between versions.

# HTTP/2 Plus Plus Protocol Overview

HTTP/2 Plus Plus provides an optimized transport for HTTP semantics. HTTP/2
HTTP/2 Plus Plus supports all of the core features of HTTP but aims to be more
efficient than HTTP/1.1.

HTTP/2 Plus Plus is a connection-oriented application-layer protocol that
runs over a TCP connection ({{TCP}}). The client is the TCP connection
initiator.

The basic protocol unit in HTTP/2 Plus Plus is a frame (section x.x). Each frame
type serves a different purpose. For example, HEADERS and DATA frames form the
basis of HTTP requests and responses ({{field-validity}}); other frame types
like SETTINGS and WINDOW_UPDATE are used in support of other HTTP/2 Plus Plus
features.

Multiplexing of requests is achieved by having each HTTP request/response
exchange associated with its own stream ({{streams-and-multiplexing}}). Streams
are largely independent of each other, so a blocked or stalled request or
response does not prevent progress on other streams.

Effective use of multiplexing depends on flow control and prioritization. Flow
control ({{flow-control}}) ensures that it is possible to efficiently use
multiplexed streams by restricting data that is transmitted to what the receiver
is able to handle. Prioritization ({{prioritization}}) ensures that limited
resources are used most effectively.

Because HTTP fields used in a connection can contain large amounts of redundant
data, frames that contain them are compressed ({{field-compression}}). This has
especially advantageous impact upon request sizes in the common case, allowing
many requests to be compressed into one packet.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

This specification describes binary formats using the conventions described in
Section 1.3 of RFC 9000 {{!QUIC=RFC9000}}. Note that this format uses network
byte order and that high-valued bits are listed before low-valued bits.

The following terms are used:

client:

: The endpoint that initiates an HTTP/2 Plus Plus connection. Clients send HTTP
requests and receive HTTP responses.

connection:

: A transport-layer connection between two endpoints.

connection error:

: An error that affects the entire HTTP/2 Plus Plus connection.

endpoint:
: Either the client or server of the connection.

frame:
: The smallest unit of communication within an HTTP/2 Plus Plus connection, consisting of a
header and a variable-length sequence of bytes structured according to the frame
type.

peer:
: An endpoint. When discussing a particular endpoint, "peer" refers to the
endpoint that is remote to the primary subject of discussion.

receiver:
: An endpoint that is receiving frames.

sender:
: An endpoint that is transmitting frames.

server:
: The endpoint that accepts an HTTP/2 Plus Plus connection. Servers receive HTTP requests
and send HTTP responses.

stream:
: A bidirectional flow of frames within the HTTP/2 Plus Plus connection.

stream error:
: An error on the individual HTTP/2 Plus Plus stream.

Finally, the terms "gateway", "intermediary", "proxy", and "tunnel" are defined
in Section 3.7 of {{RFC9110}}. Intermediaries act as both client and server at
different times.

The term "content" as it applies to message bodies is defined in {{Section 6.4
of RFC9110}}.

# Starting HTTP/2 Plus Plus

Implementations that generate HTTP requests need to discover whether a server
supports HTTP/2 Plus Plus.

HTTP/2 Plus Plus uses only the "https" URI scheme defined in {{Section 4.2 of
RFC9110}}, with the same default port numbers as HTTP/1.1 {{RFC9112}}. The URI
does not include any indication about what HTTP versions an upstream server (the
immediate peer to which the client wishes to establish a connection) supports.

A client that makes a request uses TLS {{!TLS=RFC8446}} with the ALPN extension
{{!TLS-ALPN=RFC7301}}.

HTTP/2 Plus Plus uses the "h2-TBD" (octect sequence TBD) protocol identifier.

Once TLS negotiation is complete, both the client and the server MUST send a
SETTINGS frame as their initial frame.

To avoid unnecessary latency, clients are permitted to send additional frames to
the server immediately after sending their SETTINGS frame, without waiting to
receive the server SETTINGS frame. It is important to note, however, that the
server SETTINGS frame might include settings that necessarily alter how a client
is expected to communicate with the server. Upon receiving the SETTINGS frame,
the client is expected to honor any settings established. In some
configurations, it is possible for the server to transmit SETTINGS before the
client sends additional frames, providing an opportunity to avoid this issue.

# HTTP/2 Plus Plus frames

Once the HTTP/2 Plus Plus connection is established, endpoints can begin
exchanging frames.

## Frame Layout

All frames have the following format:

~~~
HTTP/2 Plus Plus Frame Format {
  Type (i),
  Length (i),
  Stream Identifier (i),
  Frame Payload (..),
}
~~~
{: #h2-pp-frame-format title="HTTP/2 Plus Plus Frame Format"}

A frame includes the following fields:

  Type:
  : A variable-length integer that identifies the frame type.

  Length:
  : A variable-length integer that describes the length in bytes of
    the Frame Payload.

  Stream Identifier:
  : A variable-length integer that identifies the stream to which the frame
    applies. The value 0x00 is reserved for frames that are associated with
    the connection as a whole.

  Frame Payload:
  : A payload, the semantics of which are determined by the Type field.

Sending large frames can result in delays in sending time-sensitive frames (such
as RESET_STREAM or WINDOW_UPDATE), which, if blocked by the transmission of a
large frame, could affect performance.

## Frame Definitions

### DATA

DATA frames (type=0x00 or 0x01) convey arbitrary, variable-length sequences of
bytes associated with HTTP request or response content.

~~~
DATA Frame {
  Type (i) = 0x00..0x01,
  Length (i),
  Stream Identifier (i),
  Data (..),
}
~~~
{: title="DATA Frame"}

DATA frames of type 0x01 causes the stream to enter one of the "half-closed"
states or the "closed" state ({{stream-states}}).

DATA frames MUST be associated with a stream. If a DATA frame is received whose
Stream Identifier field is 0x00, the recipient MUST respond with a connection
error ({{connection-error-handling}}) of type PROTOCOL_ERROR.

DATA frames are subject to flow control and can only be sent when a stream is in
the "open" or "half-closed (remote)" state. The entire DATA frame payload is
included in flow control. If a DATA frame is received whose stream is not in the
"open" or "half-closed (local)" state, the recipient MUST respond with a stream
error ({{stream-error-handling}}) of type STREAM_CLOSED.

### HEADERS

The HEADERS frame (type=0x02 or 0x03) is used to carry an HTTP field
section that is encoded using HPACK. Despite the name, a HEADERS frame can carry
a header section or a trailer section.

~~~
HEADERS Frame {
  Type (i) = 0x02..0x03,
  Length (i),
  Stream Identifier (i),
  Encoded Field Section (..),
}
~~~
{: title="HEADERS Frame"}

HEADERS frames of type 0x02 open a stream ({{stream-states}}).

HEADERS frames of type 0x03 cause the stream to enter one of the "half-closed"
states or the "closed" state ({{stream-states}}).

HEADERS frames can be sent on a stream in the "idle", "open", or "half-closed
(remote)" state.

HEADERS frames MUST be associated with a stream. If a HEADERS frame is received
whose Stream Identifier field is 0x00, the recipient MUST respond with a
connection error ({{connection-error-handling}}) of type PROTOCOL_ERROR.

The HEADERS frame changes the connection state as described in
{{field-compression}}.

### RESET_STREAM {#reset-stream}

The RESET_STREAM frame (type=0x04) allows for immediate termination of a stream.
RESET_STREAM is sent to request cancellation of a stream or to indicate that an
error condition has occurred.

~~~
RESET_STREAM Frame {
  Type (i) = 0x04,
  Length (i),
  Stream Identifier (i),
  Error Code (i),
}
~~~
{: title="RESET_STREAM Frame"}

The RESET_STREAM frame contains a variable-length integer that conveys and error
code to indicate why the stream is being terminated.

The RESET_STREAM frame fully terminates the referenced stream and causes it to
enter the "closed" state. After receiving a RESET_STREAM on a stream, the
receiver MUST NOT send additional frames for that stream. However, after sending
the RESET_STREAM, the sending endpoint MUST be prepared to receive and process
additional frames sent on the stream that might have been sent by the peer prior
to the arrival of the RESET_STREAM.

RESET_STREAM frames MUST be associated with a stream. If a RESET_STREAM frame is
received with a stream identifier of 0x00, the recipient MUST treat this as a
connection error ({{connection-error-handling}}) of type PROTOCOL_ERROR.

RESET_STREAM frames MUST NOT be sent for a stream in the "idle" state. If a
RESET_STREAM frame identifying an idle stream is received, the recipient MUST
treat this as a connection error ({{connection-error-handling}}) of type
PROTOCOL_ERROR.


### SETTINGS

The SETTINGS frame (type=0x05) conveys configuration parameters that affect how
endpoints communicate, such as preferences and constraints on peer behavior. The
SETTINGS frame is also used to acknowledge the receipt of those settings.
Individually, a configuration parameter from a SETTINGS frame is referred to as
a "setting".

Settings are not negotiated; they describe characteristics of the sending peer,
which are used by the receiving peer. Different values for the same setting can
be advertised by each peer. For example, a client might set a high initial
flow-control window, whereas a server might set a lower value to conserve
resources.

A SETTINGS frame MUST be sent by both endpoints at the start of a connection and
MUST NOT be sent subsequently. If an endpoint receives a second SETTINGS frame,
the endpoint MUST respond with a connection error of PROTOCOL_ERROR.

SETTINGS frames always apply to a connection, never a single stream. The stream
identifier for a SETTINGS frame MUST be zero (0x00). If an endpoint receives a
SETTINGS frame whose Stream Identifier field is anything other than 0x00, the
endpoint MUST respond with a connection error ({{connection-error-handling}}) of
type PROTOCOL_ERROR.

The SETTINGS frame affects connection state. A badly formed or incomplete
SETTINGS frame MUST be treated as a connection error
({{connection-error-handling}}) of type PROTOCOL_ERROR.

~~~
Setting {
  Identifier (i),
  Value (i),
}

SETTINGS Frame {
  Type (i) = 0x05,
  Length (i),
  Stream Identifier (i),
  Setting (..) ...,
}
~~~
{: title="SETTINGS Frame"}

An implementation MUST ignore any parameter with an identifier it does not
understand.

#### Defined SETTINGS Parameters

SETTINGS_HEADER_TABLE_SIZE (0x01):
  : This setting allows the sender to inform the remote endpoint of the maximum
    size of the compression table used to decode field blocks, in units of
    bytes. The encoder can select any size equal to or less than this value by
    using signaling specific to the compression format inside a field block
    (see {{!COMPRESSION=RFC7541}}). The initial value is 4,096 bytes.

SETTINGS_INITIAL_WINDOW_SIZE (0x04):
  : This setting indicates the sender's initial window size (in units of bytes)
    for stream-level flow control. The initial value is 216-1 (65,535) bytes.

This setting affects the window size of all streams (see Section TODO).

SETTINGS_MAX_FIELD_SECTION_SIZE (0x06):
  : This advisory setting informs a peer of the maximum field section size that
    the sender is prepared to accept, in units of bytes. See
    {{header-size-constraints}} for usage.

  : For any given request, a lower limit than what is advertised MAY be
  enforced. The initial value of this setting is unlimited.

An endpoint that receives a SETTINGS frame with any unknown or unsupported
identifier MUST ignore that setting.

### PING

The PING frame (type=0x06 or 0x07) is a mechanism for measuring a minimal
round-trip time from the sender, as well as determining whether an idle
connection is still functional. PING frames can be sent from any endpoint.

~~~
PING Frame {
  Type (i) = 0x06..0x07,
  Length (i),
  Stream Identifier (i),
  Opaque Data (64),
}
~~~
{: title="PING Frame"}

PING frames MUST contain 8 bytes of opaque data in the frame payload. A sender
can include any value it chooses and use those bytes in any fashion.

Receivers of a PING frame of type 0x06 MUST send a PING frame with the type 0x07
in response, with an identical frame payload. PING responses SHOULD be given
higher priority than any other frame.

PING frames are not associated with any individual stream. If a PING frame is
received with a Stream Identifier field value other than 0x00, the recipient
MUST respond with a connection error ({{connection-error-handling}}) of type
PROTOCOL_ERROR.

Receipt of a PING frame with a length field value other than 8 MUST be treated
as a connection error ({{connection-error-handling}}) of type FRAME_SIZE_ERROR.

### GOAWAY

The GOAWAY frame (type=0x07) is used to initiate shutdown of a connection or to
signal serious error conditions. GOAWAY allows an endpoint to gracefully stop
accepting new streams while still finishing processing of previously established
streams. This enables administrative actions, like server maintenance.

There is an inherent race condition between an endpoint starting new streams and
the remote peer sending a GOAWAY frame. To deal with this case, the GOAWAY
contains the stream identifier of the last peer-initiated stream that was or
might be processed on the sending endpoint in this connection. For instance, if
the server sends a GOAWAY frame, the identified stream is the highest-numbered
stream initiated by the client.

Once the GOAWAY is sent, the sender will ignore frames sent on streams initiated
by the receiver if the stream has an identifier higher than the included last
stream identifier. Receivers of a GOAWAY frame MUST NOT open additional streams
on the connection, although a new connection can be established for new streams.

If the receiver of the GOAWAY has sent data on streams with a higher stream
identifier than what is indicated in the GOAWAY frame, those streams are not or
will not be processed. The receiver of the GOAWAY frame can treat the streams as
though they had never been created at all, thereby allowing those streams to be
retried later on a new connection.

Endpoints SHOULD always send a GOAWAY frame before closing a connection so that
the remote peer can know whether a stream has been partially processed or not.
For example, if an HTTP client sends a POST at the same time that a server
closes a connection, the client cannot know if the server started to process
that POST request if the server does not send a GOAWAY frame to indicate what
streams it might have acted on.

An endpoint might choose to close a connection without sending a GOAWAY for
misbehaving peers.

A GOAWAY frame might not immediately precede closing of the connection; a
receiver of a GOAWAY that has no more use for the connection SHOULD still send a
GOAWAY frame before terminating the connection.

~~~
GOAWAY Frame {
  Type (i) = 0x08,
  Length (i),
  Stream Identifier (i),
  Last-Stream-ID (i),
  Error Code (i),
  Additional Debug Data (..),
}
~~~
{: title="GOAWAY Frame"}

The GOAWAY frame applies to the connection, not a specific stream. An endpoint
MUST treat a GOAWAY frame with a stream identifier other than 0x00 as a
connection error ({{connection-error-handling}}) of type PROTOCOL_ERROR.

The last stream identifier in the GOAWAY frame contains the highest-numbered
stream identifier for which the sender of the GOAWAY frame might have taken some
action on or might yet take action on. All streams up to and including the
identified stream might have been processed in some way. The last stream
identifier can be set to 0 if no streams were processed.

On streams with lower- or equal-numbered identifiers that were not closed
completely prior to the connection being closed, reattempting requests,
transactions, or any protocol activity is not possible, except for idempotent
actions like HTTP GET, PUT, or DELETE. Any protocol activity that uses
higher-numbered streams can be safely retried using a new connection.

Activity on streams numbered lower than or equal to the last stream identifier
might still complete successfully. The sender of a GOAWAY frame might gracefully
shut down a connection by sending a GOAWAY frame, maintaining the connection in
an "open" state until all in-progress streams complete.

An endpoint MAY send multiple GOAWAY frames if circumstances change. For
instance, an endpoint that sends GOAWAY with NO_ERROR during graceful shutdown
could subsequently encounter a condition that requires immediate termination of
the connection. The last stream identifier from the last GOAWAY frame received
indicates which streams could have been acted upon. Endpoints MUST NOT increase
the value they send in the last stream identifier, since the peers might already
have retried unprocessed requests on another connection.

A client that is unable to retry requests loses all requests that are in flight
when the server closes the connection. This is especially true for
intermediaries that might not be serving clients using HTTP/2 Plus Plus. A
server that is attempting to gracefully shut down a connection SHOULD send an
initial GOAWAY frame with the last stream identifier set to $BIG_NUMBER and a
NO_ERROR code. This signals to the client that a shutdown is imminent and that
initiating further requests is prohibited. After allowing time for any in-flight
stream creation (at least one round-trip time), the server MAY send another
GOAWAY frame with an updated last stream identifier. This ensures that a
connection can be cleanly shut down without losing requests.

After sending a GOAWAY frame, the sender can discard frames for streams
initiated by the receiver with identifiers higher than the identified last
stream. However, any frames that alter connection state cannot be completely
ignored. For instance, HEADERS, PUSH_PROMISE, and CONTINUATION frames MUST be
minimally processed to ensure that the state maintained for field section
compression is consistent (see {{field-compression}}); similarly, DATA frames
MUST be counted toward the connection flow-control window. Failure to process
these frames can cause flow control or field section compression state to become
unsynchronized.

The GOAWAY frame also contains an error code ({{error-codes}}) that contains the
reason for closing the connection.

Endpoints MAY append opaque data to the frame payload of any GOAWAY frame.
Additional debug data is intended for diagnostic purposes only and carries no
semantic value. Debug information could contain security- or privacy-sensitive
data. Logged or otherwise persistently stored debug data MUST have adequate
safeguards to prevent unauthorized access.

### WINDOW_UPDATE {#window-update}

~~~
WINDOW_UPDATE Frame {
  Type (i) = 0x09,
  Length (i),
  Stream Identifier (i),
  Window Size Increment (i),
}
~~~
{: title="WINDOW_UPDATE Frame"}

The WINDOW_UPDATE frame can be specific to a stream or to the entire connection.
In the former case, the frame's stream identifier indicates the affected stream;
in the latter, the value "0" indicates that the entire connection is the subject
of the frame.

A receiver MUST treat the receipt of a WINDOW_UPDATE frame with a flow-control
window increment of 0 as a stream error ({{stream-error-handling}}) of type
PROTOCOL_ERROR; errors on the connection flow-control window MUST be treated as
a connection error ({{connection-error-handling}}).

WINDOW_UPDATE can be sent by a peer that has initiated a stream closure with
appropriate HEADERS or DATA frame types. This means that a receiver could
receive a WINDOW_UPDATE frame on a stream in a "half-closed (remote)" or
"closed" state. A receiver MUST NOT treat this as an error (see
{{stream-states}}).

A receiver that receives a flow-controlled frame MUST always account for its
contribution against the connection flow-control window, unless the receiver
treats this as a connection error ({{connection-error-handling}}). This is
necessary even if the frame is in error. The sender counts the frame toward the
flow-control window, but if the receiver does not, the flow-control window at
the sender and receiver can become different.

TODO: import almost all the H2 flow control guidance

### PADDING

TODO PADDING.

Is it needed and can it be moved into a separate frame safely?

### MAX_STREAMS

TODO MAX_STREAMS

Import from QUIC

## Field Compression

HTTP/2 Plus Plus uses HPACK {{COMPRESSION}} to compress header and trailer
sections, including the control data present in the header section.

TODO import just enough from RFC 9113 for things to make sense.

To allow for better compression efficiency, the Cookie header field
({{?COOKIES=RFC6265}}) MAY be split into separate field lines, each with one or
more cookie-pairs, before compression. If a decompressed field section contains
multiple cookie field lines, these MUST be concatenated into a single byte
string using the two-byte delimiter of "; " (ASCII 0x3b, 0x20) before being
passed into a context other than HTTP/2, HTTP/2 Plus Plus or HTTP/3, such as an
HTTP/1.1 connection, or a generic HTTP server application.

## Header Size Constraints

An HTTP/2 Plus Plus implementation MAY impose a limit on the maximum size of the
message header it will accept on an individual HTTP message. A server that
receives a larger header section than it is willing to handle can send an HTTP
431 (Request Header Fields Too Large) status code ({{!RFC6585}}). A client can
discard responses that it cannot process. The size of a field list is calculated
based on the uncompressed size of fields, including the length of the name and
value in bytes plus an overhead of 32 bytes for each field.

If an implementation wishes to advise its peer of this limit, it can be conveyed
as a number of bytes in the SETTINGS_MAX_FIELD_SECTION_SIZE parameter. An
implementation that has received this parameter SHOULD NOT send an HTTP message
header that exceeds the indicated size, as the peer will likely refuse to
process it. However, an HTTP message can traverse one or more intermediaries
before reaching the origin server; see {{Section 3.7 of RFC9110}}. Because this
limit is applied separately by each implementation that processes the message,
messages below this limit are not guaranteed to be accepted.

## Compression State

TODO consider if/how allowing only 1 SETTINGS and not providing a SETTINGS ACK
affects the text in RFC 9113.

# Streams and Multiplexing

A "stream" is an independent, bidirectional sequence of frames exchanged between
the client and server within an HTTP/2 Plus Plus connection. Streams have
several important characteristics:

* A single HTTP/2 Plus Plus connection can contain multiple concurrently open
  streams, with either endpoint interleaving frames from multiple streams.
* Streams can be established and used unilaterally or shared by either endpoint.
* Streams can be closed by either endpoint.
* The order in which frames are sent is significant. Recipients process frames
  in the order they are received. In particular, the order of HEADERS and DATA
  frames is semantically significant.
* Streams are identified by an integer. Stream identifiers are assigned to
  streams by the endpoint initiating the stream.

## Stream States

The lifecycle of a stream is shown below.

~~~
                             +--------+
                             |        |
                             |  idle  |
                             |        |
                             +--------+
                                 |
                                 | send H /
                                 | recv H
                                 |
                                 v
              recv D (0x1) / +--------+ send D (0x1) /
              recv H (0x3)   |        | send H (0x3)
           ,-----------------+  open  +------------.
           |                 |        |            |
           v                 +---+----+            v
      +----------+               |           +----------+
      |   half-  |               |           |   half-  |
      |  closed  |               | send R /  |  closed  |
      | (remote) |               | recv R    | (local)  |
      +----+-----+               |           +-----+----+
           |                     |                 |
           |  send D (0x1) /     |  recv D (0x1) / |
           |  send H (0x3) /     |  recv H (0x3) / |
           |  send R /           v        send R / |
           |  recv R         +--------+   recv R   |
           `---------------->|        |<-----------'
                             | closed |
                             |        |
                             +--------+
~~~
{: title="Stream States"}

send:
  : endpoint sends this frame

recv:
  : endpoint receives this frame

H:
  : HEADERS frame

H (0x3):
  : HEADERS frame of type 0x03

D (0x1):
  : DATA frame of type 0x01

R:
  : RESET_STREAM frame

Note that this diagram shows stream state transitions and the frames that affect
those transitions only. For the purpose of state transitions, a single HEADERS
frame of type 0x03 can cause two state transitions (idle to open, and open to
half-closed).

Both endpoints have a subjective view of the state of a stream that could be
different when frames are in transit. Endpoints do not coordinate the creation
of streams; they are created unilaterally by either endpoint. The negative
consequences of a mismatch in states are limited to the "closed" state after
sending RESET_STREAM, where frames might be received for some time after
closing.

Streams have the following states:

idle:
  : All streams start in the "idle" state.

  : The following transitions are valid from this state:

  : * Sending a HEADERS frame as a client, or receiving a HEADERS frame of
  either type as a server, causes the stream to become "open". The stream
  identifier is selected as described in {{stream-states}}.1. The same HEADERS frame
  can also cause a stream to immediately become "half-closed" if the type is
  0x03.
  * Opening a stream with a higher-valued stream identifier causes the stream to
transition immediately to a "closed" state; note that this transition is not
shown in the diagram.

  : Receiving any frame other than HEADERS on a stream in this state MUST be
treated as a connection error ({{connection-error-handling}}) of type PROTOCOL_ERROR.

half-closed (local):

  : A stream that is in the "half-closed (local)" state cannot be used for sending frames other than WINDOW_UPDATE and RESET_STREAM.

  : A stream transitions from this state to "closed" when a DATA frame of type
  0x01 or a HEADERS frame of type 0x03 is received or when either peer sends a
  RESET_STREAM frame.

  : An endpoint can receive any type of frame in this state. Providing
  flow-control credit using WINDOW_UPDATE frames is necessary to continue
  receiving flow-controlled frames. In this state, a receiver can ignore
  WINDOW_UPDATE frames, which might arrive for a short period after a either a
  DATA frame of type 0x01 or a HEADERS frame of type 0x03 is sent.

half-closed (remote):
  : A stream that is "half-closed (remote)" is no longer being used by the peer to send frames. In this state, an endpoint is no longer obligated to maintain a receiver flow-control window.

  : If an endpoint receives additional frames, other than WINDOW_UPDATE,  or
  RESET_STREAM, for a stream that is in this state, it MUST respond with a
  stream error ({{stream-error-handling}}) of type STREAM_CLOSED.

  : A stream that is "half-closed (remote)" can be used by the endpoint to send
  frames of any type. In this state, the endpoint continues to observe
  advertised stream-level flow-control limits ({{flow-control}}).

  : A stream can transition from this state to "closed" by sending a DATA frame
  of type 0x01 or a HEADERS frame of type 0x03 or when either peer sends a
  RESET_STREAM frame.

closed:

  : The "closed" state is the terminal state.

  : A stream enters the "closed" state after an endpoint both sends and receives
  a fa DATA frame of type 0x01 or a HEADERS frame of type 0x03. A stream also
  enters the "closed" state after an endpoint either sends or receives a
  RESET_STREAM frame.

  : An endpoint MUST NOT send frames on a closed stream. An endpoint MAY treat
  receipt of any frame on a closed stream as a connection error
  ({{connection-error-handling}}) of type STREAM_CLOSED, except as noted below.

  : An endpoint that sends a DATA frame of type 0x01 or a HEADERS frame of type
  0x03 or a RESET_STREAM frame might receive a WINDOW_UPDATE or RESET_STREAM
  frame from its peer in the time before the peer receives and processes the
  frame that closes the stream.

  : An endpoint that sends a RESET_STREAM frame on a stream that is in the
"open" or "half-closed (local)" state could receive any type of frame. The peer
might have sent or enqueued for sending these frames before processing the
RESET_STREAM frame. An endpoint MUST minimally process and then discard any
frames it receives in this state. This means updating header compression state
for HEADERS frames. Additionally, the content of DATA frames counts toward the
connection flow-control window.

  : An endpoint can perform this minimal processing for all streams that are in
  the "closed" state. Endpoints MAY use other signals to detect that a peer has
  received the frames that caused the stream to enter the "closed" state and
  treat receipt of any frame as a connection error
  ({{connection-error-handling}}) of type PROTOCOL_ERROR. Endpoints can use
  frames that indicate that the peer has received the closing signal to drive
  this. Endpoints SHOULD NOT use timers for this purpose. For example, PING
  frames, receiving data on streams that were created after closing the stream,
  or responses to requests created after closing the stream.

In the absence of more specific rules, implementations SHOULD treat the receipt
of a frame that is not expressly permitted in the description of a state as a
connection error ({{connection-error-handling}}) of type PROTOCOL_ERROR.

The rules in this section only apply to frames defined in this document. Receipt
of frames for which the semantics are unknown cannot be treated as an error, as
the conditions for sending and receiving those frames are also unknown; see
{{extending}}.

Streams are identified by an unsigned 31-bit integer. Streams initiated by a
client MUST use odd-numbered stream identifiers; those initiated by the server
MUST use even-numbered stream identifiers. A stream identifier of zero (0x00) is
used for connection control messages; the stream identifier of zero cannot be
used to establish a new stream.

The identifier of a newly established stream MUST be numerically greater than
all streams that the initiating endpoint has opened or reserved. This governs
streams that are opened using a HEADERS frame and streams that are reserved
using PUSH_PROMISE. An endpoint that receives an unexpected stream identifier
MUST respond with a connection error ({{connection-error-handling}}) of type
PROTOCOL_ERROR.

A HEADERS frame will transition the client-initiated stream identified by the
stream identifier in the frame header from "idle" to "open". A PUSH_PROMISE
frame will transition the server-initiated stream identified by the Promised
Stream ID field in the frame payload from "idle" to "reserved (local)" or
"reserved (remote)". When a stream transitions out of the "idle" state, all
streams in the "idle" state that might have been opened by the peer with a
lower-valued stream identifier immediately transition to "closed". That is, an
endpoint may skip a stream identifier, with the effect being that the skipped
stream is immediately closed.

### Stream Identifiers

Stream identifiers cannot be reused. Long-lived connections can result in an
endpoint exhausting the available range of stream identifiers. A client that is
unable to establish a new stream identifier can establish a new connection for
new streams. A server that is unable to establish a new stream identifier can
send a GOAWAY frame so that the client is forced to open a new connection for
new streams.

### Stream Concurrency

TODO MAX_STREAMS

Import from QUIC

## Flow Control

Using streams for multiplexing introduces contention over use of the TCP
connection, resulting in blocked streams. A flow-control scheme ensures that
streams on the same connection do not destructively interfere with each other.
Flow control is used for both individual streams and the connection as a whole.

HTTP/2 Plus Plus provides for flow control through use of the WINDOW_UPDATE
frame ({{window-update}}).

### Flow-Control Principles

HTTP/2 Plus Plus stream flow control aims to allow a variety of flow-control
algorithms to be used without requiring protocol changes. Flow control has the
following characteristics:

1. Flow control is specific to a connection. HTTP/2 Plus Plus flow control
   operates between the endpoints of a single hop and not over the entire
   end-to-end path.
2. Flow control is based on WINDOW_UPDATE frames. Receivers advertise how many
   bytes they are prepared to receive on a stream and for the entire
   connection. This is a credit-based scheme.
3. Flow control is directional with overall control provided by the receiver. A
   receiver MAY choose to set any window size that it desires for each stream
   and for the entire connection. A sender MUST respect flow-control limits
   imposed by a receiver. Clients, servers, and intermediaries all independently
   advertise their flow-control window as a receiver and abide by the
   flow-control limits set by their peer when sending.
4. The initial value for the flow-control window is 65,535 bytes for both new
   streams and the overall connection.
5. The frame type determines whether flow control applies to a frame. Of the
   frames specified in this document, only DATA frames are subject to flow
   control; all other frame types do not consume space in the advertised
   flow-control window. This ensures that important control frames are not
   blocked by flow control.
6. An endpoint can choose to disable its own flow control, but an endpoint
   cannot ignore flow-control signals from its peer.
7. HTTP/2 Plus Plus defines only the format and semantics of the WINDOW_UPDATE
   frame ({{window-update}}). This document does not stipulate how a receiver
   decides when to send this frame or the value that it sends, nor does it
   specify how a sender chooses to send packets. Implementations are able to
   select any algorithm that suits their needs.

Implementations are also responsible for prioritizing the sending of requests
and responses, choosing how to avoid head-of-line blocking for requests, and
managing the creation of new streams. Algorithm choices for these could interact
with any flow-control algorithm.

### Appropriate Use of Flow Control

Flow control is defined to protect endpoints that are operating under resource
constraints. For example, a proxy needs to share memory between many connections
and also might have a slow upstream connection and a fast downstream one. Flow
control addresses cases where the receiver is unable to process data on one
stream yet wants to continue to process other streams in the same connection.

Deployments that do not require this capability can advertise a flow-control
window of the maximum size ($BIG_NUMBER) and can maintain this window by sending
a WINDOW_UPDATE frame when any data is received. This effectively disables flow
control for that receiver. Conversely, a sender is always subject to the
flow-control window advertised by the receiver.

Deployments with constrained resources (for example, memory) can employ flow
control to limit the amount of memory a peer can consume. Note, however, that
this can lead to suboptimal use of available network resources if flow control
is enabled without knowledge of the bandwidth * delay product (see
{{?RFC7323}}).

Even with full awareness of the current bandwidth * delay product,
implementation of flow control can be difficult. Endpoints MUST read and process
HTTP/2 Plus Plus frames from the TCP receive buffer as soon as data is
available. Failure to read promptly could lead to a deadlock when critical
frames, such as WINDOW_UPDATE, are not read and acted upon. Reading frames
promptly does not expose endpoints to resource exhaustion attacks, as HTTP/2
Plus Plus flow control limits resource commitments.

### Flow-Control Performance

If an endpoint cannot ensure that its peer always has available flow-control
window space that is greater than the peer's bandwidth * delay product on this
connection, its receive throughput will be limited by HTTP/2 Plus Plus flow
control. This will result in degraded performance.

Sending timely WINDOW_UPDATE frames can improve performance. Endpoints will want
to balance the need to improve receive throughput with the need to manage
resource exhaustion risks and should take careful note of Section 10 in defining
their strategy to manage window sizes.

## Prioritization

In a multiplexed protocol like HTTP/2 Plus Plus, prioritizing allocation of
bandwidth and computation resources to streams can be critical to attaining good
performance. A poor prioritization scheme can result in HTTP/2 Plus Plus
providing poor performance. With no parallelism at the TCP layer, performance
could be significantly worse than HTTP/1.1.

It is RECOMMENDED that endpoints use the Extensible Priorities scheme
{{!PRIORITIES=RFC9218}}.

## Error Handling

HTTP/2 Plus Plus framing permits two classes of errors:

 * An error condition that renders the entire connection unusable is a
   connection error.
 * An error in an individual stream is a stream error.

A list of error codes is included in {{error-codes}}.

It is possible that an endpoint will encounter frames that would cause multiple
errors. Implementations MAY discover multiple errors during processing, but they
SHOULD report at most one stream and one connection error as a result.

The first stream error reported for a given stream prevents any other errors on
that stream from being reported. In comparison, the protocol permits multiple
GOAWAY frames, though an endpoint SHOULD report just one type of connection
error unless an error is encountered during graceful shutdown. If this occurs,
an endpoint MAY send an additional GOAWAY frame with the new error code, in
addition to any prior GOAWAY that contained NO_ERROR.

If an endpoint detects multiple different errors, it MAY choose to report any
one of those errors. If a frame causes a connection error, that error MUST be
reported. Additionally, an endpoint MAY use any applicable error code when it
detects an error condition; a generic error code (such as PROTOCOL_ERROR or
INTERNAL_ERROR) can always be used in place of more specific error codes.

### Connection Error Handling

A connection error is any error that prevents further processing of the frame
layer or corrupts any connection state.

An endpoint that encounters a connection error SHOULD first send a GOAWAY frame
({{goaway}}) with the stream identifier of the last stream that it successfully
received from its peer. The GOAWAY frame includes an error code
({{error-codes}}) that indicates why the connection is terminating. After
sending the GOAWAY frame for an error condition, the endpoint MUST close the TCP
connection.

It is possible that the GOAWAY will not be reliably received by the receiving
endpoint. In the event of a connection error, GOAWAY only provides a best-effort
attempt to communicate with the peer about why the connection is being
terminated.

An endpoint can end a connection at any time. In particular, an endpoint MAY
choose to treat a stream error as a connection error. Endpoints SHOULD send a
GOAWAY frame when ending a connection, providing that circumstances permit it.

### Stream Error Handling

A stream error is an error related to a specific stream that does not affect
processing of other streams.

An endpoint that detects a stream error sends a RESET_STREAM frame
({{reset-stream}}) that contains the stream identifier of the stream where the
error occurred. The RESET_STREAM frame includes an error code that indicates the
type of error.

A RESET_STREAM is the last frame that an endpoint can send on a stream. The peer
that sends the RESET_STREAM frame MUST be prepared to receive any frames that
were sent or enqueued for sending by the remote peer. These frames can be
ignored, except where they modify connection state (such as the state maintained
for field section compression ({{field-compression}}) or flow control).

Normally, an endpoint SHOULD NOT send more than one RESET_STREAM frame for any
stream. However, an endpoint MAY send additional RESET_STREAM frames if it
receives frames on a closed stream after more than a round-trip time. This
behavior is permitted to deal with misbehaving implementations.

To avoid looping, an endpoint MUST NOT send a RESET_STREAM in response to a
RESET_STREAM frame.

### Connection Termination

If the TCP connection is closed or reset while streams remain in the "open" or
"half-closed" states, then the affected streams cannot be automatically retried
(see {{request-reliability}} for details).

## Extending HTTP/2 Plus Plus {#extending}

TODO

# Error Codes

Error codes are used in RESET_STREAM and GOAWAY frames to convey the reasons for
the stream or connection error.

Error codes share a common code space. Some error codes apply only to either
streams or the entire connection and have no defined semantics in the other
context.

The following error codes are defined:

NO_ERROR (0x00):
  : The associated condition is not a result of an error. For example, a GOAWAY
    might include this code to indicate graceful shutdown of a connection.

PROTOCOL_ERROR (0x01):
  : The endpoint detected an unspecific protocol error. This error is for use
    when a more specific error code is not available.

INTERNAL_ERROR (0x02):
  : The endpoint encountered an unexpected internal error.

FLOW_CONTROL_ERROR (0x03):
  : The endpoint detected that its peer violated the flow-control protocol.

RESERVED (0x04):
  : Was HTTP/2 SETTINGS_TIMEOUT.

STREAM_CLOSED (0x05):
  : The endpoint received a frame after a stream was half-closed.

FRAME_SIZE_ERROR (0x06):
  : The endpoint received a frame with an invalid size.

REFUSED_STREAM (0x07):
  : The endpoint refused the stream prior to performing any application
    processing (see {{request-reliability}} for details).

CANCEL (0x08):
  : The endpoint uses this error code to indicate that the stream is no longer
    needed.

COMPRESSION_ERROR (0x09):
  : The endpoint is unable to maintain the field section compression context
    for the connection.

CONNECT_ERROR (0x0a):
  : The connection established in response to a CONNECT request
   ({{the-connect-method}}) was reset or abnormally closed.

ENHANCE_YOUR_CALM (0x0b):
  : The endpoint detected that its peer is exhibiting a behavior that might be
    generating excessive load.

INADEQUATE_SECURITY (0x0c):
  : The underlying transport has properties that do not meet minimum security
    requirements (see {{use-of-tls-features}}).

HTTP_1_1_REQUIRED (0x0d):
  : The endpoint requires that HTTP/1.1 be used instead of HTTP/2 Plus Plus.

Unknown or unsupported error codes MUST NOT trigger any special behavior. These
MAY be treated by an implementation as being equivalent to INTERNAL_ERROR.

# Expressing HTTP Semantics in HTTP/2 Plus Plus

HTTP/2 Plus Plus is an instantiation of the HTTP message abstraction ({{Section 6 of
RFC9110}}).

## HTTP Message Framing

A client sends an HTTP request on a new stream, using a previously unused stream
identifier ({{stream-states}}.1). A server sends an HTTP response on the same
stream as the request.

An HTTP message (request or response) consists of:

1. one HEADERS frame containing the header section (see {{Section 6.3 of
   RFC9110}}),
2. zero or more DATA frames containing the message content (see {{Section 6.4 of
   RFC9110}}), and
3. optionally, one HEADERS frame  containing the trailer section, if present
   (see {{Section 6.5 of RFC9110}}).

For a response only, a server MAY send any number of interim responses before
the HEADERS frame containing a final response. An interim response consists of a
HEADERS frame containing the control data and header section of an interim (1xx)
HTTP response (see Section 15 of {{RFC9110}}). A HEADERS frame of type 0x03 that
carries an informational status code is malformed ({{malformed-messages}}).

The last frame in the sequence ends the stream using either a DATA frame of type
0x01 or a HEADERS frame of type 0x03.

HTTP/2 Plus Plus uses DATA frames to carry message content. The chunked transfer
encoding defined in {{Section 7.1 of RFC9110}} cannot be used in HTTP/2 Plus
Plus; see {{connection-specific-header-fields}}.

Trailer fields are carried a HEADERS frame of type 0x03. Trailers MUST NOT
include pseudo-header fields ({{http-control-data}}). An endpoint that receives
pseudo-header fields in trailers MUST treat the request or response as malformed
({{malformed-messages}}).

An endpoint that receives a HEADERS frame of type 0x02 after receiving the
HEADERS frame that opens a request or after receiving a final
(non-informational) status code MUST treat the corresponding request or response
as malformed ({{malformed-messages}}).

An HTTP request/response exchange fully consumes a single stream. A request
starts with the HEADERS frame that puts the stream into the "open" state. The
request ends with a either a DATA frame of type 0x01 or a HEADERS frame of type
0x03, which causes the stream to become "half-closed (local)" for the client and
"half-closed (remote)" for the server. A response stream starts with zero or
more interim responses in HEADERS frames, followed by a HEADERS frame containing
a final status code.

An HTTP response is complete after the server sends -- or the client receives --
either a DATA frame of type 0x01 or a HEADERS frame of type 0x03. A server can
send a complete response prior to the client sending an entire request if the
response does not depend on any portion of the request that has not been sent
and received. When this is true, a server MAY request that the client abort
transmission of a request without error by sending a RESET_STREAM with an error
code of NO_ERROR after sending a complete response (i.e., either a DATA frame of
type 0x01 or a HEADERS frame of type 0x03). Clients MUST NOT discard responses
as a result of receiving such a RESET_STREAM, though clients can always discard
responses at their discretion for other reasons.

### Malformed Messages

A malformed request or response is one that is an otherwise valid sequence of
HTTP/2 Plus Plus frames but is invalid due to the presence of extraneous frames,
prohibited fields or pseudo-header fields, the absence of mandatory
pseudo-header fields, the inclusion of uppercase field names, or invalid field
names and/or values (in certain circumstances; see {{http-fields}}).

A request or response that includes message content can include a content-length
header field. A request or response is also malformed if the value of a
content-length header field does not equal the sum of the DATA frame payload
lengths that form the content, unless the message is defined as having no
content. For example, 204 or 304 responses contain no content, as does the
response to a HEAD request. A response that is defined to have no content, as
described in {{Section 6.4.1 of RFC9110}}, MAY have a non-zero content-length
header field, even though no content is included in DATA frames.

Intermediaries that process HTTP requests or responses (i.e., any intermediary
not acting as a tunnel) MUST NOT forward a malformed request or response.
Malformed requests or responses that are detected MUST be treated as a stream
error ({{stream-error-handling}}) of type PROTOCOL_ERROR.

For malformed requests, a server MAY send an HTTP response prior to closing or
resetting the stream. Clients MUST NOT accept a malformed response.

Endpoints that progressively process messages might have performed some
processing before identifying a request or response as malformed. For instance,
it might be possible to generate an informational or 404 status code without
having received a complete request. Similarly, intermediaries might forward
incomplete messages before detecting errors. A server MAY generate a final
response before receiving an entire request when the response does not depend on
the remainder of the request being correct.

These requirements are intended to protect against several types of common
attacks against HTTP; they are deliberately strict because being permissive can
expose implementations to these vulnerabilities.

## HTTP Fields

HTTP fields {{Section 5 of RFC9110}} are conveyed by HTTP/2 Plus Plus in HEADERS
frames, compresses with HPACK {{COMPRESSION}}.

Field names MUST be converted to lowercase when constructing an HTTP/2 Plus Plus
message.

### Field Validity

The definitions of field names and values in HTTP prohibit some characters that
HPACK might be able to convey. HTTP/2 Plus Plus implementations SHOULD validate
field names and values according to their definitions in {{Sections 5.1 and 5.5
of RFC9110}}, respectively, and treat messages that contain prohibited
characters as malformed ({{malformed-messages}}).

Failure to validate fields can be exploited for request smuggling attacks. In
particular, unvalidated fields might enable attacks when messages are forwarded
using HTTP/1.1 {{RFC9112}}, where characters such as carriage return (CR), line
feed (LF), and COLON are used as delimiters. Implementations MUST perform the
following minimal validation of field names and values:

A field name MUST NOT contain characters in the ranges 0x00-0x20, 0x41-0x5a, or
0x7f-0xff (all ranges inclusive). This specifically excludes all non-visible
ASCII characters, ASCII SP (0x20), and uppercase characters ('A' to 'Z', ASCII
0x41 to 0x5a). With the exception of pseudo-header fields
({{http-control-data}}), which have a name that starts with a single colon,
field names MUST NOT include a colon (ASCII COLON, 0x3a). A field value MUST NOT
contain the zero value (ASCII NUL, 0x00), line feed (ASCII LF, 0x0a), or
carriage return (ASCII CR, 0x0d) at any position. A field value MUST NOT start
or end with an ASCII whitespace character (ASCII SP or HTAB, 0x20 or 0x09).
Note: An implementation that validates fields according to the definitions in
{{Sections 5.1 and 5.5 of RFC9110}} only needs an additional check that field
names do not include uppercase characters.

A request or response that contains a field that violates any of these
conditions MUST be treated as malformed {{malformed-messages}}. In particular,
an intermediary that does not process fields when forwarding messages MUST NOT
forward fields that contain any of the values that are listed as prohibited
above.

When a request message violates one of these requirements, an implementation
SHOULD generate a 400 (Bad Request) status code (see {{Section 15.5.1 of
RFC9110}}), unless a more suitable status code is defined or the status code
cannot be sent (e.g., because the error occurs in a trailer field).

### Connection-Specific Header Fields

HTTP/2 Plus Plus does not use the Connection header field {{Section 7.6.1 of
RFC9110}} to indicate connection-specific header fields; in this protocol,
connection-specific metadata is conveyed by other means. An endpoint MUST NOT
generate an HTTP/2 Plus Plus message containing connection-specific header
fields. This includes the Connection header field and those listed as having
connection-specific semantics in {{Section 7.6.1 of RFC9110}} (that is,
Proxy-Connection, Keep-Alive, Transfer-Encoding, and Upgrade). Any message
containing connection-specific header fields MUST be treated as malformed
({{malformed-messages}}).

The only exception to this is the TE header field, which MAY be present in an
HTTP/2 Plus Plus request; when it is, it MUST NOT contain any value other than
"trailers".

An intermediary transforming an HTTP/1.x message to HTTP/2 Plus Plus MUST remove
connection-specific header fields as discussed in {{Section 7.6.1 of RFC9110}},
or their messages will be treated by other HTTP/2 Plus Plus endpoints as
malformed ({{malformed-messages}}).

## HTTP Control Data

HTTP/2 Plus Plus uses special pseudo-header fields beginning with a ':'
character (ASCII 0x3a) to convey message control data (see {{Section 6.2 of
RFC9110}}).

Pseudo-header fields are not HTTP header fields. Endpoints MUST NOT generate
pseudo-header fields other than those defined in this document. Note that an
extension could negotiate the use of additional pseudo-header fields; see
{{extending}}.

Pseudo-header fields are only valid in the context in which they are defined.
Pseudo-header fields defined for requests MUST NOT appear in responses;
pseudo-header fields defined for responses MUST NOT appear in requests.
Pseudo-header fields MUST NOT appear in a trailer section. Endpoints MUST treat
a request or response that contains undefined or invalid pseudo-header fields as
malformed ({{malformed-messages}}).

All pseudo-header fields MUST appear in the header section before regular
header fields. Any request or response that contains a pseudo-header field that
appears in a header section after a regular header field MUST be treated as
malformed ({{malformed-messages}}).

The same pseudo-header field name MUST NOT appear more than once in a field
block. A field block for an HTTP request or response that contains a repeated
pseudo-header field name MUST be treated as malformed ({{malformed-messages}}).

### Request Pseudo-Header Fields
The following pseudo-header fields are defined for requests:

":method":

: Contains the HTTP method ({{Section 9 of RFC9110}}).

":scheme":

: Contains the scheme portion of the request target URI ({{Section 3.1 of
!RFC3986}}).

: When generating a request directly, or from the scheme of a translated request
(for example, see {{Section 3.3 of RFC9112}}). Scheme is omitted for CONNECT
requests ({{the-connect-method}}).

: ":scheme" is not restricted to "http" and "https" schemed URIs. A proxy or
gateway can translate requests for non-HTTP schemes, enabling the use of HTTP to
interact with non-HTTP services.

":authority"

: Contains the authority portion ({{Section 3.2 of RFC3986}}) of the target URI
({{Section 7.1 of RFC9110}}). The recipient of a request MUST NOT use the Host
header field to determine the target URI if ":authority" is present.

: Clients that generate HTTP/2 Plus Plus requests directly MUST use the
":authority" pseudo-header field to convey authority information, unless there
is no authority information to convey (in which case it MUST NOT generate
":authority").

: Clients MUST NOT generate a request with a Host header field that differs from
the ":authority" pseudo-header field. A server SHOULD treat a request as
malformed if it contains a Host header field that identifies an entity that
differs from the entity in the ":authority" pseudo-header field. The values of
fields need to be normalized to compare them (see {{Section 6.2 of RFC3986}}).
An origin server can apply any normalization method, whereas other servers MUST
perform scheme-based normalization (see {{Section 6.2.3 of RFC3986}}) of the two
fields.

: An intermediary that forwards a request over HTTP/2 Plus Plus MUST construct
an ":authority" pseudo-header field using the authority information from the
control data of the original request, unless the original request's target URI
does not contain authority information (in which case it MUST NOT generate
":authority"). Note that the Host header field is not the sole source of this
information; see {{Section 7.2 of RFC9110}}.

: An intermediary that needs to generate a Host header field (which might be
necessary to construct an HTTP/1.1 request) MUST use the value from the
":authority" pseudo-header field as the value of the Host field, unless the
intermediary also changes the request target. This replaces any existing Host
field to avoid potential vulnerabilities in HTTP routing.

: An intermediary that forwards a request over HTTP/2 Plus Plus MAY retain any
Host header field.

: Note that request targets for CONNECT or asterisk-form OPTIONS requests never
include authority information; see {{Sections 7.1 and 7.2 of RFC9110}}.

: ":authority" MUST NOT include the deprecated userinfo subcomponent for "http"
or "https" schemed URIs.

":path":

: Contains the path and query parts of the target URI (the absolute-path
production and, optionally, a '?' character followed by the query production;
see {{Section 4.1 of RFC9110}}). A request in asterisk form (for OPTIONS)
includes the value '*' for the ":path" pseudo-header field.

: This pseudo-header field MUST NOT be empty for "http" or "https" URIs; "http"
or "https" URIs that do not contain a path component MUST include a value of
'/'. The exceptions to this rule are:

: * an OPTIONS request for an "http" or "https" URI that does not include a path
component; these MUST include a ":path" pseudo-header field with a value of '*'
(see {{Section 7.1 of RFC9110}}).

: * CONNECT requests ({{the-connect-method}}), where the ":path" pseudo-header
field is omitted.

":protocol":

: NEW MTI! See {{!RFC8446}}

All HTTP/2 Plus Plus requests MUST include exactly one valid value for the
":method", ":scheme", and ":path" pseudo-header fields, unless they are CONNECT
requests ({{the-connect-method}}) - TODO explain :protocol. An HTTP request that
omits mandatory pseudo-header fields is malformed ({{malformed-messages}}).

Individual requests do not carry an explicit indicator of protocol version. All
HTTP/2 Plus Plus requests implicitly have a protocol version of "2.lol" (see
{{Section 6.2 of RFC9110}}).

### Response Pseudo-Header Fields

For HTTP/2 Plus Plus responses, a single ":status" pseudo-header field is
defined that carries the HTTP status code field (see {{Section 15 of RFC9110}}).
This pseudo-header field MUST be included in all responses, including interim
responses; otherwise, the response is malformed ({{malformed-messages}}).

HTTP/2 Plus Plus responses implicitly have a protocol version of "2.lol".

## The CONNECT Method

TODO Consider deprecating the RFC7540 use and instead mandate
{{?I-D.ietf-httpbis-connect-tcp}}. This would affect other references to CONNECT
in this draft.

## The Upgrade Header Field

HTTP/2 Plus Plus does not support the 101 (Switching Protocols) informational
status code {{Section 15.2.2 of RFC9110}}.

The semantics of 101 (Switching Protocols) aren't applicable to a multiplexed
protocol. Similar functionality can be achieved through the :protocol
psuedo-header.

## Request Reliability

In general, an HTTP client is unable to retry a non-idempotent request when an
error occurs because there is no means to determine the nature of the error (see
{{Section 9.2.2 of RFC9110}}). It is possible that some server processing
occurred prior to the error, which could result in undesirable effects if the
request were reattempted.

HTTP/2 Plus Plus provides two mechanisms for providing a guarantee to a client
that a request has not been processed:

* The GOAWAY frame indicates the highest stream number that might have been
  processed. Requests on streams with higher numbers are therefore guaranteed to
  be safe to retry.

* The REFUSED_STREAM error code can be included in a RESET_STREAM frame to
indicate that the stream is being closed prior to any processing having
occurred. Any request that was sent on the reset stream can be safely retried.
Requests that have not been processed have not failed; clients MAY automatically
retry them, even those with non-idempotent methods.

A server MUST NOT indicate that a stream has not been processed unless it can
guarantee that fact. If frames that are on a stream are passed to the
application layer for any stream, then REFUSED_STREAM MUST NOT be used for that
stream, and a GOAWAY frame MUST include a stream identifier that is greater than
or equal to the given stream identifier.

In addition to these mechanisms, the PING frame provides a way for a client to
easily test a connection. Connections that remain idle can become broken,
because some middleboxes (for instance, network address translators or load
balancers) silently discard connection bindings. The PING frame allows a client
to safely test whether a connection is still active without sending a request.

# HTTP/2 Plus Plus Connections

This section outlines attributes of HTTP that improve interoperability, reduce
exposure to known security vulnerabilities, or reduce the potential for
implementation variation.

## Connection Management
HTTP/2 Plus Plus connections are persistent. For best performance, it is
expected that clients will not close connections until it is determined that no
further communication with a server is necessary (for example, when a user
navigates away from a particular web page) or until the server closes the
connection.

Clients SHOULD NOT open more than one HTTP/2 Plus Plus connection to a given
host and port pair, where the host is derived from a URI, a selected alternative
service {{?ALT-SVC=RFC7838}}, or a configured proxy.

A client can create additional connections as replacements, either to replace
connections that are near to exhausting the available stream identifier space
({{stream-states}}.1), to refresh the keying material for a TLS connection, or
to replace connections that have encountered errors
({{connection-error-handling}}).

A client MAY open multiple connections to the same IP address and TCP port using
different Server Name Indication {{?TLS-EXT=RFC6066}} values or to provide
different TLS client certificates but SHOULD avoid creating multiple connections
with the same configuration.

Servers are encouraged to maintain open connections for as long as possible but
are permitted to terminate idle connections if necessary. When either endpoint
chooses to close the transport-layer TCP connection, the terminating endpoint
SHOULD first send a GOAWAY ({{goaway}}) frame so that both endpoints can
reliably determine whether previously sent frames have been processed and
gracefully complete or terminate any necessary remaining tasks.

### Connection Reuse

Connections that are made to an origin server, either directly or through a
tunnel created using the CONNECT method ({{the-connect-method}}), MAY be reused
for requests with multiple different URI authority components. A connection can
be reused as long as the origin server is authoritative (Section 10). For TCP
connections without TLS, this depends on the host having resolved to the same IP
address.

For "https" resources, connection reuse additionally depends on having a
certificate that is valid for the host in the URI. The certificate presented by
the server MUST satisfy any checks that the client would perform when forming a
new TLS connection for the host in the URI. A single certificate can be used to
establish authority for multiple origins. {{{{field-compression}} of RFC9110}}
describes how a client determines whether a server is authoritative for a URI.

In some deployments, reusing a connection for multiple origins can result in
requests being directed to the wrong origin server. For example, TLS termination
might be performed by a middlebox that uses the TLS Server Name Indication
{{TLS-EXT}} extension to select an origin server. This means that it is possible
for clients to send requests to servers that might not be the intended target
for the request, even though the server is otherwise authoritative.

A server that does not wish clients to reuse connections can indicate that it is
not authoritative for a request by sending a 421 (Misdirected Request) status
code in response to the request (see {{Section 15.5.20 of RFC9110}}).

A client that is configured to use a proxy over HTTP/2 Plus Plus directs
requests to that proxy through a single connection. That is, all requests sent
via a proxy reuse the connection to the proxy.

## Use of TLS Features

Implementations of HTTP/2 Plus Plus MUST use TLS version 1.3 {{TLS}} or higher.
The general TLS usage guidance in {{!TLSBCP=RFC7525}} SHOULD be followed, with
some additional restrictions that are specific to HTTP/2 Plus Plus.

The TLS implementation MUST support the Server Name Indication (SNI) {{TLS-EXT}}
extension to TLS. If the server is identified by a domain name
{{!DNS-TERMS=RFC8499}}, clients MUST send the server_name TLS extension unless
an alternative mechanism to indicate the target host is used.

HTTP/2 Plus Plus servers MUST NOT send post-handshake TLS 1.3 CertificateRequest
messages. HTTP/2 Plus Plus clients MUST treat a TLS post-handshake
CertificateRequest message as a connection error ({{connection-error-handling}})
of type PROTOCOL_ERROR.

The prohibition on post-handshake authentication applies even if the client
offered the "post_handshake_auth" TLS extension. Post-handshake authentication
support might be advertised independently of ALPN {{TLS-ALPN}}. Clients might
offer the capability for use in other protocols, but inclusion of the extension
cannot imply support within HTTP/2 Plus Plus.

{{TLS}} defines other post-handshake messages, NewSessionTicket and KeyUpdate,
which can be used as they have no direct interaction with HTTP/2 Plus Plus.
Unless the use of a new type of TLS message depends on an interaction with the
application-layer protocol, that TLS message can be sent after the handshake
completes.

TLS early data MAY be used to send requests, provided that the guidance in
{{!RFC8470}} is observed. Clients send requests in early data assuming initial
values for all server settings.

# Security Considerations

TODO - probably all of {{RFC9113}}.


# IANA Considerations

This document has too many IANA actions to write about right now.


--- back

# Acknowledgments
{:numbered="false"}

Thanks to all the contributors of the HTTP/2 and HTTP/3 specs, along with the
HTTP WG discussion around defining new ALPNs to address issues in HTTP/2
deployments
