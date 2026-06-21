---
title: "MOCHA Chat: Text Messaging over MoQ Transport"
abbrev: "mocha-chat"
category: std

docname: draft-jennings-moq-mocha-chat-latest
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
  JSON: RFC8259

informative:
  MIMI-CONTENT:
    title: "More Instant Messaging Interoperability (MIMI) Content Format"
    date: 2024
    author:
    - fullname: R. Mahy
    target: https://datatracker.ietf.org/doc/draft-ietf-mimi-content/

--- abstract

This document specifies the text messaging functionality for MOCHA
(MoQ Open Communication & Hosting Architecture). It defines how
participants send and receive text messages in channels using
MoQ Transport (MOQT) publish/subscribe primitives. Each device
publishes messages on its own track within a channel namespace,
enabling decentralized message production with relay-based
fan-out. This specification covers message naming, format,
causal ordering, delivery, roster management, and channel
discovery for text-based chat.

--- middle

# Introduction

Text messaging is the most fundamental operation in any
communication system. This document specifies how MOCHA enables
real-time text exchange between participants in channels using
MOQT's publish/subscribe model.

MOCHA Chat assigns each publishing device its own MOQT track
within a channel namespace. Messages are grouped by NTP time
in minutes, enabling straightforward time-based history
retrieval and relay caching. Causal ordering is maintained
through a previous-message linkage that forms a DAG across
the per-device tracks in a channel.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "SHOULD", and "MAY" in this
document are to be interpreted as described in BCP 14
{{!RFC2119}} {{!RFC8174}}.

This document uses terminology and architecture defined in
{{MOCHA-ARCH}}, including Provider, Organization, Team, Channel,
Device, DevID, GroupID, and ObjectID. The MOQT transport
primitives (Track, Group, Object, Namespace, Relay) are defined
in {{MoQTransport}}.

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

DevID:
: A unique identifier for a publishing device. Each device that
  can send messages has its own DevID and publishes on its own
  track.

MsgID:
: A globally unique identifier for a message, formed by hashing
  the full track name concatenated with the Group ID and
  Object ID.

# MoQ Chat Naming {#chat-naming}

## Channel Namespace

The MOQT tracks that carry chat messages use a namespace of:

~~~
  mocha-v1-<Provider>-<OrgID>-chat-<TeamID>-<ChannelID>
~~~

For example, messages in the "general" channel of team
"engineering" at organization "acme" hosted by provider
"example.com":

~~~
  mocha-v1-example.com-acme-chat-engineering-general
~~~

## Track Names

Each device publishes messages on its own track within the
channel namespace:

~~~
  msg-v1-<DevID>
~~~

For example, if Alice's device has DevID "alice-laptop-1":

~~~
  Namespace: mocha-v1-example.com-acme-chat-engineering-general
  Track Name: msg-v1-alice-laptop-1
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
  Namespace: mocha-v1-example.com-acme-chat-engineering-general
    |
    +-- Track: msg-v1-alice-laptop-1
    |   +-- Group 29598720 (minute: 2024-05-15 16:00 UTC)
    |   |   +-- Object 0: "Hello everyone"
    |   |   +-- Object 1: "Ready for standup?"
    |   |
    |   +-- Group 29598721 (minute: 2024-05-15 16:01 UTC)
    |       +-- Object 0: "Let's start"
    |
    +-- Track: msg-v1-bob-desktop-1
    |   +-- Group 29598720
    |   |   +-- Object 0: "Hey Alice!"
    |   |
    |   +-- Group 29598721
    |       +-- Object 0: "Sounds good"
    |       +-- Object 1: "Sharing my screen now"
    |
    +-- Track: msg-v1-carol-phone-1
        +-- Group 29598721
            +-- Object 0: "On my way, 2 min"
~~~

# Message Format {#message-format}

## Encoding

Messages are encoded as JSON {{JSON}} and compressed before
being placed in the MOQT object payload. Implementations
MUST support gzip compression and MAY support brotli. The
compression algorithm in use is signaled via the track's
metadata.

