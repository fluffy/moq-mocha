---
title: "MOCHA Reactions: Real-Time Reactions over MoQ Transport"
abbrev: "mocha-reactions"
category: std

docname: draft-jennings-moq-mocha-reactions-latest
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
 - Reactions
 - Emoji
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
  MOCHA-CHAT: I-D.draft-jennings-moq-mocha-chat
  JSON: RFC8259


informative:

--- abstract

This document specifies reactions for MOCHA (MoQ Open
Communication & Hosting Architecture). It supports
message-targeted reactions (emoji on a specific chat message)
and channel-level reactions (engagement signals such as
applause or raised hands).

--- middle

# Introduction

This document specifies how MOCHA enables real-time reaction
exchange using MOQT's publish/subscribe model. Two modes
are supported:

Message-Targeted Reactions:
: Emoji responses referencing a specific chat message MsgID.

Channel-Level Reactions:
: Standalone engagement signals (applause, raised hands)
  directed at the channel as a whole.

Both modes use the same track structure. For large-scale
scenarios, MOQT relay filtering (top-N track selection,
property-based object filtering) allows subscribers to
receive a manageable subset of the reaction stream.


# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology defined in {{MOCHA-ARCH}}
and MOQT primitives defined in {{MoQTransport}}. Message
identifiers (MsgID) are defined in {{MOCHA-CHAT}}.


# Reaction Track Naming {#reaction-naming}

## Reactions Namespace

~~~
  ("mocha_v1", <Provider>, <OrgID>, "reactions", <TeamID>, <ChannelID>)
~~~

## Track Names

Each device publishes reactions on its own track:

~~~
  react_v1_<DevID>
~~~

## Object Semantics

Within each group, reactions are assigned sequentially
increasing Object IDs starting at 0. Each reaction occupies
one MOQT object.


# Reaction Format {#reaction-format}

## Encoding

Reactions are encoded as JSON {{JSON}} and compressed per
the MOCHA_COMPRESSION track property defined in
{{MOCHA-CHAT}}.

## Object and Track Properties {#reaction-properties}

Object Properties (set per reaction object):

REACTION_TYPE (Property Type TBD1):
: Integer reaction type (1=emoji, 2=vote, 3=raise,
  4=applause, 5=custom). Enables category filtering.

REACTION_VALUE (Property Type TBD2):
: Integer encoding of the specific value within its type
  (e.g., fire=4, raise=1). Enables value-level filtering.
  Encodings defined in {{value-encoding}}.

Track Property (set per reaction track):

REACTION_ACTIVITY (Property Type TBD3):
: Activity level, updated with each published object.
  Enables top-N track selection at the relay.

## Reaction Value Encoding {#value-encoding}

Each reaction type defines integer encodings for its values:

Emoji (type 1):

| Integer | Value      |
|:======= |:========== |
| 1       | thumbs-up  |
| 2       | thumbs-down|
| 3       | heart      |
| 4       | fire       |
| 5       | laugh      |
| 6       | sad        |
| 7       | wow        |
| 8       | clap       |
| 9       | thinking   |
| 10      | eyes       |

Values 11-127 are reserved for future standard emoji.
Values 128+ are available for application-defined emoji.

Raise (type 3):

| Integer | Value |
|:======= |:===== |
| 1       | raise |
| 2       | lower |

Vote (type 2) and Applause (type 4) value encodings are
application-defined.

## Reaction Structure

~~~json
{
  "type": 1,
  "value": "thumbs-up",
  "ts": 1715788800000,
  "sender": {
    "userId": "alice@acme.example.com",
    "devId": "alice-laptop-1"
  },
  "target": {
    "msgId": "a1b2c3...f0a1b2"
  },
  "ext": {}
}
~~~

### Fields

type (REQUIRED):
: Integer reaction type. See {{reaction-types}}.

value (REQUIRED):
: String identifying the specific reaction (e.g.,
  "thumbs-up", "fire", "raise").

ts (REQUIRED):
: Creation time in milliseconds since Unix epoch.

sender (REQUIRED):
: Object with userId and devId fields.

