---
title: "MOCHA Chat: Messaging over MoQ Transport"
abbrev: "mocha-chat"
category: std

docname: draft-jennings-moq-mocha-chat-00
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Media Over QUIC"
keyword:
 - MoQ
 - MOQT
 - MOCHA
 - Chat
 - Messaging
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"

author:
  -
    fullname: Cullen Jennings
    organization: Cisco
    email: fluffy@iii.ca

  -
    fullname: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:
  MoQTransport: I-D.draft-ietf-moq-transport
  MOCHA: I-D.draft-jennings-moq-mocha
  MOCHA-ARCH: I-D.draft-jennings-moq-mocha-arch
  MOCHA-IDENTITY: I-D.draft-jennings-moq-mocha-identity
  JSON: RFC8259

  RFC1952:
  RFC7932:
  RFC8126:

informative:
  MIMI-CONTENT:
    title: "More Instant Messaging Interoperability (MIMI) Content Format"
    date: 2024
    author:
    - fullname: R. Mahy
    target: https://datatracker.ietf.org/doc/draft-ietf-mimi-content/

--- abstract

This document specifies the messaging functionality for MOCHA
(MoQ Open Communication & Hosting Architecture). It defines how
participants send and receive messages in channels using
MoQ Transport (MOQT) publish/subscribe primitives. Each device
publishes messages on its own track within a channel namespace,
enabling decentralized message production with relay-based
fan-out. This specification covers message naming, format,
causal ordering, delivery, roster management, and channel
discovery for text-based chat.

--- middle

# Introduction

Messaging is the most fundamental operation in any
communication system. This document specifies how MOCHA enables
real-time message exchange between participants in channels using
MOQT's publish/subscribe model.

MOCHA Chat assigns each publishing device its own MOQT track
within a channel namespace. Messages are grouped by NTP time
in minutes, enabling straightforward time-based history
retrieval and relay caching. Causal ordering is maintained
through a previous-message linkage that forms a DAG across
the per-device tracks in a channel.


# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology and architecture defined in
{{MOCHA-ARCH}}, including Provider, Organization, Team, Channel,
Device, GroupID, and ObjectID. The MOQT transport
primitives (Track, Group, Object, Namespace, Relay) are defined
in {{MoQTransport}}. HClientID and HDevID are defined in
{{MOCHA-IDENTITY}}.

The following additional terms are used:

Provider:
: The service provider operating the MOCHA deployment. Identified
  by a domain name or similar unique string.

OrgID:
: The organization identifier within a provider's deployment.

TeamID:
: An identifier for a team or workspace within an organization.

ChannelID:
: An identifier for a specific channel within a team.

MsgID:
: A globally unique identifier for a message. Implementations
  SHOULD use a UUID (version 4) to generate MsgIDs.

# MOCHA Chat Naming {#chat-naming}

## Identifier Derivation {#id-derivation}

The HClientID and HDevID used in namespace tuples and
track names are derived per {{MOCHA-IDENTITY}},
Section "Identifier Derivation".

## Channel Namespace

The MOQT tracks that carry chat messages use a namespace
tuple with the following fields:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "chat", <TeamID>,
   <ChannelID>, <HClientID>)
~~~

For example, messages in the "general" channel of team
"engineering" at organization "acme" hosted by provider
"example.com":

~~~
  ("mocha_v1", "example.com", "acme", "chat", "engineering",
   "general", <HClientID>)
~~~

## Track Names

Each device publishes messages on its own track within the
channel namespace. The track name is a single byte sequence:

~~~
  msg_v1_<HDevID>
~~~

For example:

~~~
  Namespace: ("mocha_v1", "example.com", "acme", "chat",
              "engineering", "general", <HClientID>)
  Track Name: "msg_v1_<HDevID>"
~~~

This per-device track design allows each device to publish
independently without coordinating Group IDs with other
publishers.

## Group Semantics

Groups are NTP time in minutes. The Group ID corresponds
to the NTP timestamp (in minutes) at the time the message
is published:

~~~
  group_id = floor(ntp_timestamp_seconds / 60)
~~~

This provides:

* Natural time-based partitioning of messages.
* Straightforward mapping from wall-clock time to Group ID
  for history retrieval.
* One-minute granularity balances caching efficiency with
  latency.

