---
title: "Real-Time Language Translation over MoQ Transport"
abbrev: "moq-translate"
category: info

docname: draft-authors-moq-translate-latest
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
 - Translation
 - Real-Time
 - Agentic
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"

author:
  -
    fullname: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:
  MoQTransport: I-D.draft-ietf-moq-transport
  JSON: RFC8259
  OPUS:
    title: "Definition of the Opus Audio Codec"
    target: https://www.rfc-editor.org/rfc/rfc6716
    date: 2012
    author:
    - fullname: JM. Valin
    - fullname: K. Vos
    - fullname: T. Terriberry

informative:
  MOCHA: I-D.draft-jennings-moq-mocha
  MOCHA-ARCH: I-D.draft-jennings-moq-mocha-arch

--- abstract

This document specifies how to build real-time language translation
and agentic media processing systems using MoQ Transport (MOQT)
publish/subscribe primitives. It defines namespace conventions,
track naming, object formats, and subscription patterns that enable
multi-participant, multi-language audio translation with transcript
generation. The architecture supports both dedicated translation
servers and composable agentic pipelines where independent AI
services subscribe to input tracks and publish transformed output
tracks through a shared MoQ relay.

--- middle

# Introduction

Real-time language translation is a compelling use case for MoQ
Transport. The publish/subscribe model maps naturally to translation
pipelines: speakers publish audio in their native language, a
translation service subscribes to all input audio, runs ASR
(Automatic Speech Recognition) and TTS (Text-to-Speech), and
publishes translated audio and transcripts as new tracks that
listeners subscribe to in their preferred language.

This document specifies one way to wire up translation over MOQT.
The design prioritizes:

* **Dynamic discovery** — participants and languages are discovered
  at runtime via SUBSCRIBE_NAMESPACE; no pre-configuration needed.
* **Per-language output** — each target language gets its own audio
  mix track and transcript track, so clients subscribe only to what
  they need.
* **Composable agents** — the same namespace conventions support
  replacing a monolithic translation server with independent
  agentic services (ASR agent, translation agent, TTS agent) that
  chain via relay-mediated pub/sub.
* **Low latency** — sub-second glass-to-glass by streaming audio
  in small frame groups over QUIC streams.

## Use Cases

### Multi-Language Conference Call

A video conference with speakers of different languages. Each
speaker hears all other speakers translated into their own language
in real-time. Transcripts appear as synchronized captions.

Example: Alice speaks English, Bob speaks Spanish, Carol speaks
Japanese. Alice hears Bob and Carol in English. Bob hears Alice
and Carol in Spanish. Carol hears Alice and Bob in Japanese.

### Agentic Media Pipeline

Independent AI services process media through a shared relay without
direct coupling. An ASR agent subscribes to raw audio and publishes
text transcripts. A translation agent subscribes to transcripts in
one language and publishes translated text. A TTS agent subscribes
to translated text and publishes synthesized audio. Each agent is
independently deployable, scalable, and replaceable.

### Live Event Translation

A speaker at a conference publishes audio. Multiple translation
servers each handle a subset of target languages. Thousands of
attendees subscribe to their preferred language track through relay
fan-out. The relay caches recent groups so late-joining attendees
hear audio from the current sentence boundary.

### Podcast/Broadcast Dubbing

A pre-recorded audio source is published as a live track. A dubbing
service subscribes, translates, and publishes voice-cloned output
in multiple languages. Downstream clients or CDN ingest nodes
subscribe to the dubbed tracks.


# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology defined in {{MoQTransport}} including
Track, Group, Object, Namespace, and Relay.

The following additional terms are used:

Session:
: A logical grouping of participants who can hear each other.
  Identified by a SessionID. Analogous to a meeting room or call.

Participant:
: A human or automated entity that publishes or subscribes to
  audio within a session. Each participant has a ParticipantID
  and a source language.

ParticipantID:
: A unique identifier for a participant within a session. Format
  is `<name>-<lang>` where `<name>` is a human-readable label and
  `<lang>` is the BCP 47 language code (e.g., `alice-en`,
  `bob-es`).