target (OPTIONAL):
: Object with msgId field (hex-encoded MsgID per
  {{MOCHA-CHAT}}). Present for message-targeted reactions;
  absent for channel-level reactions. {#target-field}

ext (OPTIONAL):
: Extension fields. Implementations MUST ignore unknown keys.

### Reaction Types {#reaction-types}

| Value | Name     | Description                              |
|:===== |:======== |:======================================== |
| 1     | emoji    | Emoji reaction (heart, fire, thumbs-up)  |
| 2     | vote     | Poll or vote response                    |
| 3     | raise    | Raise/lower hand                         |
| 4     | applause | Applause or cheer                        |
| 5     | custom   | Application-defined reaction             |

Extensible via IANA registry. Value encodings per type are
defined in {{value-encoding}}.

## Reaction Removal

To retract a reaction, a device publishes an object with the
same type, value, and target, plus `"remove": true`. The
receiver removes the matching reaction from display.


# Sending Reactions {#sending}

To send a reaction, a device publishes a MOQT object to its
track within the reactions namespace, setting REACTION_TYPE
and REACTION_VALUE object properties and updating the
REACTION_ACTIVITY track property.

~~~
  Alice                     Relay
    |                         |
    |  PUBLISH object         |
    |  Namespace: ("mocha_v1", "example.com", "acme",
    |              "reactions", "engineering", "general")
    |  Track: react_v1_alice-laptop-1
    |  [object properties + compressed JSON reaction]
    |------------------------>|
    |                         |
    |                         |  (relay caches and fans out
    |                         |   to matching subscribers)
    |                         |
~~~

## Rate Limiting

* Message-targeted: At most 1 reaction of a given type per
  message per device (newer replaces older).
* Channel-level: At most 1 reaction per second per device.

Relays MAY enforce rate limits and drop excess objects.


# Receiving Reactions {#receiving}

A participant discovers reaction tracks via
SUBSCRIBE_NAMESPACE on the reactions namespace prefix,
then subscribes to the announced tracks.

## Processing Message-Targeted Reactions

1. Look up the referenced MsgID in the local message store.
2. Display the reaction on that message.
3. Same sender + same type + same target: newer replaces older.
4. If remove=true, remove the matching reaction.

## Processing Channel-Level Reactions

1. Display as a transient signal (e.g., floating emoji,
   applause indicator).
2. For raise-hand: maintain a queue until "lower" is received.

## Late Join

Clients MAY use Joining FETCH to retrieve recent reaction
groups for message-targeted reaction counts. Channel-level
historical signals MAY be ignored.


# Relay Filtering for Large-Scale Reactions {#relay-filtering}

For large-scale scenarios, subscribers use MOQT subscription
filters to receive a manageable subset of the reaction stream.

## Selecting the Most Active Reactors

A presenter subscribes to the reactions namespace with a
Track Filter on REACTION_ACTIVITY, requesting the top-N
most active tracks. The relay dynamically promotes and
demotes tracks as activity shifts.

~~~
  Presenter                    Relay                 Publishers
    |                            |                       |
    |  SUBSCRIBE_NAMESPACE       |                       |
    |  Prefix: ("mocha_v1",...   |                       |
    |    "reactions","eng","gen")|                       |
    |  Track Filter:             |                       |
    |    Property=REACTION_ACTIVITY
    |    MaxTracks=N             |                       |
    |--------------------------->|                       |
    |                            |                       |
    |                            |    Objects arrive from
    |                            |    1000s of publishers
    |                            |<----------------------|
    |                            |                       |
    |                            |  (Relay ranks tracks  |
    |                            |   by REACTION_ACTIVITY|
    |                            |   selects top-N)      |
    |                            |                       |
    |  PUBLISH (track promoted)  |                       |
    |  react_v1_user-042         |                       |
    |<---------------------------|                       |
    |                            |                       |
    |  Objects from top-N tracks |                       |
    |<---------------------------|                       |
    |                            |                       |
    |  (Activity shifts:         |                       |
    |   user-042 demoted,        |                       |
    |   user-789 promoted)       |                       |
    |                            |                       |
    |  REQUEST_UPDATE Fwd=0      |                       |
    |  (react_v1_user-042)       |                       |
    |<---------------------------|                       |
    |                            |                       |
    |  PUBLISH (track promoted)  |                       |
    |  react_v1_user-789         |                       |
    |<---------------------------|                       |
    |                            |                       |
    |  Objects from new top-N    |                       |
    |<---------------------------|                       |
    |                            |                       |
~~~

The presenter client counts reactions locally from the
filtered stream.

## Subscribe to Track with Object Property Filter

A subscriber can SUBSCRIBE to a reaction track with a
property filter on REACTION_TYPE or REACTION_VALUE. The
relay only forwards objects matching the filter.

~~~
  Moderator                    Relay                 Publisher
    |                            |                       |
    |  SUBSCRIBE                 |                       |
    |  Track: react_v1_user-042  |                       |
    |  Property Filter:          |                       |
    |    REACTION_TYPE = 3       |                       |
    |    (raise-hand only)       |                       |
    |--------------------------->|                       |
    |                            |                       |
    |                            |  Objects arrive:      |
    |                            |  emoji (type=1)       |
    |                            |  raise-hand (type=3)  |
    |                            |  applause (type=4)    |
    |                            |<----------------------|
    |                            |                       |
    |                            |  (Relay filters:      |
    |                            |   drops type!=3)      |
    |                            |                       |
    |  Only raise-hand objects   |                       |
    |<---------------------------|                       |
    |                            |                       |
~~~

These two approaches can be combined: top-N track selection
on a namespace subscription together with object property
filters, so the relay selects the most active tracks AND
only forwards objects matching the specified criteria.

# Security Considerations

TODO

# IANA Considerations

TODO

--- back

# Acknowledgments
{:numbered="false"}

The authors thank the IETF MoQ working group for the
foundational work on Media over QUIC Transport.