Compression reduces bandwidth while maintaining
human-readability of the wire format for debugging (by
decompressing captured payloads).

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
  "expires": 0,
  "ext": {}
}
~~~

The message fields are defined as follows:

### Top-Level Fields

msgId (REQUIRED):
: The globally unique identifier for this message, hex-encoded.
  Computed as defined in {{msg-id-computation}}.

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
  the DevID in the track name from which the message
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

The type enum is extensible. Future specifications define
additional values for media, reactions, and other content.

## MsgID Computation {#msg-id-computation}

The MsgID for a message is formed by computing the SHA-256
hash of the concatenation of:

1. The full track name (namespace + "/" + track name)
2. The Group ID (encoded as a big-endian 64-bit unsigned integer)
3. The Object ID (encoded as a big-endian 64-bit unsigned integer)

~~~
  MsgID = SHA-256(
    full_track_name ||
    big_endian_uint64(group_id) ||
    big_endian_uint64(object_id)
  )
~~~

For example:

~~~
  full_track_name = "mocha-v1-example.com-acme-chat-engineering-general/msg-v1-alice-laptop-1"
  group_id = 29598720
  object_id = 0

  MsgID = SHA-256(full_track_name || 0x0000000001C3E400 || 0x0000000000000000)
~~~

This construction ensures MsgIDs are globally unique and
deterministically computable by any participant that knows
the track name and position. The hex encoding of the MsgID
in JSON uses lowercase hexadecimal characters.

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
    "devId": "alice-laptop-1"
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

# Sending Messages {#sending}

## Publish Flow

To send a message, a device publishes a MOQT object to its
own track within the channel namespace:

1. Device determines the current Group ID from NTP time
   in minutes.
2. Device assigns the next Object ID for this group on its
   track.
3. Device computes the MsgID from the track name, group, and
   object.
4. Device records the MsgID(s) of the most recent message(s)
   it has received in this channel as the previous list.
5. Device constructs the ChatMessage, serializes as JSON,
   compresses, and publishes as a MOQT object.

~~~
  Alice                     Relay
    |                         |
    |  PUBLISH object         |
    |  Namespace: mocha-v1-example.com-acme-chat-eng-general
    |  Track: msg-v1-alice-laptop-1
    |  Group: 29598720        |
    |  Object: 0              |
    |  [compressed JSON ChatMessage]
    |------------------------>|
    |                         |
    |                         |  (relay caches and fans out
    |                         |   to subscribers of this track)
    |                         |
~~~

## Authorization

A device MUST hold a valid authorization token granting
publish access to its track within the channel namespace
before publishing. Authorization mechanisms are defined
in {{MOCHA}}.

# Receiving Messages {#receiving}

## Subscribe Flow

To receive messages in a channel, a participant subscribes
to the message tracks of all active devices in the channel.
Track discovery is accomplished via the roster (see
{{roster-naming}}) or SUBSCRIBE_NAMESPACE:

~~~
  Bob                       Relay
    |                         |
    |  SUBSCRIBE_NAMESPACE    |
    |  Prefix: mocha-v1-example.com-acme-chat-eng-general
    |------------------------>|
    |                         |
    |  (Relay publishes matching tracks:)
    |  msg-v1-alice-laptop-1  |
    |  msg-v1-carol-phone-1   |
    |<------------------------|
    |                         |
    |  (Client accepts tracks)|
    |                         |
    |  OBJECTs arrive from    |
    |  all device tracks      |
    |<------------------------|
    |                         |
~~~

Alternatively, if the roster provides full track names,
a participant can SUBSCRIBE to individual tracks directly.

## Late Join

When a participant subscribes to a channel, it uses MOQT's
Joining FETCH to retrieve recent groups (minutes) from the
relay's cache for each device track. This provides message
history for context.

## Deduplication

Participants MUST deduplicate messages based on MsgID. The
same message may be received multiple times due to relay
topology or reconnection.

## Channel Reconstruction

To display a unified channel view, a participant:

1. Receives messages from all device tracks in the channel.
2. Computes or verifies each message's MsgID.
3. Builds the causal DAG from the previous fields.
4. Linearizes messages for display (see {{causal-ordering}}).

# MoQ Roster Naming {#roster-naming}

## Roster Namespace

The roster tracks for a channel use the namespace:

~~~
  mocha-v1-<Provider>-<OrgID>-roster-<TeamID>-<ChannelID>
~~~

For example:

~~~
  mocha-v1-example.com-acme-roster-engineering-general
~~~

## Roster Track Names

Each device publishes its user information on a track:

~~~
  user-<DevID>
~~~

For example:

~~~
  Namespace: mocha-v1-example.com-acme-roster-engineering-general
  Track Name: user-alice-laptop-1
~~~

## Roster Object Format

Each object on a roster track contains a JSON document
providing information about the mapping of the device to
a user in the identity system:

~~~json
{
  "user_id": "alice@acme.example.com",
  "display_name": "Alice Smith",
  "dev_id": "alice-laptop-1",
  "joined_at": 1715788000000,
  "status": "active"
}
~~~

## Roster Usage

Participants subscribe to the roster namespace to discover
which devices are active in a channel. The roster provides
the set of msg-v1-\<DevID\> tracks that should be subscribed
for receiving channel messages.

When a new device joins a channel, it publishes a roster
object. When a device leaves, it publishes an updated roster
object with status "departed" or the track simply becomes
inactive.

# Channel Discovery {#channel-discovery}

## Channel Info Namespace

Channel discovery tracks use the namespace:

~~~
  mocha-v1-<Provider>-<OrgID>-channel_info-<TeamID>
~~~

For example:

~~~
  mocha-v1-example.com-acme-channel_info-engineering
~~~

## Channel Info Track Names

Each device (or server) that creates or manages channels
publishes channel information on a track:

~~~
  channel_<DevID>
~~~

For example:

~~~
  Namespace: mocha-v1-example.com-acme-channel_info-engineering
  Track Name: channel_admin-server-1
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

## Channel Discovery Flow

A participant discovers available channels by subscribing to
the channel info namespace for their team:

~~~
  Client                    Relay
    |                         |
    |  SUBSCRIBE_NAMESPACE    |
    |  Prefix: mocha-v1-example.com-acme-channel_info-engineering
    |------------------------>|
    |                         |
    |  (Relay announces channel info tracks)
    |<------------------------|
    |                         |
    |  SUBSCRIBE channel info track
    |------------------------>|
    |                         |
    |  OBJECT: channel list   |
    |<------------------------|
    |                         |
    |  (Client displays available channels to user)
    |                         |
~~~

# Delivery Properties {#delivery}

## Reliability

Text messages MUST be delivered reliably using QUIC streams.
Messages use MOQT's stream-based delivery to ensure no
message loss.

## Fan-Out

When multiple participants subscribe to the same device
track, the relay performs fan-out delivery. Each published
message is delivered to all subscribers of that track.

## Caching

Relays MUST cache published message objects for the duration
of their configured cache retention. Cached objects serve
late-join (Joining FETCH) and history retrieval. Since groups
correspond to NTP minutes, a relay caching the last N groups
provides approximately N minutes of history.

# Security Considerations

## Transport Security

All message delivery MUST use TLS 1.3 via QUIC or
WebTransport, providing hop-by-hop encryption between
participants and relays.

## Authorization

Devices MUST be authorized before publishing or subscribing
to tracks. Authorization is enforced at the relay as
specified in {{MOCHA}}.

## Message Integrity

The MsgID construction (hash of track name + position) allows
any recipient to verify the claimed identity of a message.
Participants SHOULD verify that the sender's dev_id in the
message matches the track name from which the message was
received.

## Content Visibility

Without end-to-end encryption, relays can observe message
content. Deployments requiring confidentiality MUST enable
E2EE as defined in {{MOCHA}}.

# IANA Considerations

This document has no IANA actions.

--- back