## Object Semantics

Within each group, messages are assigned sequentially
increasing Object IDs starting at 0. Each message occupies
exactly one MOQT object.

## MOQT Object Mapping

~~~
  Namespace: ("mocha_v1", "example.com", "acme", "chat",
              "engineering", "general")
    |
    +-- Track: msg_v1_alice-laptop-1
    |   +-- Group 29598720 (minute: 2024-05-15 16:00 UTC)
    |   |   +-- Object 0: "Hello everyone"
    |   |   +-- Object 1: "Ready for standup?"
    |   |
    |   +-- Group 29598721 (minute: 2024-05-15 16:01 UTC)
    |       +-- Object 0: "Let's start"
    |
    +-- Track: msg_v1_bob-desktop-1
    |   +-- Group 29598720
    |   |   +-- Object 0: "Hey Alice!"
    |   |
    |   +-- Group 29598721
    |       +-- Object 0: "Sounds good"
    |       +-- Object 1: "Sharing my screen now"
    |
    +-- Track: msg_v1_carol-phone-1
        +-- Group 29598721
            +-- Object 0: "On my way, 2 min"
~~~

# Message Format {#message-format}

## Encoding

Messages are encoded as JSON {{JSON}} and compressed before
being placed in the MOQT object payload. The compression
algorithm is signaled via the MOCHA_COMPRESSION track
property (see {{compression-property}}).

## Compression Track Property {#compression-property}

The MOCHA_COMPRESSION track property signals the compression
algorithm applied to ALL objects on a message track. Publishers
MUST include this property in the PUBLISH message
(publisher-initiated flow) or SUBSCRIBE_OK
(subscriber-initiated flow).

Subscribers MUST check for this property before processing
track payloads. If the property is absent, the subscriber
MUST treat the payload as uncompressed. If the property is
present with a value the subscriber does not support, the
subscriber MUST NOT attempt to process the payload and SHOULD
unsubscribe from the track.

The following compression algorithm values are defined:

| Value | Algorithm     | Reference   |
|:===== |:============= |:=========== |
| 0     | None          | This document |
| 1     | GZIP          | {{!RFC1952}} |
| 2     | Brotli        | {{!RFC7932}} |

Implementations MUST support GZIP (value 1) and MAY support
Brotli (value 2). A publisher that does not compress MUST
either set the property to 0 or omit it entirely.

## Message Structure

The message structure follows the MIMI content format style
{{MIMI-CONTENT}}, adapted for MOCHA's channel-based messaging.
Fields use short keys for compact JSON encoding.

~~~json
{
  "msgId": "<hex-encoded MsgID>",
  "type": 0,
  "ts": 1715788800000,
  "sender": {
    "userId": "<user identifier>",
    "displayName": "<human-readable name>",
    "devId": "<device identifier>"
  },
  "content": {
    "type": "text/plain",
    "body": "<message text>"
  },
  "prev": ["<hex-encoded MsgID>", ...],
  "threadId": "<hex-encoded MsgID>",
  "replaces": "<hex-encoded MsgID>",
  "expires": 0,
  "ext": {}
}
~~~

The message fields are defined as follows:

### Top-Level Fields

msgId (REQUIRED):
: The globally unique identifier for this message within the
  channel. Implementations SHOULD use a UUID (version 4).

type (REQUIRED):
: Integer message type. This specification defines type 0
  (none/system) and type 1 (text). The type field is
  extensible for future message categories.

ts (REQUIRED):
: The time the message was created, in milliseconds since
  the Unix epoch. Set by the publisher's local clock.

sender (REQUIRED):
: The originator of the message. See {{sender-info}}.

content (REQUIRED):
: The message payload. See {{content-structure}}.

prev (REQUIRED):
: A list of hex-encoded MsgIDs of the immediately preceding
  message(s) in the channel. This forms a causal ordering
  DAG across all tracks in the channel. An empty array
  indicates no known predecessors. See {{causal-ordering}}.

threadId (OPTIONAL):
: The hex-encoded MsgID of the root message that started
  the thread. Present when this message is a reply within
  a thread. Threads are flat — nested threading is not
  supported. If absent, the message is a top-level channel
  message.

