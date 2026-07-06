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
  MOCHA-IDENTITY: I-D.draft-jennings-moq-mocha-identity
  JSON: RFC8259


informative:

--- abstract

This document specifies reactions for MOCHA (MoQ Open
Communication & Hosting Architecture). It supports
message-targeted reactions (emoji on a specific chat message)
and standalone reactions (engagement signals such as
applause or raised hands).

--- middle

# Introduction

This document specifies how MOCHA enables real-time reaction
exchange using MOQT's publish/subscribe model. Two modes
are supported:

Message-Targeted Reactions:
: Emoji responses referencing a specific chat message MsgID.

Standalone Reactions:
: Standalone engagement signals (applause, raised hands)
  directed at the channel as a whole.

Both modes use the same track structure.


# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology defined in {{MOCHA-ARCH}}
and MOQT primitives defined in {{MoQTransport}}. Message
identifiers (MsgID) are defined in {{MOCHA-CHAT}}.


# Reaction Track Naming {#reaction-naming}

## Reactions Namespace

~~~
  ("mocha_v1", <Provider>, <OrgID>, "reactions", <TeamID>,
   <ChannelID>, <HClientID>)
~~~

HClientID and HDevID are derived per {{MOCHA-IDENTITY}},
Section "Identifier Derivation".

## Track Names

Each device publishes reactions on its own track:

~~~
  react_v1_<HDevID>
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

## Reaction Structure

~~~json
{
  "type": 1,
  "value": "thumbs-up",
  "ts": 1715788800000,
  "sender": {
    "userId": "<obtained via credential, see MOCHA-IDENTITY>",
    "devId": "<obtained via credential, see MOCHA-IDENTITY>"
  },
  "target": {
    "msgId": "a1b2c3...f0a1b2"
  },
  "ext": {}
}
~~~

## Fields

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
  absent for standalone reactions. {#target-field}

ext (OPTIONAL):
: Extension fields. Implementations MUST ignore unknown keys.

## Reaction Types {#reaction-types}

| Value | Name     | Description                              |
|:===== |:======== |:======================================== |
| 1     | emoji    | Emoji reaction (heart, fire, thumbs-up)  |
| 2     | vote     | Poll or vote response                    |
| 3     | raise    | Raise/lower hand                         |
| 4     | applause | Applause or cheer                        |
| 5     | custom   | Application-defined reaction             |

Extensible via IANA registry.

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

TODO: The range of emoji values to be supported is not clear
at this point and further discussion is needed.

Values 11-127 are reserved for future standard emoji.
Values 128+ are available for application-defined emoji.

Raise (type 3):

| Integer | Value |
|:======= |:===== |
| 1       | raise |
| 2       | lower |

Vote (type 2) and Applause (type 4) value encodings are
application-defined.

## Reaction Removal

To retract a reaction, a device publishes an object with the
same type, value, and target, plus `"remove": true`. The
receiver removes the matching reaction from display.


# Sending Reactions {#sending}

To send a reaction, a device publishes a MOQT object to its
track within the reactions namespace.

## Rate Limiting

* Message-targeted: At most 1 reaction of a given type per
  message per device (newer replaces older).
* Standalone: At most 1 reaction per second per device.

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

## Processing Standalone Reactions

1. Display as a transient signal (e.g., floating emoji,
   applause indicator).
2. For raise-hand: maintain a queue until "lower" is received.


# Security Considerations

TODO

# IANA Considerations

TODO

--- back

# Acknowledgments
{:numbered="false"}

TODO
