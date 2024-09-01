---
title: "Media over QUIC Transport"
abbrev: moq-transport
docname: draft-ietf-moq-transport-latest
date: {DATE}
category: std

ipr: trust200902
area:  "Web and Internet Transport"
submissionType: IETF
workgroup: "Media Over QUIC"
keyword:
 - media over quic
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "moq-wg/moq-transport"
  latest: "https://moq-wg.github.io/moq-transport/draft-ietf-moq-transport.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: L. Curley
    name: Luke Curley
    organization: Discord
    email: kixelated@gmail.com

  -
    ins: K. Pugin
    name: Kirill Pugin
    organization: Meta
    email: ikir@meta.com

  -
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

  -
    ins: V. Vasiliev
    name: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com

  -
    ins: I. Swett
    name: Ian Swett
    organization: Google
    email: ianswett@google.com
    role: editor

normative:
  QUIC: RFC9000
  WebTransport: I-D.ietf-webtrans-http3

informative:

--- abstract

This document defines the core behavior for Media over QUIC Transport
(MOQT), a media transport protocol designed to operate over QUIC and
WebTransport, which have similar functionality. MOQT allows a producer of
media to publish data and have it consumed via subscription by a
multiplicity of endpoints. It supports intermediate content distribution
networks and is designed for high scale and low latency distribution.

--- middle


# Introduction

Media Over QUIC Transport (MOQT) is a protocol that is optimized
for the QUIC protocol {{QUIC}}, either directly or via WebTransport
{{WebTransport}}, for the dissemination of media. MOQT utilizes a
publish/subscribe workflow in which producers of media publish data in
response to subscription requests from a multiplicity of endpoints. MOQT
supports wide range of use-cases with different resiliency and latency
(live, interactive) needs without compromising the scalability and cost
effectiveness associated with content delivery networks.

MOQT is a generic protocol is designed to work in concert with multiple
MoQ Streaming Formats. These MoQ Streaming Formats define how content is
encoded, packaged, and mapped to MOQT objects, along with policies for
discovery and subscription.

* {{model}} describes the object model employed by MOQT.

* {{session}} covers aspects of setting up a MOQT session.

* {{priorities}} covers mechanisms for prioritizing subscriptions.

* {{relays-moq}} covers behavior at the relay entities.

* {{message}} covers how messages are encoded on the wire.


## Motivation

The development of MOQT is driven by goals in a number of areas -
specifically latency, the robustness of QUIC, workflow efficiency and
relay support.

### Latency

HTTP Adaptive Streaming (HAS) has been successful at achieving scale
although often at the cost of latency. Latency is necessary to correct
for variable network throughput. Ideally live content is consumed at the
same bitrate it is produced. End-to-end latency would be fixed and only
subject to encoding and transmission delays. Unfortunately, networks
have variable throughput, primarily due to congestion. Attempting to
deliver content encoded at a higher bitrate than the network can support
causes queuing along the path from producer to consumer. The speed at
which a protocol can detect and respond to queuing determines the
overall latency. TCP-based protocols are simple but are slow to detect
congestion and suffer from head-of-line blocking. Protocols utilizing
UDP directly can avoid queuing, but the application is then responsible
for the complexity of fragmentation, congestion control, retransmissions,
receiver feedback, reassembly, and more. One goal of MOQT is to achieve
the best of both these worlds: leverage the features of QUIC to create a
simple yet flexible low latency protocol that can rapidly detect and
respond to congestion.

### Leveraging QUIC

The parallel nature of QUIC streams can provide improvements in the face
of loss. A goal of MOQT is to design a streaming protocol to leverage
the transmission benefits afforded by parallel QUIC streams as well
exercising options for flexible loss recovery. Applying {{QUIC}} to HAS
via HTTP/3 has not yet yielded generalized improvements in
throughput. One reason for this is that sending segments down a single
QUIC stream still allows head-of-line blocking to occur.

### Universal

Internet delivered media today has protocols optimized for ingest and
separate protocols optimized for distribution. This protocol switch in
the distribution chain necessitates intermediary origins which
re-package the media content. While specialization can have its
benefits, there are gains in efficiency to be had in not having to
re-package content. A goal of MOQT is to develop a single protocol which
can be used for transmission from contribution to distribution. A
related goal is the ability to support existing encoding and packaging
schemas, both for backwards compatibility and for interoperability with
the established content preparation ecosystem.

### Relays

An integral feature of a protocol being successful is its ability to
deliver media at scale. Greatest scale is achieved when third-party
networks, independent of both the publisher and subscriber, can be
leveraged to relay the content. These relays must cache content for
distribution efficiency while simultaneously routing content and
deterministically responding to congestion in a multi-tenant network. A
goal of MOQT is to treat relays as first-class citizens of the protocol
and ensure that objects are structured such that information necessary
for distribution is available to relays while the media content itself
remains opaque and private.

## Terms and Definitions

{::boilerplate bcp14-tagged}

Client:

: The party initiating a MoQ transport session.

Server:

: The party accepting an incoming transport session.

Endpoint:

: A Client or Server.

Publisher:

: An endpoint that handles subscriptions by sending requested Objects from the requested track.

Subscriber:

: An endpoint that subscribes to and receives tracks.

Original Publisher:

: The initial publisher of a given track.

End Subscriber:

: A subscriber that initiates a subscription and does not send the data on to other subscribers.

Relay:

: An entitly that is both a Publisher and a Subscriber, but not the Original
Publisher or End Subscriber.

Upstream:

: In the direction of the Original Publisher

Downstream:

: In the direction of the End Subscriber(s)

Transport session:

: A raw QUIC connection or a WebTransport session.

Congestion:

: Packet loss and queuing caused by degraded or overloaded networks.

Group:

: A temporal sequence of objects. A group represents a join point in a
  track. See ({{model-group}}).

Object:

: An object is an addressable unit whose payload is a sequence of
  bytes. Objects form the base element in the MOQT model. See
  ({{model-object}}).

Track:

: An encoded bitstream. Tracks contain a sequential series of one or
  more groups and are the subscribable entity with MOQT.
  See ({{model-track}}).


## Notational Conventions

This document uses the conventions detailed in ({{?RFC9000, Section 1.3}})
when describing the binary encoding.

As a quick reference, the following list provides a non normative summary
of the parts of RFC9000 field syntax that are used in this specification.

x (L):

: Indicates that x is L bits long

x (i):

: Indicates that x holds an integer value using the variable-length
  encoding as described in ({{?RFC9000, Section 16}})

x (..):

: Indicates that x can be any length including zero bits long.  Values
 in this format always end on a byte boundary.

[x (L)]:

: Indicates that x is optional and has a length of L

x (L) ...:

: Indicates that x is repeated zero or more times and that each instance
  has a length of L

This document extends the RFC9000 syntax and with the additional field types:

x (b):

: Indicates that x consists of a variable length integer encoding as
  described in ({{?RFC9000, Section 16}}), followed by that many bytes
  of binary data

x (f):

: Indicates that x is a flag and is encoded as a single byte with the
  value 0 or 1. A value of 0 indicates the flag is false or off, while a
  value of 1 indicates the flag is true or on. Any other value is a
  protocol error and SHOULD terminate the session with a Protocol
  Violation ({{session-termination}}).


To reduce unnecessary use of bandwidth, variable length integers SHOULD
be encoded using the least number of bytes possible to represent the
required value.


# Object Model {#model}

MOQT has a hierarchical object model for data, comprised of objects,
groups and tracks.

## Objects {#model-object}

The basic data element of MOQT is an object.  An object is an
addressable unit whose payload is a sequence of bytes.  All objects
belong to a group, indicating ordering and potential
dependencies. {{model-group}}  An object is uniquely identified by
its track namespace, track name, group ID, and object ID, and must be an
identical sequence of bytes regardless of how or where it is retrieved.
An Object can become unavailable, but it's contents MUST NOT change over
time.

Objects are comprised of two parts: metadata and a payload.  The metadata is
never encrypted and is always visible to relays. The payload portion may be
encrypted, in which case it is only visible to the Original Publisher and End
Subscribers. The application is solely responsible for the content of the object
payload. This includes the underlying encoding, compression, any end-to-end
encryption, or authentication. A relay MUST NOT combine, split, or otherwise
modify object payloads.