Translation Server:
: A service that subscribes to participant audio, performs ASR,
  translation, and TTS, then publishes translated audio and
  transcripts.

Agent:
: An independent service that performs one stage of a media
  processing pipeline (e.g., ASR-only, translation-only,
  TTS-only). Agents communicate exclusively through the relay
  using the namespace conventions in this document.

Mix Track:
: A server-published audio track containing the mixed translated
  output of all speakers in a given target language.


# Namespace Design {#namespace-design}

## Design Principles

The namespace is structured as a tuple that encodes enough context
for any subscriber to understand what a track contains without
out-of-band signaling. The hierarchy enables SUBSCRIBE_NAMESPACE
at any level for progressive discovery.

All namespace tuples follow this pattern:

~~~
  (<app>, <env>, <session_id>, <role>, <sub_role>, [<participant_id>], [<lang>])
~~~

Where:

* `<app>` — Application identifier (e.g., "translate_v1")
* `<env>` — Deployment environment (e.g., "prod", "staging")
* `<session_id>` — Unique session/meeting identifier
* `<role>` — "client" or "server"
* `<sub_role>` — "in" for client input, omitted for server output
* `<participant_id>` — Identifies the publishing participant
* `<lang>` — BCP 47 language code of the content

## Client Input Namespace

Participants publish their source audio under:

~~~
  ("translate_v1", <env>, <session_id>, "client", "in", <participant_id>, <lang>)
~~~

Example: Alice speaking English in session "mtg-42" on production:

~~~
  ("translate_v1", "prod", "mtg-42", "client", "in", "alice-en", "en")
~~~

The track name is always `audio`.

## Server Output Namespace

The translation server publishes output under:

~~~
  ("translate_v1", <env>, <session_id>, "server", <instance_id>)
~~~

Where `<instance_id>` is a unique identifier for the server instance
(e.g., `srv-a3f7c201`). This allows multiple server instances to
publish without collision and enables clients to discover all active
servers via SUBSCRIBE_NAMESPACE.

Track names under the server namespace:

| Track Name          | Content                                    |
|:=================== |:========================================== |
| `mix/<lang>/audio`  | Mixed translated audio for target language |
| `transcript/<lang>` | Transcript objects for target language     |

Example server tracks for a session with English and Spanish:

~~~
  Namespace: ("translate_v1", "prod", "mtg-42", "server", "srv-a3f7c201")

  Tracks:
    mix/en/audio        — All non-EN speakers translated to English
    mix/es/audio        — All non-ES speakers translated to Spanish
    transcript/en       — Transcripts in English
    transcript/es       — Transcripts in Spanish
~~~

## Discovery via SUBSCRIBE_NAMESPACE {#discovery}

The namespace hierarchy enables targeted discovery at each level:

| Subscriber | SUBSCRIBE_NAMESPACE Prefix                                | Discovers                       |
|:========== |:========================================================= |:=============================== |
| Server     | `("translate_v1", <env>, <session_id>, "client", "in")`  | All participant input tracks    |
| Client     | `("translate_v1", <env>, <session_id>, "server")`        | All server instances and tracks |

When a new participant joins and publishes, the relay announces their
namespace to all entities that have a matching SUBSCRIBE_NAMESPACE
prefix. The server can then subscribe to that participant's audio
track without any out-of-band signaling.

## MOQT Object Mapping

~~~
  Namespace: ("translate_v1", "prod", "mtg-42", "client", "in", "alice-en", "en")
    |
    +-- Track: audio
        +-- Group 0
        |   +-- Object 0: [20ms Opus frame]
        |   +-- Object 1: [20ms Opus frame]
        |   +-- ...
        |   +-- Object 49: [20ms Opus frame]
        |
        +-- Group 1
        |   +-- Object 0: [20ms Opus frame]
        |   +-- ...
        +-- ...

  Namespace: ("translate_v1", "prod", "mtg-42", "server", "srv-a3f7c201")
    |
    +-- Track: mix/en/audio
    |   +-- Group 0
    |   |   +-- Object 0: [response_type | flags | audio_data]
    |   |   +-- ...
    |
    +-- Track: transcript/en
        +-- Group 0
        |   +-- Object 0: {"participant_id": "bob-es", "text": "Hola...",
        |   |               "translation": "Hello...", ...}
        +-- ...