replaces (OPTIONAL):
: The hex-encoded MsgID of a prior message that this message
  edits or deletes. Present when type is 2 (edit) or
  3 (delete). See {{message-edit-delete}}.

expires (OPTIONAL):
: Expiration time in milliseconds since the Unix epoch. If
  non-zero, clients SHOULD hide the message after this time.
  A value of 0 indicates no expiration.

ext (OPTIONAL):
: An object containing extension fields. Implementations
  MUST ignore unknown extension keys. Extensions are
  namespaced to avoid collisions (see {{MOCHA}}).

### Sender Info {#sender-info}

The sender object identifies the message author:

userId (REQUIRED):
: A stable identifier for the user within the identity
  system (e.g., "alice@acme.example.com").

displayName (REQUIRED):
: A human-readable label for rendering in the UI.

devId (REQUIRED):
: The identifier of the publishing device. MUST match
  the HDevID in the track name from which the message
  is published.

### Content Structure {#content-structure}

The content object carries the message payload, following
MIMI's content disposition pattern:

type (REQUIRED):
: A MIME type string indicating the content format. This
  specification defines "text/plain" for plain text messages.
  Implementations MUST support "text/plain".

body (REQUIRED):
: The content payload. For "text/plain", this is a UTF-8
  string containing the message text.

Future specifications MAY define additional content types
(e.g., "text/markdown"). Clients receiving an unknown content
type SHOULD render the body as plain text if it is a string,
or display a placeholder indicating an unsupported format.

### Message Type Values

| Value | Name   | Description                          |
|:===== |:====== |:==================================== |
| 0     | none   | System or control message (no body)  |
| 1     | text   | Text message                         |
| 2     | edit   | Replaces content of a prior message  |
| 3     | delete | Removes a prior message from display |

The type enum is extensible. Future specifications define
additional values for media and other content.

## Example Message

A text message as JSON (before compression):

~~~json
{
  "msgId": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "type": 1,
  "ts": 1715788800000,
  "sender": {
    "userId": "alice@acme.example.com",
    "displayName": "Alice Smith",
    "devId": "<HashDevID>"
  },
  "content": {
    "type": "text/plain",
    "body": "Hello everyone, ready for the standup?"
  },
  "prev": ["e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6"]
}
~~~

# Causal Ordering {#causal-ordering}

## Previous Message List

Each message contains a "previous" field listing the MsgID(s)
of the most recent message(s) the publisher has observed in
the channel at the time of publishing. This forms a directed
acyclic graph (DAG) across all per-device tracks in a channel.

~~~
  Causal DAG across devices:

  Alice's track:     A1 -----> A2 ---------> A3
                      \          \           /
  Bob's track:         --> B1 --> B2 ------>
                                   \
  Carol's track:                    --> C1
~~~

In this example:

* A1 has no previous (first message in channel, or references
  messages not shown).