## Peeps {#model-peep}

A peep is a sequence of one or more objects from the same group
({{model-group}}) in ascending order by Object ID. Objects in a peep
have a dependency and priority relationship consistent with sharing a QUIC
stream. In some cases, a Group will be most effectively delivered using more
than one QUIC stream.

When a Track's forwarding preference (see {{object-fields}}) is "Track" or
"Datagram", Objects are not sent in Peeps, no Peep IDs are assigned, and the
description in the remainder of this section does not apply.

QUIC streams offer in-order reliable delivery and the ability to cancel sending
and retransmission of data. Furthermore, many implementations offer the ability
to control the relative priority of streams, which allows control over the
scheduling of sending data on active streams.

Every object within a Group belongs to exactly one Peep.

Objects from two peeps cannot be sent on the same QUIC stream. Objects from the
same Peep MUST NOT be sent on different QUIC streams, unless one of the streams
was reset prematurely, or upstream conditions have forced objects from a Peep
to be sent out of Object ID order.

Original publishers assign each Peep a Peep ID, and do so as they see fit.  The
scope of a Peep ID is a Group, so Peeps from different Groups MAY share a Peep
ID without implying any relationship between them. In general, publishers assign
objects to peeps in order to leverage the features of QUIC streams as described
above.

An example strategy for using QUIC stream properties follows. If object B is
dependent on object A, then delivery of B can follow A, i.e. A and B can be
usefully delivered over a single QUIC stream. Furthermore, in this example:

- If an object is dependent on all previous objects in a Peep, it is added to
that Peep.

- If an object is not dependent on all of the objects in a Peep, it goes in
a different Peep.

- There are often many ways to compose Peeps that meet these criteria. Where
possible, choose the composition that results in the fewest Peeps in a group
to minimize the number of QUIC streams used.


## Groups {#model-group}

A group is a collection of objects and is a sub-unit of a track
({{model-track}}).  Objects within a group SHOULD NOT depend on objects
in other groups.  A group behaves as a join point for subscriptions.
A new subscriber might not want to receive the entire track, and may
instead opt to receive only the latest group(s).  The publisher then
selectively transmits objects based on their group membership.

## Track {#model-track}

A track is a sequence of groups ({{model-group}}). It is the entity
against which a subscriber issues a subscription request.  A subscriber
can request to receive individual tracks starting at a group boundary,
including any new objects pushed by the publisher while the track is
active.

### Track Naming and Scopes {#track-name}

In MOQT, every track has a track name and a track namespace associated
with it.  A track name identifies an individual track within the
namespace.

A MOQT scope is a set of servers (as identified by their connection
URIs) for which the tuple of Track Name and Track Namespace are
guaranteed to be unique and identify a specific track. It is up to
the application using MOQT to define how broad or narrow the scope is.
An application that deals with connections between devices
on a local network may limit the scope to a single connection; by
contrast, an application that uses multiple CDNs to serve media may
require the scope to include all of those CDNs.

Because the tuple of Track Namespace and Track Name are unique within an
MOQT scope, they can be used as a cache key.
MOQT does not provide any in-band content negotiation methods similar to
the ones defined by HTTP ({{?RFC9110, Section 10}}); if, at a given
moment in time, two tracks within the same scope contain different data,
they have to have different names and/or namespaces.

In this specification, both the Track Namespace and the Track Name are
not constrained to a specific encoding. They carry a sequence of
bytes and comparison between two Track Namespaces or Track Names is
done by exact comparison of the bytes. Specifications that use MoQ Transport
may constrain the information in these fields, for example by restricting
them to UTF-8. Any specification that does needs to specify the
canonicalization into the bytes in the Track Namespace or Track Name
such that exact comparison works.

### Connection URL

Each track MAY have one or more associated connection URLs specifying
network hosts through which a track may be accessed. The syntax of the
Connection URL and the associated connection setup procedures are
specific to the underlying transport protocol usage {{session}}.


# Sessions {#session}

## Session establishment {#session-establishment}

This document defines a protocol that can be used interchangeably both
over a QUIC connection directly [QUIC], and over WebTransport
[WebTransport].  Both provide streams and datagrams with similar
semantics (see {{?I-D.ietf-webtrans-overview, Section 4}}); thus, the
main difference lies in how the servers are identified and how the
connection is established.  When using QUIC, datagrams MUST be
supported via the [QUIC-DATAGRAM] extension, which is already a
requirement for WebTransport over HTTP/3.

There is no definition of the protocol over other transports,
such as TCP, and applications using MoQ might need to fallback to
another protocol when QUIC or WebTransport aren't available.

### WebTransport

A MOQT server that is accessible via WebTransport can be identified
using an HTTPS URI ({{!RFC9110, Section 4.2.2}}).  A MOQT session can be
established by sending an extended CONNECT request to the host and the
path indicated by the URI, as described in {{WebTransport, Section 3}}.

### QUIC

A MOQT server that is accessible via native QUIC can be identified by a
URI with a "moq" scheme.  The "moq" URI scheme is defined as follows,
using definitions from {{!RFC3986}}:

~~~~~~~~~~~~~~~
moq-URI = "moqt" "://" authority path-abempty [ "?" query ]
~~~~~~~~~~~~~~~

The `authority` portion MUST NOT contain a non-empty `host` portion.
The `moq` URI scheme supports the `/.well-known/` path prefix defined in
{{!RFC8615}}.

This protocol does not specify any semantics on the `path-abempty` and
`query` portions of the URI.  The contents of those are left up to the
application.

The client can establish a connection to a MoQ server identified by a
given URI by setting up a QUIC connection to the host and port
identified by the `authority` section of the URI.  The `path-abempty`
and `query` portions of the URI are communicated to the server using the
PATH parameter ({{path}}) which is sent in the CLIENT_SETUP message at the
start of the session.  The ALPN value {{!RFC7301}} used by the protocol
is `moq-00`.

## Version and Extension Negotiation {#version-negotiation}

Endpoints use the exchange of Setup messages to negotiate the MOQT version and
any extensions to use.

The client indicates the MOQT versions it supports in the CLIENT_SETUP message
(see {{message-setup}}). It also includes the union of all Setup Parameters
{{setup-params}} required for a handshake by any of those versions.

Within any MOQT version, clients request the use of extensions by adding Setup
parameters corresponding to that extension. No extensions are defined in this
document.

The server replies with a SERVER_SETUP message that indicates the chosen
version, includes all parameters required for a handshake in that version, and
parameters for every extension requested by the client that it supports.

New versions of MOQT MUST specify which existing extensions can be used with
that version. New extensions MUST specify the existing versions with which they
can be used.

If a given parameter carries the same information in multiple versions,
but might have different optimal values in those versions, there SHOULD be
separate Setup parameters for that information in each version.

## Session initialization {#session-init}

The first stream opened is a client-initiated bidirectional control stream where
the peers exchange Setup messages ({{message-setup}}).  All messages defined in
this draft except OBJECT and OBJECT_WITH_LENGTH are sent on the control stream
after the Setup message. Control messages MUST NOT be sent on any other stream,
and a peer receiving a control message on a different stream closes the session
as a 'Protocol Violation'. Objects MUST NOT be sent on the control stream, and a
peer receiving an Object on the control stream closes the session as a 'Protocol
Violation'.

This draft only specifies a single use of bidirectional streams. Objects are
sent on unidirectional streams.  Because there are no other uses of
bidirectional streams, a peer MAY currently close the session as a
'Protocol Violation' if it receives a second bidirectional stream.

The control stream MUST NOT be abruptly closed at the underlying transport
layer.  Doing so results in the session being closed as a 'Protocol Violation'.

## Stream Cancellation

Streams aside from the control stream MAY be canceled due to congestion
or other reasons by either the publisher or subscriber. Early termination of a
stream does not affect the MoQ application state, and therefore has no
effect on outstanding subscriptions.

## Termination  {#session-termination}