~~~


# Audio Object Format {#audio-format}

## Input Audio (Client → Server)

Each MOQT object contains a single audio frame. The payload is
raw Opus-encoded audio {{OPUS}} at 16kHz sample rate, mono channel.

Object metadata:

| Field      | Value                          |
|:========== |:============================== |
| GroupID    | Increments every N frames      |
| ObjectID   | Frame index within group (0 to N-1) |
| Priority   | 128 (default, configurable)    |
| TTL        | 5000ms                         |

The group size N determines how many frames are batched into one
QUIC stream. Default is 50 frames (1 second of audio at 20ms per
frame). Smaller groups reduce latency at the cost of more stream
overhead. Larger groups improve efficiency for high-participant
sessions.

**Option:** PCM (signed 16-bit little-endian) MAY be used instead
of Opus when the client and server are co-located or bandwidth is
not a concern. The encoding is signaled during session negotiation
(see {{session-negotiation}}).

## Output Audio (Server → Client) {#output-audio}

Server output objects prepend an 8-byte header before the audio
payload:

~~~
  +-------------------+--------------+--------------------+
  | Response Type     |    Flags     |   Audio Payload    |
  |   (4 bytes)       |  (4 bytes)   |   (variable)       |
  +-------------------+--------------+--------------------+
~~~

### Response Type (uint32, big-endian)

| Value | Name               | Description                               |
|:===== |:================== |:========================================= |
| 0     | MIX_PLAYBACK       | Mixed translation for the subscriber's language |
| 1     | SELF_TRANSLATED    | Speaker's own audio translated (for monitoring) |
| 2     | OTHER_TRANSLATED   | Single other speaker translated           |
| 3     | OTHER_PASSTHROUGH  | Untranslated audio from another speaker   |

MIX_PLAYBACK (type 0) is the primary output. Clients that want
a simple experience subscribe only to `mix/<lang>/audio` and play
every object with response type 0.

**Option:** Types 1-3 exist for advanced clients that want to
render individual speaker audio separately (e.g., spatial audio
positioning or per-speaker volume control). Most implementations
need only type 0.

### Flags (uint32, big-endian)

| Bit  | Meaning                        |
|:==== |:============================== |
| 0x01 | Contains translated audio      |
| 0x02 | Contains translation echo      |

### Audio Payload

Opus-encoded audio matching the negotiated sample rate. Same
encoding as input audio.


# Transcript Object Format {#transcript-format}

Transcript objects are JSON {{JSON}} encoded. Each object represents
one utterance segment (partial or final).

~~~json
{
  "participant_id": "bob-es",
  "source_lang": "es",
  "text": "Hola, ¿cómo están todos?",
  "translation": "Hello, how is everyone?",
  "is_final": true,
  "start_time_ms": 1500,
  "end_time_ms": 3200
}
~~~

| Field            | Type    | Description                                    |
|:================ |:======= |:============================================== |
| participant_id   | string  | Who spoke this segment                         |
| source_lang      | string  | BCP 47 code of the original speech             |
| text             | string  | Original speech text (ASR output)              |
| translation      | string  | Translated text in the track's target language |
| is_final         | boolean | `false` for interim results, `true` for final  |
| start_time_ms    | integer | Segment start relative to group start          |
| end_time_ms      | integer | Segment end relative to group start            |

Interim transcripts (`is_final: false`) allow clients to display
in-progress captions. The final transcript for a segment supersedes
all interim transcripts with the same `start_time_ms` from the same
`participant_id`.


# Session Lifecycle {#session-lifecycle}

## Session Negotiation {#session-negotiation}

Sessions are established through out-of-band signaling (WebSocket,
gRPC, or HTTP). The negotiation produces a `SessionConfig` that
both client and server use to construct namespaces:

~~~json
{
  "relay_url": "moq://relay.example.com:4433",
  "namespace_prefix": ["translate_v1", "prod"],
  "session_id": "mtg-42",
  "transport": "quic",
  "audio_sample_rate": 16000,
  "audio_format": "opus",
  "output_languages": ["en", "es", "ja"]
}
~~~

The `output_languages` field tells the server which target language
tracks to publish. It is typically the union of all participant
languages in the session.

**Option:** For serverless/peer-to-peer deployments, the session
config can be exchanged via a shared document or discovery service
rather than a signaling connection.

## Participant Join Flow

~~~
  Client                         Relay                       Server
    |                              |                            |
    |  QUIC connect                |                            |
    |----------------------------->|                            |
    |                              |                            |
    |  PUBLISH(audio)              |                            |
    |  ns: (...client/in/alice-en/en)                           |
    |----------------------------->|                            |
    |                              |  ANNOUNCE (namespace)      |
    |                              |--------------------------->|
    |                              |                            |
    |                              |  SUBSCRIBE(audio)          |
    |                              |  ns: (...client/in/alice-en/en)
    |                              |<---------------------------|
    |                              |                            |
    |  SUBSCRIBE(mix/en/audio)     |                            |
    |  ns: (...server/srv-xxx)     |                            |
    |----------------------------->|                            |
    |                              |  Objects flow              |
    |<---------------------------->|<=========================>|
~~~

Step by step:

1. Client connects to the relay via QUIC (or WebTransport).
2. Client starts PUBLISH on its input namespace/track (`audio`).
3. Relay announces the new namespace to the server (which has a
   matching SUBSCRIBE_NAMESPACE prefix for `client/in`).
4. Server subscribes to the client's audio track.
5. Client subscribes to the server's mix track in its target
   language.
6. Audio objects flow bidirectionally through the relay.

## Session Teardown

Sessions auto-terminate after a configurable inactivity timeout
(default: 2 minutes of no audio frames from any participant).
When a participant leaves:

1. Client stops publishing (closes QUIC stream or sends FIN).
2. Relay propagates unsubscribe to server.
3. Server removes participant from translation pipeline.
4. If no participants remain after timeout, server tears down
   the session and stops publishing output tracks.


# Agentic Architecture {#agentic}

The namespace conventions in this document support decomposing
a monolithic translation server into independent agents that
communicate exclusively through the MoQ relay. This section
describes how to wire up an agentic pipeline.

## Why Agents?

A monolithic translation server bundles ASR, machine translation,
and TTS into a single process. This works but has limitations:

* **Scaling** — ASR is compute-intensive; you may want 10x ASR
  instances per TTS instance, but they're locked together.
* **Replaceability** — swapping a TTS model requires redeploying
  the entire server.
* **Composition** — you can't easily add new processing stages
  (sentiment analysis, content moderation, speaker diarization)
  without modifying the server.

Agents solve this by making each stage an independent MoQ
publisher/subscriber that reads input from one set of tracks and
writes output to another. The relay handles discovery and fan-out.

## Agent Namespace Convention

Agents publish under a namespace that identifies their role and
the processing stage:

~~~
  ("translate_v1", <env>, <session_id>, "agent", <agent_type>, <instance_id>)
~~~

Where `<agent_type>` identifies the processing stage:

| agent_type   | Input                     | Output                     |
|:============ |:========================= |:========================== |
| `asr`        | Client audio tracks       | `transcript/<lang>` tracks |
| `mt`         | ASR transcript tracks     | `translated/<lang>` tracks |
| `tts`        | MT translated text tracks | `audio/<lang>` tracks      |
| `mod`        | Any text tracks           | `moderated/<lang>` tracks  |

## Agentic Pipeline Example