* B1.previous = [A1] (Bob saw Alice's first message).
* A2.previous = [B1] (Alice saw Bob's reply).
* B2.previous = [A2] (Bob saw Alice's second message).
* C1.previous = [B2] (Carol saw Bob's second message).
* A3.previous = [B2, C1] (Alice saw both Bob and Carol's
  latest messages).

## Ordering Semantics

The previous message list provides causal ordering:

* If message B lists message A in its previous list
  (directly or transitively), then A causally precedes B.
* Messages with no causal relationship are concurrent and
  may be displayed in any order (typically by timestamp).
* Clients reconstruct channel ordering by traversing the
  DAG.

## Display Ordering

Clients SHOULD use the causal DAG to determine display
order. When causal ordering does not fully determine order
(concurrent messages), clients SHOULD fall back to timestamp
ordering.

A simple linearization algorithm:

1. Topologically sort messages using the DAG.
2. For concurrent messages (no causal relationship), sort
   by timestamp.
3. For identical timestamps, sort by MsgID (lexicographic
   on the hash bytes) for deterministic ordering.

# Message Edit and Delete {#message-edit-delete}

A sender can edit or delete a message it previously published
by publishing a new object on the same track with the
`replaces` field set to the MsgID of the original message.

## Edit (type 2)

An edit message replaces the content of a prior message.
The `content` field carries the new text. The original
message's position in the causal DAG is preserved; the edit
object itself also participates in the DAG via its own `prev`
field.

~~~json
{
  "msgId": "b2c3d4...",
  "type": 2,
  "ts": 1715788900000,
  "sender": {
    "userId": "alice@acme.example.com",
    "displayName": "Alice Smith",
    "devId": "<HashDevID>"
  },
  "content": {
    "type": "text/plain",
    "body": "Hello everyone, ready for standup? (corrected)"
  },
  "prev": ["a1b2c3..."],
  "replaces": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
}
~~~

## Delete (type 3)

A delete message removes a prior message from display. No
`content` field is required; if present it is ignored.

~~~json
{
  "msgId": "c3d4e5...",
  "type": 3,
  "ts": 1715789000000,
  "sender": {
    "userId": "alice@acme.example.com",
    "displayName": "Alice Smith",
    "devId": "<HashDevID>"
  },
  "prev": ["b2c3d4..."],
  "replaces": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
}
~~~

## Processing Rules

1. Only the original sender (matching userId) MAY edit or
   delete a message. Receivers MUST ignore edit or delete
   objects from a different userId than the original message.

2. Multiple edits to the same message are permitted. The
   edit with the latest causal ordering (per
   {{causal-ordering}}) takes precedence.

3. A delete after an edit takes precedence: the message is
   removed from display regardless of prior edits.

4. Receivers SHOULD display an indicator that a message has
   been edited. Receivers MAY offer access to the edit
   history.

5. Relays forward edit and delete objects like any other
   message object without interpreting their semantics.


# Threading {#threading}

MOCHA supports threaded conversations where replies are
grouped under a parent message. Threads are flat — nested
threading is not supported.

## Thread Replies

To reply within a thread, a device publishes a message on
its regular `msg_v1_<HDevID>` track with the `threadId` field
set to the MsgID of the root message that started the
thread. The root message itself has no `threadId` field.

## Thread Display

Clients SHOULD group messages with the same `threadId` and
display them as a thread under the root message. The root
message is identified by its MsgID matching the `threadId`
of the replies.

## Thread Edits and Deletes

Edit and delete messages (types 2 and 3) within a thread
MUST carry the same `threadId` as the message they replace.
This allows clients to correctly scope the edit or delete
to the thread view.

# MOCHA Roster Naming {#roster-naming}

The roster provides a discovery mechanism for participants
to learn which devices are active in a channel. Each device
publishes its availability and user identity on a dedicated
roster track. Participants subscribe to the roster namespace
to build the set of message tracks they should follow.

## Roster Namespace

The roster tracks for a channel use the namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "roster", <TeamID>,
   <ChannelID>, <HClientID>)
~~~

HClientID and HDevID are derived per {{id-derivation}}.

For example:

~~~
  ("mocha_v1", "example.com", "acme", "roster", "engineering",
   "general", <HClientID>)
~~~

## Roster Track Names

Each device publishes its user information on a track:

~~~
  user_<HDevID>
~~~

For example:

~~~
  Namespace: ("mocha_v1", "example.com", "acme", "roster",
              "engineering", "general", <HClientID>)
  Track Name: "user_<HDevID>"
~~~

## Roster Object Format

Each object on a roster track contains a JSON document
providing information about the mapping of the device to
a user in the identity system. The display_name is obtained
from the user's PAB entry (see {{MOCHA}}) and the user_id
is obtained from the identity system (see {{MOCHA-IDENTITY}}).

~~~json
{
  "user_id": "alice@acme.example.com",
  "display_name": "Alice Smith",
  "dev_id": "<HashDevID>",
  "joined_at": 1715788000000,
  "status": "active"
}
~~~

## Roster Usage

Participants subscribe to the roster namespace to discover
which devices are active in a channel. The roster provides
the set of msg_v1_\<HDevID\> tracks that should be subscribed
for receiving channel messages.

When a new device joins a channel, it publishes a roster
object. When a device leaves, it publishes an updated roster
object with status "departed" or the track simply becomes
inactive.

# Channel Discovery {#channel-discovery}

Channel discovery allows participants to learn which channels
are available within a team. A designated device (or server)
publishes channel metadata on a channel info track. Participants
subscribe to this namespace to present users with a list of
channels they can join.

## Channel Info Namespace

Channel discovery tracks use the namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "channel_info", <TeamID>)
~~~

For example:

~~~
  ("mocha_v1", "example.com", "acme", "channel_info", "engineering")
~~~

## Channel Info Track Names

Each device (or server) that creates or manages channels
publishes channel information on a track:

~~~
  channel_<HDevID>
~~~

For example:

~~~
  Namespace: ("mocha_v1", "example.com", "acme",
              "channel_info", "engineering")
  Track Name: "channel_admin-server-1"
~~~

## Channel Info Object Format

Objects on channel info tracks contain the names and metadata
of channels available in the team:

~~~json
{
  "channels": [
    {
      "channel_id": "general",
      "name": "#general",
      "topic": "General discussion",
      "type": "public"
    },
    {
      "channel_id": "backend",
      "name": "#backend",
      "topic": "Backend engineering",
      "type": "public"
    },
    {
      "channel_id": "secret-project",
      "name": "#secret-project",
      "type": "private"
    }
  ]
}
~~~

# Availability {#availability}

MOCHA provides an availability system for communicating user
availability and status within a channel. Channel membership
implicitly grants availability visibility — all channel members
can see each other's availability without explicit subscription
consent.

## Availability Namespace

Availability tracks for a channel use the namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "availability", <TeamID>,
   <ChannelID>, <HClientID>)
~~~

HClientID is derived per {{id-derivation}}.

## Availability Track Names

Each device publishes its availability on a dedicated track:

~~~
  availability_v1_<HDevID>
~~~

HDevID is derived per {{id-derivation}}.

## Availability States

| State     | Description                              |
|-----------|------------------------------------------|
| online    | User is active and available             |
| away      | User is idle or stepped away             |
| dnd       | Do not disturb — suppress notifications  |
| offline   | User is disconnected                     |

## Availability Object Format

Each object on an availability track contains:

~~~json
{
  "status": "online",
  "status_message": "In meetings until 3pm",
  "last_active": 1715790000000,
  "idle_seconds": 0
}
~~~

Fields:

- **status**: One of the availability states above.
- **status_message**: Optional human-readable status text.
- **last_active**: NTP timestamp (milliseconds) of last
  user activity.
- **idle_seconds**: Seconds since last user interaction on
  this device. Set to 0 when active.

## Availability Updates

A device publishes a new availability object whenever its state
changes (e.g., user becomes idle, changes status, or goes
offline). Each new object replaces the previous state.

Devices SHOULD publish an availability object with status
"offline" when gracefully disconnecting. If a device
disconnects without publishing an offline status, subscribers
detect the absence via track inactivity.

## Typing Indicators {#typing-indicators}

Typing indicators are ephemeral availability signals scoped to
a channel. They use the same availability namespace with a
separate track:

~~~
  typing_v1_<HDevID>
~~~

### Typing Object Format

~~~json
{
  "is_typing": true,
  "thread_id": null,
  "ts": 1715790000000
}
~~~

Typing indicators:

- SHOULD be delivered via QUIC datagrams (unreliable) for
  low latency.
- SHOULD timeout after 5 seconds without an update.
- MAY be disabled per-user or per-channel.

## Subscribing to Availability

Participants subscribe (SUBSCRIBE_TRACKS) to the availability
namespace to receive availability and typing updates from all
channel members. This follows the same pattern as roster
subscription — joining a channel implicitly subscribes
the device to availability.

# Security Considerations

TODO

# IANA Considerations

## MOCHA_COMPRESSION Track Property

This document requests IANA registration of the
MOCHA_COMPRESSION track property in the "MoQ Track Properties"
registry (if established by {{MoQTransport}}).

Property Name:
: MOCHA_COMPRESSION

Property Type:
: Track Property

Value Type:
: Varint

Description:
: Signals the compression algorithm applied to all objects
  on a MOCHA message track.

Reference:
: This document, {{compression-property}}

## MOCHA Compression Algorithms Registry

This document establishes a new "MOCHA Compression Algorithms"
registry with the following initial entries:

| Value | Algorithm | Reference       |
|:===== |:========= |:=============== |
| 0     | None      | This document   |
| 1     | GZIP      | {{!RFC1952}}    |
| 2     | Brotli    | {{!RFC7932}}    |

New entries require Specification Required (per {{!RFC8126}}).

--- back