The transport session can be terminated at any point.  When native QUIC
is used, the session is closed using the CONNECTION\_CLOSE frame
({{QUIC, Section 19.19}}).  When WebTransport is used, the session is
closed using the CLOSE\_WEBTRANSPORT\_SESSION capsule ({{WebTransport,
Section 5}}).

The application MAY use any error message and SHOULD use a relevant
code, as defined below:

|------|---------------------------|
| Code | Reason                    |
|-----:|:--------------------------|
| 0x0  | No Error                  |
|------|---------------------------|
| 0x1  | Internal Error            |
|------|---------------------------|
| 0x2  | Unauthorized              |
|------|---------------------------|
| 0x3  | Protocol Violation        |
|------|---------------------------|
| 0x4  | Duplicate Track Alias     |
|------|---------------------------|
| 0x5  | Parameter Length Mismatch |
|------|---------------------------|
| 0x6  | Too Many Subscribes       |
|------|---------------------------|
| 0x10 | GOAWAY Timeout            |
|------|---------------------------|

* No Error: The session is being terminated without an error.

* Internal Error: An implementation specific error occurred.

* Unauthorized: The endpoint breached an agreement, which MAY have been
 pre-negotiated by the application.

* Protocol Violation: The remote endpoint performed an action that was
  disallowed by the specification.

* Duplicate Track Alias: The endpoint attempted to use a Track Alias
  that was already in use.

* Too Many Subscribes: The session was closed because the subscriber used
  a Subscribe ID equal or larger than the current Maximum Subscribe ID.

* GOAWAY Timeout: The session was closed because the client took too long to
  close the session in response to a GOAWAY ({{message-goaway}}) message.
  See session migration ({{session-migration}}).

## Migration {#session-migration}

MoqTransport requires a long-lived and stateful session. However, a service
provider needs the ability to shutdown/restart a server without waiting for all
sessions to drain naturally, as that can take days for long-form media.
MoqTransport avoids this via the GOAWAY message ({{message-goaway}}).

The server sends a GOAWAY message, signaling that the client should establish a
new session and migrate any active subscriptions. The GOAWAY message may contain
a new URI for the new session, otherwise the current URI is reused. The server
SHOULD terminate the session with 'GOAWAY Timeout' after a sufficient timeout if
there are still open subscriptions on a connection.

The GOAWAY message does not immediately impact subscription state. A subscriber
SHOULD individually UNSUBSCRIBE for each existing subscription, while a
publisher MAY reject new SUBSCRIBEs while in the draining state. When the server
is a subscriber, it SHOULD send a GOAWAY message prior to any UNSUBSCRIBE
messages.

After the client receives a GOAWAY, it's RECOMMENDED that the client waits until
there are no more active subscriptions before closing the session with NO_ERROR.
Ideally this is transparent to the application using MOQT, which involves
establishing a new session in the background and migrating active subscriptions
and announcements. The client can choose to delay closing the session if it
expects more OBJECTs to be delivered. The server closes the session with a
'GOAWAY Timeout' if the client doesn't close the session quickly enough.


# Priorities {#priorities}

MoQ priorities allow a subscriber and original publisher to influence
the transmission order of Objects within a session in the presence of
congestion.

Given the critical nature of control messages and their relatively
small size, the control stream SHOULD be prioritized higher than all
subscribed Objects.

The subscriber indicates the priority of a subscription via the
Subscriber Priority field and the original publisher indicates priority
in every stream or datagram header.  As such, the subscriber's priority is a
property of the subscription and the original publisher's priority is a
property of the Track and the Objects it contains. In both cases, a lower
value indicates a higher priority, with 0 being the highest priority.

When Objects are contained in Peeps, all Objects in the Peep have the same
priority.

The Subscriber Priority is considered first when selecting a subscription
to send data on within a given session. When two or more subscriptions
have equal subscriber priority, the original publisher priority is considered
next and can change within the track, so subscriptions are prioritized based
on the highest priority data available to send. For example, if the subscription
had data at priority 6 and priority 10 to send, the subscription priority would
be 6. When both the subscriber and original publisher priorities for a
subscription are equal, how much data to send from each subscription is
implementation-dependent, but the expectation is that all subscriptions will
be able to send some data.

The subscriber's priority can be changed via a SUBSCRIBE_UPDATE message.
This updates the priority of all unsent data within the subscription,
though the details of the reprioitization are implementation-specific.

Subscriptions have a Group Order of either 'Ascending' or 'Descending',
which indicates whether the lowest or highest Group Id SHOULD be sent first
when multiple Groups are available to send.  A subscriber can specify either
'Ascending' or 'Descending' in the SUBSCRIBE message or they can specify they
want to use the Original Publisher's Group Order, which is indicated in
the corresponding SUBSCRIBE_OK.

Within the same Group, and the same priority level,
Objects with a lower Object Id are always sent before objects with a
higher Object Id, regardless of the specified Group Order. If the group
contains more than one Peep and the priority varies between these Peeps,
higher priority Peeps are sent before lower priority Peeps. If the specified
priority of two Peeps in a Group are equal, the lower Peep ID has priority.
Within a Peep, Objects MUST bs sent in increasing Object ID order.

The Group Order cannot be changed via a SUBSCRIBE_UPDATE message, and
instead an UNSUBSCRIBE and SUBSCRIBE can be used.

Relays SHOULD respect the subscriber and original publisher's priorities.
Relays SHOULD NOT directly use Subscriber Priority or Group Order
from incoming subscriptions for upstream subscriptions. Relays use of
Subscriber Priority for upstream subscriptions can be based on
factors specific to it, such as the popularity of the
content or policy, or relays can specify the same value for all
upstream subscriptions.

MoQ Sessions can span multiple namespaces, and priorities might not
be coordinated across namespaces.  The subscriber's priority is
considered first, so there is a mechanism for a subscriber to fix
incompatibilities between different namespaces prioritization schemes.
Additionally, it is anticipated that when multiple namespaces
are present within a session, the namespaces could be coordinating,
possibly part of the same application.  In cases when pooling among
namespaces is expected to cause issues, multiple MoQ sessions, either
within a single connection or on multiple connections can be used.


# Relays {#relays-moq}

Relays are leveraged to enable distribution scale in the MoQ
architecture. Relays can be used to form an overlay delivery network,
similar in functionality to Content Delivery Networks
(CDNs). Additionally, relays serve as policy enforcement points by
validating subscribe and publish requests at the edge of a network.

Relays can cache Objects, but are not required to.

## Subscriber Interactions

Subscribers interact with the Relays by sending a SUBSCRIBE
({{message-subscribe-req}}) control message for the tracks of
interest. Relays MUST ensure subscribers are authorized to access the
content associated with the track. The authorization
information can be part of subscription request itself or part of the
encompassing session. The specifics of how a relay authorizes a user are
outside the scope of this specification.

The subscriber making the subscribe request is notified of the result of
the subscription, via SUBSCRIBE_OK ({{message-subscribe-ok}}) or the
SUBSCRIBE_ERROR {{message-subscribe-error}} control message.
The entity receiving the SUBSCRIBE MUST send only a single response to
a given SUBSCRIBE of either SUBSCRIBE_OK or SUBSCRIBE_ERROR.
If a relay does not already have a subscription for the track,
or if the subscription does not cover all the requested Objects, it
will need to make an upstream subscription.  The relay SHOULD NOT
return a SUBCRIBE_OK until at least one SUBSCRIBE_OK has been
received for the track, to ensure the Group Order is correct.

For successful subscriptions, the publisher maintains a list of
subscribers for each track. Each new OBJECT belonging to the
track within the subscription range is forwarded to each active
subscriber, dependent on the congestion response. A subscription
remains active until the publisher of the track terminates the
track with a SUBSCRIBE_DONE (see {{message-subscribe-done}}).

Objects MUST NOT be sent for unsuccessful subscriptions, and if a subscriber
receives a SUBSCRIBE_ERROR after receiving objects, it MUST close the session
with a 'Protocol Violation'.

A relay MUST not reorder or drop objects received on a multi-object stream when
forwarding to subscribers, unless it has application specific information.