~~~
  Client Audio                    ASR Agent                MT Agent
  (alice-en/audio)                                        
       |                              |                       |
       | Opus frames                  |                       |
       |--[relay]-------------------->|                       |
       |                              |                       |
       |                    transcript/en (JSON)              |
       |                              |---[relay]------------>|
       |                              |                       |
       |                              |            translated/es (JSON)
       |                              |                       |---[relay]--->
       |                                                                    |
       |                                                          TTS Agent |
       |                                                                    |
       |                                                    audio/es (Opus) |
       |<--[relay]----------------------------------------------------------+
       |
  Client plays translated audio
~~~

Each agent:

1. Connects to the relay.
2. Issues SUBSCRIBE_NAMESPACE for its input prefix.
3. As input tracks are discovered, subscribes to them.
4. Processes objects as they arrive.
5. Publishes output on its own namespace/tracks.

## Agent Discovery

Agents discover each other the same way clients discover servers —
via SUBSCRIBE_NAMESPACE:

| Agent     | SUBSCRIBE_NAMESPACE Prefix                                    | Discovers            |
|:========= |:============================================================= |:==================== |
| ASR       | `("translate_v1", <env>, <session_id>, "client", "in")`      | Client audio inputs  |
| MT        | `("translate_v1", <env>, <session_id>, "agent", "asr")`      | ASR transcript output |
| TTS       | `("translate_v1", <env>, <session_id>, "agent", "mt")`       | MT text output       |
| Client    | `("translate_v1", <env>, <session_id>, "agent", "tts")`      | TTS audio output     |

This allows agents to be started in any order. When a new agent
comes online and starts publishing, downstream agents automatically
discover it through relay announcements.

## Mixing Monolithic and Agentic

A deployment can mix approaches. For example, use a monolithic
server for the hot path (ASR+MT+TTS) but add an independent
content moderation agent that subscribes to the same transcript
tracks. The moderation agent publishes flags on a `moderated/`
track that the client can optionally subscribe to for filtering.


# Delivery Properties {#delivery}

## Reliability

All audio and transcript objects are delivered via QUIC streams
(reliable, ordered within a stream). This ensures no audio gaps
from packet loss. For ultra-low-latency deployments where slight
audio gaps are acceptable, QUIC datagrams (unreliable) MAY be
used — signaled via `trackMode: "datagram"` in session config.

## Priority and Preemption

Audio tracks carrying active speech SHOULD be published at
higher priority than silence. Relay implementations MAY use
QUIC stream priorities to preempt less important traffic during
congestion.

Suggested priority values:

| Content              | Priority |
|:==================== |:======== |
| Active speech audio  | 64       |
| Silence/comfort noise| 192      |
| Transcripts          | 128      |

## Caching and Late Join

Relays SHOULD cache the most recent N groups of each track to
support late-joining subscribers. A subscriber that joins mid-
session receives cached audio from the current group boundary,
enabling a smooth start without waiting for the next utterance.

Recommended cache: last 5 groups (5 seconds of audio at default
group size).

## Fan-Out

The relay handles multi-subscriber delivery. A single translation
server publishes `mix/en/audio` once, and the relay duplicates it
to all English-speaking subscribers. This scales to thousands of
listeners without additional server load.


# Security Considerations

## Transport Security

All QUIC connections to the relay MUST use TLS 1.3. Certificates
MUST be validated. This protects audio and transcript content
in transit between each participant/server and the relay.

## Authorization

The relay MUST enforce authorization on publish and subscribe
operations. A participant SHOULD only be able to:

* Publish to their own input namespace.
* Subscribe to server/agent output namespaces within their session.

Tokens are issued during session negotiation and presented to the
relay on connect. Token format and validation are deployment-
specific and out of scope for this document.

## Content Confidentiality

Audio content is visible to the relay in cleartext. For deployments
requiring end-to-end encryption, an E2EE layer (such as MLS-based
key management) can be applied at the application layer. This is
out of scope for this document but compatible with the architecture.

## Agent Isolation

In agentic deployments, each agent has its own relay credentials
scoped to its input/output namespaces. An ASR agent cannot publish
to TTS output tracks or subscribe to other agents' internal state.
This limits blast radius if an agent is compromised.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments

Thanks to the IETF MoQ working group for developing the transport
layer that makes this architecture possible.