Relays MAY aggregate authorized subscriptions for a given track when
multiple subscribers request the same track. Subscription aggregation
allows relays to make only a single upstream subscription for the
track. The published content received from the upstream subscription
request is cached and shared among the pending subscribers.

The application SHOULD use a relevant error code in SUBSCRIBE_ERROR,
as defined below:

|------|---------------------------|
| Code | Reason                    |
|-----:|:--------------------------|
| 0x0  | Internal Error            |
|------|---------------------------|
| 0x1  | Invalid Range             |
|------|---------------------------|
| 0x2  | Retry Track Alias         |
|------|---------------------------|

The application SHOULD use a relevant status code in
SUBSCRIBE_DONE, as defined below:

|------|---------------------------|
| Code | Reason                    |
|-----:|:--------------------------|
| 0x0  | Unsubscribed              |
|------|---------------------------|
| 0x1  | Internal Error            |
|------|---------------------------|
| 0x2  | Unauthorized              |
|------|---------------------------|
| 0x3  | Track Ended               |
|------|---------------------------|
| 0x4  | Subscription Ended        |
|------|---------------------------|
| 0x5  | Going Away                |
|------|---------------------------|
| 0x6  | Expired                   |
|------|---------------------------|

## Publisher Interactions

Publishing through the relay starts with publisher sending ANNOUNCE
control message with a `Track Namespace` ({{model-track}}).

Relays MUST ensure that publishers are authorized by:

- Verifying that the publisher is authorized to publish the content
  associated with the set of tracks whose Track Namespace matches the
  announced namespace. Specifics of where the authorization happens,
  either at the relays or forwarded for further processing, depends on
  the way the relay is managed and is application specific (typically
  based on prior business agreement).

Relays respond with an ANNOUNCE_OK or ANNOUNCE_ERROR control message
providing the result of announcement. The entity receiving the
ANNOUNCE MUST send only a single response to a given ANNOUNCE of
either ANNOUNCE_OK or ANNOUNCE_ERROR.  When a publisher wants to stop
new subscriptions for an announced namespace it sends an UNANNOUNCE.
A subscriber indicates it will no longer route subscriptions for a
namespace it previously responded ANNOUNCE_OK to by sending an
ANNOUNCE_CANCEL.

A relay manages sessions from multiple publishers and subscribers,
connecting them based on the track namespace. This MUST use an exact
match on track namespace unless otherwise negotiated by the application.
For example, a SUBSCRIBE namespace=foobar message will be forwarded to
the session that sent ANNOUNCE namespace=foobar.

OBJECT message headers carry a short hop-by-hop `Track Alias` that maps to
the Full Track Name (see {{message-subscribe-ok}}). Relays use the
`Track Alias` of an incoming OBJECT message to identify its track and find
the active subscribers for that track. Relays MUST forward OBJECT messages to
matching subscribers in accordance to each subscription's priority, group order,
and delivery timeout.

## Relay Object Handling

MOQT encodes the delivery information for a stream via OBJECT headers
({{message-object}}).  A relay MUST NOT modify Object properties when
forwarding.

A relay MUST treat the object payload as opaque.  A relay MUST NOT
combine, split, or otherwise modify object payloads.  A relay SHOULD
prioritize sending Objects based on {{priorities}}.

A publisher SHOULD begin sending incomplete objects when available to
avoid incurring additional latency.

A relay that reads from a stream and writes to stream in order will
introduce head-of-line blocking.  Packet loss will cause stream data to
be buffered in the library, awaiting in order delivery, which will
increase latency over additional hops.  To mitigate this, a relay SHOULD
read and write stream data out of order subject to flow control
limits.  See section 2.2 in {{QUIC}}.

# Control Messages {#message}

MOQT uses a single bidirectional stream to exchange control messages, as
defined in {{session-init}}.  Every signle message on the control stream is
formatted as follows:

~~~
MOQT Control Message {
  Message Type (i),
  Message Payload (..),
}
~~~
{: #moq-transport-message-format title="MOQT Message"}

|-------|-----------------------------------------------------|
| ID    | Messages                                            |
|------:|:----------------------------------------------------|
| 0x2   | SUBSCRIBE_UPDATE ({{message-subscribe-update-req}})|
|-------|-----------------------------------------------------|
| 0x3   | SUBSCRIBE ({{message-subscribe-req}})               |
|-------|-----------------------------------------------------|
| 0x4   | SUBSCRIBE_OK ({{message-subscribe-ok}})             |
|-------|-----------------------------------------------------|
| 0x5   | SUBSCRIBE_ERROR ({{message-subscribe-error}})       |
|-------|-----------------------------------------------------|
| 0x6   | ANNOUNCE  ({{message-announce}})                    |
|-------|-----------------------------------------------------|
| 0x7   | ANNOUNCE_OK ({{message-announce-ok}})               |
|-------|-----------------------------------------------------|
| 0x8   | ANNOUNCE_ERROR ({{message-announce-error}})         |
|-------|-----------------------------------------------------|
| 0x9   | UNANNOUNCE  ({{message-unannounce}})                |
|-------|-----------------------------------------------------|
| 0xA   | UNSUBSCRIBE ({{message-unsubscribe}})               |
|-------|-----------------------------------------------------|
| 0xB   | SUBSCRIBE_DONE ({{message-subscribe-done}})         |
|-------|-----------------------------------------------------|
| 0xC   | ANNOUNCE_CANCEL ({{message-announce-cancel}})       |
|-------|-----------------------------------------------------|
| 0xD   | TRACK_STATUS_REQUEST ({{message-track-status-req}}) |
|-------|-----------------------------------------------------|
| 0xE   | TRACK_STATUS ({{message-track-status}})             |
|-------|-----------------------------------------------------|
| 0x10  | GOAWAY ({{message-goaway}})                         |
|-------|-----------------------------------------------------|
| 0x40  | CLIENT_SETUP ({{message-setup}})                    |
|-------|-----------------------------------------------------|
| 0x41  | SERVER_SETUP ({{message-setup}})                    |
|-------|-----------------------------------------------------|

An endpoint that receives an unknown message type MUST close the session.

## Parameters {#params}

Some messages include a Parameters field that encode optional message
elements. They contain a type, length, and value.

Senders MUST NOT repeat the same parameter type in a message. Receivers
SHOULD check that there are no duplicate parameters and close the session
as a 'Protocol Violation' if found.

Receivers ignore unrecognized parameters.

The format of Parameters is as follows:

~~~
Parameter {
  Parameter Type (i),
  Parameter Length (i),
  Parameter Value (..),
}
~~~
{: #moq-param format title="MOQT Parameter"}

Parameter Type is an integer that indicates the semantic meaning of the
parameter. Setup message parameters use a namespace that is constant across all
MoQ Transport versions. All other messages use a version-specific namespace. For
example, the integer '1' can refer to different parameters for Setup messages
and for all other message types.

SETUP message parameter types are defined in {{setup-params}}. Version-
specific parameter types are defined in {{version-specific-params}}.

The Parameter Length field of the String Parameter encodes the length
of the Parameter Value field in bytes.

Each parameter description will indicate the data type in the Parameter Value
field. If a receiver understands a parameter type, and the parameter length
implied by that type does not match the Parameter Length field, the receiver
MUST terminate the session with error code 'Parameter Length Mismatch'.

### Version Specific Parameters {#version-specific-params}

Each version-specific parameter definition indicates the message types in which
it can appear. If it appears in some other type of message, it MUST be ignored.
Note that since Setup parameters use a separate namespace, it is impossible for
these parameters to appear in Setup messages.

#### AUTHORIZATION INFO {#authorization-info}

AUTHORIZATION INFO parameter (key 0x02) identifies a track's authorization
information in a SUBSCRIBE or ANNOUNCE message. This parameter is populated for
cases where the authorization is required at the track level. The value is an
ASCII string.

#### DELIVERY TIMEOUT Parameter {#delivery-timeout}

The DELIVERY TIMEOUT parameter (key 0x03) MAY appear in a SUBSCRIBE,
SUBSCRIBE_OK, or a SUBSCRIBE_UDPATE message.  It is the duration in milliseconds
the relay SHOULD continue to attempt forwarding Objects after they have been
received.  The start time for the timeout is based on when the beginning of the
Object is received, and does not depend upon the forwarding preference. There is
no explicit signal that an Object was not sent because the delivery timeout
was exceeded.

If both the subscriber and publisher specify the parameter, they use the min of the
two values for the subscription.  The publisher SHOULD always specify the value
received from an upstream subscription when there is one, and nothing otherwise.
If an earlier Object arrives later than subsequent Objects, relays can consider
the receipt time as that of the next later Object, with the assumption that the
Object's data was reordered.

If neither the subscriber or publisher specify DELIVERY TIMEOUT, Objects are
delivered as indicated by their Group Order and Priority.

When sent by a subscriber, this parameter is intended to be specific to a
subscription, so it SHOULD NOT be forwarded upstream by a relay that intends
to serve multiple subscriptions for the same track.

Publishers SHOULD consider whether the entire Object is likely to be delivered
before sending any data for that Object, taking into account priorities,
congestion control, and any other relevant information.

#### MAX CACHE DURATION Parameter {#max-cache-duration}

MAX_CACHE_DURATION (key 0x04): An integer expressing a number of milliseconds. If
present, the relay MUST NOT start forwarding any individual Object received
through this subscription after the specified number of milliseconds has elapsed
since the beginning of the Object was received.  This means Objects earlier
in a multi-object stream will expire earlier than Objects later in the stream.
Once Objects have expired, their state becomes unknown, and a relay that
handles a subscription that includes those Objects re-requests them.


## CLIENT_SETUP and SERVER_SETUP {#message-setup}

The `CLIENT_SETUP` and `SERVER_SETUP` messages are the first messages exchanged
by the client and the server; they allows the peers to establish the mutually
supported version and agree on the initial configuration before any objects are
exchanged. It is a sequence of key-value pairs called Setup parameters; the
semantics and format of which can vary based on whether the client or server is
sending.  To ensure future extensibility of MOQT, the peers MUST ignore unknown
setup parameters. TODO: describe GREASE for those.

The wire format of the Setup messages are as follows:

~~~
CLIENT_SETUP Message Payload {
  Number of Supported Versions (i),
  Supported Version (i) ...,
  Number of Parameters (i) ...,
  Setup Parameters (..) ...,
}

SERVER_SETUP Message Payload {
  Selected Version (i),
  Number of Parameters (i) ...,
  Setup Parameters (..) ...,
}
~~~
{: #moq-transport-setup-format title="MOQT Setup Messages"}

The available versions and Setup parameters are detailed in the next sections.

### Versions {#setup-versions}

MoQ Transport versions are a 32-bit unsigned integer, encoded as a varint.
This version of the specification is identified by the number 0x00000001.
Versions with the most significant 16 bits of the version number cleared are
reserved for use in future IETF consensus documents.

The client offers the list of the protocol versions it supports; the
server MUST reply with one of the versions offered by the client. If the
server does not support any of the versions offered by the client, or
the client receives a server version that it did not offer, the
corresponding peer MUST close the session.

\[\[RFC editor: please remove the remainder of this section before
publication.]]

The version number for the final version of this specification (0x00000001), is
reserved for the version of the protocol that is published as an RFC.
Version numbers used to identify IETF drafts are created by adding the draft
number to 0xff000000. For example, draft-ietf-moq-transport-13 would be
identified as 0xff00000D.

### Setup Parameters {#setup-params}

#### ROLE {#role}

The ROLE parameter (key 0x00) allows each endpoint to independently specify what
functionality they support for the session. It has three possible values,
which are of type varint:

0x01: Publisher

: The endpoint can process subscriptions and send objects, but not subscribe.
  The endpoint MUST NOT send a SUBSCRIBE message and an ANNOUNCE MUST NOT be
  sent to it.

0x02: Subscriber

: The endpoint can send subscriptions and receive objects, but not publish.
  The endpoint MUST NOT send an ANNOUNCE message and a SUBSCRIBE MUST NOT be
  sent to it.

0x03: PubSub

: The endpoint can act as a publisher or subscriber, and can send or process
  any message type.

Both endpoints MUST send a ROLE parameter with one of the three values
specified above. Both endpoints MUST close the session if the ROLE
parameter is missing or is not one of the three above-specified values.

#### PATH {#path}

The PATH parameter (key 0x01) allows the client to specify the path of
the MoQ URI when using native QUIC ({{QUIC}}).  It MUST NOT be used by
the server, or when WebTransport is used.  If the peer receives a PATH
parameter from the server, or when WebTransport is used, it MUST close
the connection. It follows the URI formatting rules {{!RFC3986}}.

When connecting to a server using a URI with the "moq" scheme, the
client MUST set the PATH parameter to the `path-abempty` portion of the
URI; if `query` is present, the client MUST concatenate `?`, followed by
the `query` portion of the URI to the parameter.

#### MAX_SUBSCRIBE_ID {#max-subscribe-id}

The MAX_SUBSCRIBE_ID parameter (key 0x02) communicates an initial value for
the Maximum Subscribe ID to the receiving subscriber. The default value is 0,
so if not specified, the peer MUST NOT create subscriptions.


## GOAWAY {#message-goaway}
The server sends a `GOAWAY` message to initiate session migration
({{session-migration}}) with an optional URI.

The server MUST terminate the session with a Protocol Violation
({{session-termination}}) if it receives a GOAWAY message. The client MUST
terminate the session with a Protocol Violation ({{session-termination}}) if it
receives multiple GOAWAY messages.

~~~
GOAWAY Message {
  New Session URI (b)
}
~~~
{: #moq-transport-goaway-format title="MOQT GOAWAY Message"}

* New Session URI: The client MUST use this URI for the new session if provided.
  If the URI is zero bytes long, the current URI is reused instead. The new
  session URI SHOULD use the same scheme as the current URL to ensure
  compatibility.



## SUBSCRIBE {#message-subscribe-req}

### SUBSCRIBE Format
A subscriber issues a SUBSCRIBE to a publisher to request a track. Subscriptions are made to receive objects close to the live edge for a given track at the publisher.

The format of SUBSCRIBE is as follows:

~~~
SUBSCRIBE Message {
  Subscribe ID (i),
  Track Alias (i),
  Track Namespace (b),
  Track Name (b),
  Subscriber Priority (8),
  Group Order (8),
  Wait For Next Group (f),
  Number of Parameters (i),
  Subscribe Parameters (..) ...
}
~~~
{: #moq-transport-subscribe-format title="MOQT SUBSCRIBE Message"}

* Subscribe ID: The subscriber specified identifier used to manage a
subscription. `Subscribe ID` is a variable length integer that MUST be
unique and monotonically increasing within a session and MUST be less
than the session's Maximum Subscribe ID.

* Track Alias: A session specific identifier for the track.
Messages that reference a track, such as OBJECT ({{message-object}}),
reference this Track Alias instead of the Track Name and Track Namespace to
reduce overhead. If the Track Alias is already being used for a different
track, the publisher MUST close the session with a Duplicate Track Alias
error ({{session-termination}}).

* Track Namespace: Identifies the namespace of the track as defined in
({{track-name}}).

* Track Name: Identifies the track name as defined in ({{track-name}}).

* Subscriber Priority: Specifies the priority of a subscription relative to
other subscriptions in the same session. Lower numbers get higher priority.
See {{priorities}}.

* Group Order: Allows the subscriber to request Objects be delivered in
Ascending (0x1) or Descending (0x2) order by group. See {{priorities}}.
A value of 0x0 indicates the original publisher's Group Order SHOULD be
used. Values larger than 0x2 are a protocol error.

* Wait For Next Group: A value of true specifies an open-ended subscription with objects from the beginning of the next group. A value of false specifies an open-ended subscription beginning from the current object of the current group.

* Subscribe Parameters: The parameters are defined in {{version-specific-params}}.

On successful subscription, the publisher MUST reply with a SUBSCRIBE_OK,
allowing the subscriber to determine the start group/object when not explicitly
specified and the publisher SHOULD start delivering objects.

If a publisher cannot satisfy the requested start or end for the subscription it
MAY send a SUBSCRIBE_ERROR with code 'Invalid Range'. A publisher MUST NOT send
objects from outside the requested start and end.

Subscribers MUST NOT issue more than one subscription for the objects of a track within the same session. If no content has been delivered yet, the subscription starts with the first published or received group.


## SUBSCRIBE_UPDATE {#message-subscribe-update-req}

A subscriber issues a SUBSCRIBE_UPDATE to a publisher to request a change to
a prior subscription.  Subscriptions can only become more narrower, not wider,
because an attempt to widen a subscription could fail.  If Objects before the
start or after the end of the current subscription are needed, a separate
subscription can be made. The start Object MUST NOT decrease and when it increases,
there is no guarantee that a publisher will not have already sent Objects before
the new start Object.  The end Object MUST NOT increase and when it decreases,
there is no guarantee that a publisher will not have already sent Objects after
the new end Object. A publisher SHOULD close the Session as a 'Protocol Violation'
if the SUBSCRIBE_UPDATE violates either rule or if the subscriber specifies a
Subscribe ID that does not exist within the Session.

Unlike a new subscription, SUBSCRIBE_UPDATE can not cause an Object to be
delivered multiple times.  Like SUBSCRIBE, EndGroup and EndObject MUST specify the
same or a later object than StartGroup and StartObject.

The format of SUBSCRIBE_UPDATE is as follows:

~~~
SUBSCRIBE_UPDATE Message {
  Subscribe ID (i),
  StartGroup (i),
  StartObject (i),
  EndGroup (i),
  EndObject (i),
  Subscriber Priority (8),
  Number of Parameters (i),
  Subscribe Parameters (..) ...
}
~~~
{: #moq-transport-subscribe-update-format title="MOQT SUBSCRIBE_UPDATE Message"}

* Subscribe ID: The subscription identifier that is unique within the session.
This MUST match an existing Subscribe ID.

* StartGroup: The start Group ID.

* StartObject: The start Object ID.

* EndGroup: The end Group ID, plus 1. A value of 0 means the subscription is
open-ended.

* EndObject: The end Object ID, plus 1. A value of 0 means the entire group is
requested.

* Subscriber Priority: Specifies the priority of a subscription relative to
other subscriptions in the same session. Lower numbers get higher priority.
See {{priorities}}.

* Subscribe Parameters: The parameters are defined in {{version-specific-params}}.

## UNSUBSCRIBE {#message-unsubscribe}

A subscriber issues a `UNSUBSCRIBE` message to a publisher indicating it is no
longer interested in receiving media for the specified track and Objects
should stop being sent as soon as possible.  The publisher sends a
SUBSCRIBE_DONE to acknowledge the unsubscribe was successful and indicate
the final Object.

The format of `UNSUBSCRIBE` is as follows:

~~~
UNSUBSCRIBE Message {
  Subscribe ID (i)
}
~~~
{: #moq-transport-unsubscribe-format title="MOQT UNSUBSCRIBE Message"}

* Subscribe ID: Subscription Identifier as defined in {{message-subscribe-req}}.

## ANNOUNCE_OK {#message-announce-ok}

The subscriber sends an ANNOUNCE_OK control message to acknowledge the
successful authorization and acceptance of an ANNOUNCE message.

~~~
ANNOUNCE_OK
{
  Track Namespace (b),
}
~~~
{: #moq-transport-announce-ok format title="MOQT ANNOUNCE_OK Message"}

* Track Namespace: Identifies the track namespace in the ANNOUNCE
message for which this response is provided.

## ANNOUNCE_ERROR {#message-announce-error}

The subscriber sends an ANNOUNCE_ERROR control message for tracks that
failed authorization.

~~~
ANNOUNCE_ERROR
{
  Track Namespace (b),
  Error Code (i),
  Reason Phrase (b),
}
~~~
{: #moq-transport-announce-error format title="MOQT ANNOUNCE_ERROR Message"}

* Track Namespace: Identifies the track namespace in the ANNOUNCE
message for which this response is provided.

* Error Code: Identifies an integer error code for announcement failure.

* Reason Phrase: Provides the reason for announcement error.

## ANNOUNCE_CANCEL {#message-announce-cancel}

The subscriber sends an `ANNOUNCE_CANCEL` control message to
indicate it will stop sending new subscriptions for tracks
within the provided Track Namespace.

If a publisher receives new subscriptions for that namespace after
receiving an ANNOUNCE_CANCEL, it SHOULD close the session as a
'Protocol Violation'.

~~~
ANNOUNCE_CANCEL Message {
  Track Namespace (b),
}
~~~
{: #moq-transport-announce-cancel-format title="MOQT ANNOUNCE_CANCEL Message"}

* Track Namespace: Identifies a track's namespace as defined in
({{track-name}}).

## TRACK_STATUS_REQUEST {#message-track-status-req}

A potential subscriber sends a 'TRACK_STATUS_REQUEST' message on the control
stream to obtain information about the current status of a given track.

A TRACK_STATUS message MUST be sent in response to each TRACK_STATUS_REQUEST.

~~~
TRACK_STATUS_REQUEST Message {
  Track Namespace (b),
  Track Name (b),
}
~~~
{: #moq-track-status-request-format title="MOQT TRACK_STATUS_REQUEST Message"}



## SUBSCRIBE_OK {#message-subscribe-ok}

A publisher sends a SUBSCRIBE_OK control message for successful
subscriptions.

~~~
SUBSCRIBE_OK
{
  Subscribe ID (i),
  Expires (i),
  Group Order (8),
  ContentExists (f),
  [Largest Group ID (i)],
  [Largest Object ID (i)],
  Number of Parameters (i),
  Subscribe Parameters (..) ...
}
~~~
{: #moq-transport-subscribe-ok format title="MOQT SUBSCRIBE_OK Message"}

* Subscribe ID: Subscription Identifier as defined in {{message-subscribe-req}}.

* Expires: Time in milliseconds after which the subscription is no
longer valid. A value of 0 indicates that the subscription does not expire
or expires at an unknown time.  Expires is advisory and a subscription can
end prior to the expiry time or last longer.

* Group Order: Indicates the subscription will be delivered in
Ascending (0x1) or Descending (0x2) order by group. See {{priorities}}.
Values of 0x0 and those larger than 0x2 are a protocol error.

* ContentExists: 1 if an object has been published on this track, 0 if not.
If 0, then the Largest Group ID and Largest Object ID fields will not be
present.

* Largest Group ID: The largest Group ID available for this track. This field
is only present if ContentExists has a value of 1.

* Largest Object ID: The largest Object ID available within the largest Group ID
for this track. This field is only present if ContentExists has a value of 1.

* Subscribe Parameters: The parameters are defined in {{version-specific-params}}.

## SUBSCRIBE_ERROR {#message-subscribe-error}

A publisher sends a SUBSCRIBE_ERROR control message in response to a
failed SUBSCRIBE.

~~~
SUBSCRIBE_ERROR
{
  Subscribe ID (i),
  Error Code (i),
  Reason Phrase (b),
  Track Alias (i),
}
~~~
{: #moq-transport-subscribe-error format title="MOQT SUBSCRIBE_ERROR Message"}

* Subscribe ID: Subscription Identifier as defined in {{message-subscribe-req}}.

* Error Code: Identifies an integer error code for subscription failure.

* Reason Phrase: Provides the reason for subscription error.

* Track Alias: When Error Code is 'Retry Track Alias', the subscriber SHOULD re-issue the
  SUBSCRIBE with this Track Alias instead. If this Track Alias is already in use,
  the subscriber MUST close the connection with a Duplicate Track Alias error
  ({{session-termination}}).


## SUBSCRIBE_DONE {#message-subscribe-done}

A publisher sends a `SUBSCRIBE_DONE` message to indicate it is done publishing
Objects for that subscription.  The Status Code indicates why the subscription ended,
and whether it was an error.

The format of `SUBSCRIBE_DONE` is as follows:

~~~
SUBSCRIBE_DONE Message {
  Subscribe ID (i),
  Status Code (i),
  Reason Phrase (b),
  ContentExists (f),
  [Final Group (i)],
  [Final Object (i)],
}
~~~
{: #moq-transport-subscribe-fin-format title="MOQT SUBSCRIBE_DONE Message"}

* Subscribe ID: Subscription identifier as defined in {{message-subscribe-req}}.

* Status Code: An integer status code indicating why the subscription ended.

* Reason Phrase: Provides the reason for subscription error.

* ContentExists: 1 if an object has been published for this subscription, 0 if
not. If 0, then the Final Group and Final Object fields will not be present.

* Final Group: The largest Group ID sent by the publisher in an OBJECT
message in this track.

* Final Object: The largest Object ID sent by the publisher in an OBJECT
message in the `Final Group` for this track.

## MAX_SUBSCRIBE_ID {#message-max-subscribe-id}

A publisher sends a MAX_SUBSCRIBE_ID message to increase the number of
subscriptions a subscriber can create within a session.

The Maximum Subscribe Id MUST only increase within a session, and
receipt of a MAX_SUBSCRIBE_ID message with an equal or smaller Subscribe ID
value is a 'Protocol Violation'.

~~~
MAX_SUBSCRIBE_ID
{
  Subscribe ID (i),
}
~~~
{: #moq-transport-max-subscribe-id format title="MOQT MAX_SUBSCRIBE_ID Message"}

* Subscribe ID: The new Maximum Subscribe ID for the session. If a Subscribe ID
equal or larger than this is received in any message, including SUBSCRIBE,
the publisher MUST close the session with an error of 'Too Many Subscribes'.
More on Subscribe ID in {{message-subscribe-req}}.


## ANNOUNCE {#message-announce}

The publisher sends the ANNOUNCE control message to advertise where the
receiver can route SUBSCRIBEs for tracks within the announced
Track Namespace. The receiver verifies the publisher is authorized to
publish tracks under this namespace.

~~~
ANNOUNCE Message {
  Track Namespace (b),
  Number of Parameters (i),
  Parameters (..) ...,
}
~~~
{: #moq-transport-announce-format title="MOQT ANNOUNCE Message"}

* Track Namespace: Identifies a track's namespace as defined in
({{track-name}})

* Parameters: The parameters are defined in {{version-specific-params}}.


## UNANNOUNCE {#message-unannounce}

The publisher sends the `UNANNOUNCE` control message to indicate
its intent to stop serving new subscriptions for tracks
within the provided Track Namespace.

~~~
UNANNOUNCE Message {
  Track Namespace (b),
}
~~~
{: #moq-transport-unannounce-format title="MOQT UNANNOUNCE Message"}

* Track Namespace: Identifies a track's namespace as defined in
({{track-name}}).


## TRACK_STATUS {#message-track-status}

A publisher sends a 'TRACK_STATUS' message on the control stream in response
to a TRACK_STATUS_REQUEST message.

~~~
TRACK_STATUS Message {
  Track Namespace (b),
  Track Name (b),
  Status Code (i),
  Last Group ID (i),
  Last Object ID (i),
}
~~~
{: #moq-track-status-format title="MOQT TRACK_STATUS Message"}

The 'Status Code' field provides additional information about the status of the
track. It MUST hold one of the following values. Any other value is a malformed
message.

0x00: The track is in progress, and subsequent fields contain the highest group
and object ID for that track.

0x01: The track does not exist. Subsequent fields MUST be zero, and any other
value is a malformed message.

0x02: The track has not yet begun. Subsequent fields MUST be zero. Any other
value is a malformed message.

0x03: The track has finished, so there is no "live edge." Subsequent fields
contain the highest Group and object ID known.

0x04: The publisher is a relay that cannot obtain the current track status from
upstream. Subsequent fields contain the largest group and object ID known.

Any other value in the Status Code field is a malformed message.

When a relay is subscribed to a track, it can simply return the highest group
and object ID it has observed, whether or not that object was cached or
completely delivered. If not subscribed, a relay SHOULD send a
TRACK_STATUS_REQUEST upstream to obtain updated information.

Alternatively, the relay MAY subscribe to the track to obtain the same
information.

If a relay cannot or will not do either, it should return its best available
information with status code 0x04.

The receiver of multiple TRACK_STATUS messages for a track uses the information
from the latest arriving message, as they are delivered in order on a single
stream.


# Data Streams {#data-streams}

A publisher sends Objects matching a subscription/fetch on Data Streams.

All MOQT data streams, as well as all datagrams, start with a
variable-length integer indicating the type of the stream in question.

|-------|-----------------------------------------------------|
| ID    | Stream Type                                         |
|------:|:----------------------------------------------------|
| 0x1   | OBJECT_DATAGRAM ({{object-datagram}})               |
|-------|-----------------------------------------------------|
| 0x50  | STREAM_HEADER_TRACK ({{stream-header-track}})       |
|-------|-----------------------------------------------------|
| 0x52  | STREAM_HEADER_PEEP  ({{stream-header-peep}})        |
|-------|-----------------------------------------------------|
| 0x54  | FETCH ({{fetch}})
|-------|-----------------------------------------------------|


An endpoint that receives an unknown stream type MUST close the session.

Every Track has a single 'Object Forwarding Preference' and the Original
Publisher MUST NOT mix different forwarding preferences within a single track.
If a subscriber receives different forwarding preferences for a track, it
SHOULD close the session with an error of 'Protocol Violation'.

## Object Headers {#message-object}

An OBJECT message contains a range of contiguous bytes from from the
specified track, as well as associated metadata required to deliver,
cache, and forward it.  Objects are sent by publishers.

### Canonical Object Fields {#object-fields}

A canonical MoQ Object has the following information:

* Track Namespace and Track Name: The track this object belongs to.

* Group ID: The object is a member of the indicated group ID
{{model-group}} within the track.

* Object ID: The order of the object within the group.  The
IDs starts at 0, increasing sequentially for each object within the
group.

* Publisher Priority: An 8 bit integer indicating the publisher's priority for
the Object {{priorities}}.

* Object Forwarding Preference: An enumeration indicating how a publisher sends
an object. The preferences are Track, Peep, and Datagram.  An Object
MUST be sent according to its `Object Forwarding Preference`, described below.

* Peep ID: The object is a member of the indicated peep ID ({{model-peep}})
within the group. This field is omitted if the Object Forwarding Preference is
Track or Datagram.

* Object Status: As enumeration used to indicate missing
objects or mark the end of a group or track. See {{object-status}} below.

* Object Payload: An opaque payload intended for an End Subscriber and SHOULD
NOT be processed by a relay. Only present when 'Object Status' is Normal (0x0).

#### Object Status {#object-status}

The Object Status informs subscribers what objects will not be received
because they were never produced, are no longer available, or because they
are beyond the end of a group or track.

`Status` can have following values:

* 0x0 := Normal object. The payload is array of bytes and can be empty.

* 0x1 := Indicates Object does not exist. Indicates that this object
         does not exist at any publisher and it will not be published in
         the future. This SHOULD be cached.

* 0x3 := Indicates end of Group. ObjectId is one greater that the
         largest object produced in the group identified by the
         GroupID. This is sent right after the last object in the
         group. If the ObjectID is 0, it indicates there are no Objects
         in this Group. This SHOULD be cached. A publisher MAY use an end of
         Group object to signal the end of all open Peeps in a Group.

* 0x4 := Indicates end of Track and Group. GroupID is one greater than
         the largest group produced in this track and the ObjectId is
         one greater than the largest object produced in that
         group. This is sent right after the last object in the
         track. This SHOULD be cached.

* 0x5 := Indicates end of Peep. Object ID is one greater than the largest
         normal object ID in the Peep.

Any other value SHOULD be treated as a protocol error and terminate the
session with a Protocol Violation ({{session-termination}}).
Any object with a status code other than zero MUST have an empty payload.

Though some status information could be inferred from QUIC stream state,
that information is not reliable and cacheable.

## Object Datagram Message {#object-datagram}

An `OBJECT_DATAGRAM` message carries a single object in a datagram.

An Object received in an `OBJECT_DATAGRAM` message has an `Object
Forwarding Preference` = `Datagram`. To send an Object with `Object
Forwarding Preference` = `Datagram`, determine the length of the fields and
payload and send the Object as datagram. In certain scenarios where the object
size can be larger than maximum datagram size for the session, the Object
will be dropped.

~~~
OBJECT_DATAGRAM Message {
  Subscribe ID (i),
  Track Alias (i),
  Group ID (i),
  Object ID (i),
  Publisher Priority (8),
  Object Payload Length (i),
  [Object Status (i)],
  Object Payload (..),
}
~~~
{: #object-datagram-format title="MOQT OBJECT_DATAGRAM Message"}

## Unidirectional Data Streams

When objects are sent on unidirectional streams, the stream begins with a stream
header message and is followed by one or more sets of serialized object fields.
If a stream ends gracefully in the middle of a serialized Object, terminate the
session with a Protocol Violation.

A publisher SHOULD NOT open more than one stream at a time with the same stream
header message type and fields.


TODO: figure out how a relay closes these streams

### Stream Header Track

When a stream begins with `STREAM_HEADER_TRACK`, all objects on the stream
belong to the track requested in the Subscribe message identified by `Subscribe
ID`.  All objects on the stream have the `Publisher Priority` specified in the
stream header.

~~~
STREAM_HEADER_TRACK Message {
  Subscribe ID (i)
  Track Alias (i),
  Publisher Priority (8),
}
~~~
{: #stream-header-track-format title="MOQT STREAM_HEADER_TRACK Message"}

All Objects received on a stream opened with STREAM_HEADER_TRACK have an `Object
Forwarding Preference` = `Track`.

To send an Object with `Object Forwarding Preference` = `Track`, find the open
stream that is associated with the subscription, or open a new one and send the
`STREAM_HEADER_TRACK` if needed, then serialize the following object fields.
The Object Status field is only sent if the Object Payload Length is zero.

~~~
{
  Group ID (i),
  Object ID (i),
  Object Payload Length (i),
  [Object Status (i)],
  Object Payload (..),
}
~~~
{: #object-track-format title="MOQT Track Stream Object Fields"}

A publisher MUST NOT send an Object on a stream if its Group ID is less than a
previously sent Group ID on that stream, or if its Object ID is less than or
equal to a previously sent Object ID with the same Group ID.

### Stream Header Peep

When a stream begins with `STREAM_HEADER_PEEP`, all objects on the stream
belong to the track requested in the Subscribe message identified by `Subscribe
ID` and the peep indicated by 'Group ID' and `Peep ID`.

~~~
STREAM_HEADER_PEEP Message {
  Subscribe ID (i),
  Track Alias (i),
  Group ID (i),
  Peep ID (i),
  Publisher Priority (8),
}
~~~
{: #stream-header-peep-format title="MOQT STREAM_HEADER_PEEP Message"}

All Objects received on a stream opened with `STREAM_HEADER_PEEP` have an
`Object Forwarding Preference` = `Peep`.

To send an Object with `Object Forwarding Preference` = `Peep`, find the open
stream that is associated with the subscription, `Group ID` and `Peep ID`,
or open a new one and send the `STREAM_HEADER_PEEP`. Then serialize the
following fields.

The Object Status field is only sent if the Object Payload Length is zero.

~~~
{
  Object ID (i),
  Object Payload Length (i),
  [Object Status (i)],
  Object Payload (..),
}
~~~
{: #object-group-format title="MOQT Group Stream Object Fields"}

A publisher MUST NOT send an Object on a stream if its Object ID is less than a
previously sent Object ID within a given group in that stream.

## Examples

Sending a track on one stream:

~~~
STREAM_HEADER_TRACK {
  Subscribe ID = 1
  Track Alias = 1
  Publisher Priority = 0
}
{
  Group ID = 0
  Object ID = 0
  Object Payload Length = 4
  Payload = "abcd"
}
{
  Group ID = 1
  Object ID = 0
  Object Payload Length = 4
  Payload = "efgh"
}
~~~

Sending a peep on one stream:

~~~
Stream = 2

STREAM_HEADER_PEEP {
  Subscribe ID = 2
  Track Alias = 2
  Group ID = 0
  Peep ID = 0
  Publisher Priority = 0
}
{
  Object ID = 0
  Object Payload Length = 4
  Payload = "abcd"
}
{
  Object ID = 1
  Object Payload Length = 4
  Payload = "efgh"
}
~~~


## Fetch

Subscribers issue fetch requests for prior objects while the subscriptions are made for the live edge. Fetch allows objects to be delivered at a requested rate and the fetched objects are delivered on the same stream as the request.

A subscriber issues a FETCH over a new bidirectional QUIC stream to a publisher to request objects from a closed range of groups within a track. Publisher responds to a FETCH request with either a FETCH_ERROR message or data corresponding to the range requested, over the same stream as the request. The object forwarding preference will not apply to fetches.

The format of FETCH is as follows:

~~~
FETCH Message {
  Track Namespace (b),
  Track Name (b),
  Priority (8),
  StartGroup (i),
  StartObject (i),
  EndGroup (i),
  EndObject (i),
  Delivery Rate (i),
  Fetch Upstream (f),
}
~~~
{: #moq-transport-fetch-format title="MOQT FETCH Message"}

* Track Namespace: Identifies the namespace of the track as defined in
({{track-name}}).

* Track Name: Identifies the track name as defined in ({{track-name}}).

* Priority: Specifies the priority of a fetch request relative to
other subscriptions or fetches in the same session. Lower numbers get higher priority. See {{priorities}}. It is typical of certain implementations to treat fetches to have lower priority than subscriptions to the live data.

* StartGroup: The start Group ID of the requested range.

* StartObject: The start Object ID of the request range.

* EndGroup: The end Group ID of the requested range.

* EndObject: The end Object ID, plus 1 of the request range. A value of 0 means the entire group is requested.

* Delivery Rate:  An integer expressing the bitrate in number of bits per second specified in the FETCH message. If present, the publisher
MUST attempt to delivery the objects at the rate requested or return FETCH_ERROR message with error code of "RATE_ERROR". If 0,
delivery rate informed by the underlying transport is choosen by the publisher.

* Fetch Upstream: An boolean expressing subscribers expectation of the publisher to fullfil a fetch reqeust. A value of true indicates the publisher SHOULD make upstream request for any requested objects not in its cache before responding to the fetch request. Otherwise, publisher SHOULD respond with the objects in its cache from the requested range.


Fetch specifies a closed subscription starting at StartObject in StartGroup and ending at EndObject in EndGroup. The start and end of the range are inclusive. EndGroup and EndObject MUST specify the same or a later object than StartGroup and StartObject. If StartGroup/EndGroup is greater than the latest group at the publisher, the publisher MUST return FETCH_ERROR with INVALID_RANGE as the error code and close the stream. Otherwise the publisher responds with objects from the requested range over the same stream as the FETCH request. The behavior at the publisher is controlled by the `Fetch Upstream` fiel and the rate at which the objects are delivered to the subscriber can be further controlled by the `Delivery Rate` field.

# Security Considerations {#security}

TODO: Expand this section, including subscriptions.

## Resource Exhaustion

Live content requires significant bandwidth and resources.  Failure to
set limits will quickly cause resource exhaustion.

MOQT uses stream limits and flow control to impose resource limits at
the network layer.  Endpoints SHOULD set flow control limits based on the
anticipated bitrate.

Endpoints MAY impose a MAX STREAM count limit which would restrict the
number of concurrent streams which a MOQT Streaming Format could have in
flight.

The publisher prioritizes and transmits streams out of order.  Streams
might be starved indefinitely during congestion.  The publisher and
subscriber MUST cancel a stream, preferably the lowest priority, after
reaching a resource limit.


# IANA Considerations {#iana}

TODO: fill out currently missing registries:

* MOQT version numbers
* Setup parameters
* Subscribe parameters
* Subscribe Error codes
* Announce Error codes
* Message types

TODO: register the URI scheme and the ALPN

# Contributors
{:numbered="false"}

- Alan Frindell
- Ali Begen
- Charles Krasic
- Christian Huitema
- Cullen Jennings
- James Hurley
- Jordi Cenzano
- Mike English
- Mo Zanaty
- Will Law
