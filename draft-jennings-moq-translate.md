---
title: "Real-Time Language Translation over MoQ Transport"
abbrev: "moq-translate"
category: info

docname: draft-jennings-moq-translate-latest
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
    fullname: Cullen Jennings
    organization: Cisco
    email: fluffy@cisco.com

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


--- abstract

This document specifies namespace conventions, track naming, object
formats, and subscription patterns for real-time language translation
over MoQ Transport (MOQT). It covers both monolithic translation
servers and composable agentic pipelines.

--- middle

# Introduction

Speakers publish audio in their native language. A translation
service subscribes to all input audio, runs ASR and TTS, and
publishes translated audio and transcripts as new tracks.
Listeners subscribe to their preferred language.

Design priorities:

* Dynamic discovery via SUBSCRIBE_NAMESPACE.
* Per-language output tracks.
* Composable agents (ASR, MT, TTS as independent services).
* Sub-second latency via small frame groups over QUIC.

# Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}} and {{MoQTransport}}.

Context:
: Groups multiple sessions (translation, ASR, TTS, client
  connections) under a single shared identifier. Identified by a
  ContextID, obtained out-of-band (join URL, scheduling API,
  meeting identifier). Use-case agnostic — could be a meeting,
  broadcast, podcast, or live event. All clients and agents
  operating under the same ContextID can discover each other.

Session:
: A per-client connection with the server. Multiple sessions
  MAY exist under one context — one per client or agent
  instance. Sessions are transient (a client may disconnect
  and reconnect, creating a new session); the context persists.

ClientID:
: Opaque, globally unique, assigned by the provider. Exchanged
  during session negotiation based on the client's identity.

DevID:
: Identifies a publishing device. One client may have
  multiple devices publishing under the same namespace.

# Namespace Design {#namespace-design}

All tuples share a 2-element prefix:

~~~
(<app>, <context_id>, ...)
~~~

## Client Input (6 elements)

Participants publish their source audio under this namespace. The
server discovers new clients by subscribing to the prefix
`(<app>, <context_id>, "client", "in")`.

~~~
(<app>, <context_id>, "client", "in", <client_id>, <lang>)
~~~

Track name: `<rendition_id>_<DevID>`

* `<rendition_id>` — codec, sample rate, channels (e.g., `opus-16000-1`)
* `<DevID>` — the publishing device

Example:

~~~
Namespace: ("acme.com/translate", "mtg-42", "client", "in", "p-a1", "en")
Track:     "opus-16000-1_dev-x1"
~~~

## Server Output (4 elements)

The translation server publishes translated audio and transcripts
under this namespace. Clients discover server instances by
subscribing to the prefix `(<app>, <context_id>, "server")`.

~~~
(<app>, <context_id>, "server", <instance_id>)
~~~

Tracks:

| Track Name                           | Content                            |
|:--- |:--- |
| `mix/<lang>/audio`                   | Mixed translation into `<lang>`    |
| `self/<lang>/audio`                  | Speaker's own audio translated     |
| `passthrough/<client_id>/audio` | Untranslated audio from a speaker  |
| `transcript/<lang>`                  | Transcript objects in `<lang>`     |

Example:

~~~
Namespace: ("acme.com/translate", "mtg-42", "server", "srv-a3f7c201")
Tracks:    mix/en/audio, mix/es/audio, transcript/en, transcript/es
~~~

## Agent Output (5 elements)

Independent processing stages (ASR, machine translation, TTS)
publish their output under this namespace. Downstream agents or
clients discover upstream agents by subscribing to the prefix
`(<app>, <context_id>, "agent", <agent_type>)`.

~~~
(<app>, <context_id>, "agent", <agent_type>, <instance_id>)
~~~

The `<agent_type>` value identifies the processing stage:

`asr` (Automatic Speech Recognition):
: Subscribes to client audio tracks. Converts speech to text and
  publishes `transcript/<lang>` tracks containing JSON transcript
  objects.

`mt` (Machine Translation):
: Subscribes to ASR transcript tracks. Translates text from one
  language to another and publishes `translated/<lang>` tracks
  containing translated text objects.

`tts` (Text-to-Speech):
: Subscribes to MT translated text tracks. Synthesizes speech
  from text and publishes `audio/<lang>` tracks containing Opus
  audio frames.

`mod` (Content Moderation):
: Subscribes to any text tracks (transcripts or translations).
  Evaluates content against policy and publishes `moderated/<lang>`
  tracks containing moderation decisions or filtered text.

## Discovery

All entities find each other at runtime using SUBSCRIBE_NAMESPACE
with a prefix match. No pre-configuration or roster is needed —
the relay announces new publishers to all matching subscribers.

| Subscriber | SUBSCRIBE_NAMESPACE Prefix                  | Discovers                    |
|:--- |:--- |:--- |
| Server     | `(<app>, <context_id>, "client", "in")`    | All client inputs       |
| Client     | `(<app>, <context_id>, "server")`          | All server instances         |
| ASR agent  | `(<app>, <context_id>, "client", "in")`    | Client audio inputs          |
| MT agent   | `(<app>, <context_id>, "agent", "asr")`    | ASR transcript output        |
| TTS agent  | `(<app>, <context_id>, "agent", "mt")`     | MT text output               |
| Client     | `(<app>, <context_id>, "agent", "tts")`    | TTS audio output             |

Agents can start in any order. The relay announces new publishers
to matching SUBSCRIBE_NAMESPACE holders automatically.

## Field Summary

* `<app>` — globally unique application ID (e.g., "acme.com/translate")
* `<context_id>` — groups related sessions together; obtained
  out-of-band (join URL, scheduling API, invite); also scopes
  namespace discovery
* `<client_id>` — globally unique, provider-assigned
* `<lang>` — BCP 47 language code
* `<instance_id>` — unique ID for a server or agent instance


# Audio Objects {#audio-format}

## Input (Client to Server)

Each MOQT object contains a single audio frame. The codec, sample
rate, and channel count are determined by the `<rendition_id>` in
the track name (e.g., `opus-16000-1` means Opus, 16kHz, mono,
producing 20ms frames).

| Field    | Value                          |
|:--- |:--- |
| GroupID  | Increments every N frames      |
| ObjectID | 0 to N-1 within group          |

Default group size N = 50 (1 second). See {{priority-ttl}} for
priority and TTL values.

## Output (Server to Client)

Pure Opus payload, no application header. Track name identifies
content type. Metadata via properties:

Track property:

| Property       | Example         | Description                    |
|:--- |:--- |:--- |
| `rendition_id` | `opus-16000-1` | `<codec>-<sample_rate>-<channels>` |

Private object properties (optional):

| Property             | Type    | Description                          |
|:--- |:--- |:--- |
| `is_translated`      | boolean | True if translated                   |
| `source_client` | string  | Original speaker's ClientID     |
| `source_lang`        | string  | BCP 47 code of original speech       |


# Transcript Objects {#transcript-format}

JSON-encoded, one object per utterance segment:

~~~json
{
  "client_id": "p-b2",
  "source_lang": "es",
  "text": "Hola, como estan todos?",
  "translation": "Hello, how is everyone?",
  "is_final": true,
  "start_time_ms": 1500,
  "end_time_ms": 3200
}
~~~

| Field          | Type    | Description                               |
|:--- |:--- |:--- |
| client_id | string  | Who spoke                                 |
| source_lang    | string  | Original language                         |
| text           | string  | ASR output                                |
| translation    | string  | Translated text in track's target lang    |
| is_final       | boolean | False for interim, true for final         |
| start_time_ms  | integer | Segment start relative to group start     |
| end_time_ms    | integer | Segment end relative to group start       |

Final transcripts supersede interim results with the same
`start_time_ms` from the same `client_id`.


# Priority and TTL {#priority-ttl}

Each track type has recommended priority and TTL values. Lower
priority numbers are delivered first under congestion.

| Track                          | Priority | TTL     |
|:--- |:--- |:--- |
| Client input audio             | 64       | 5000ms  |
| Server mix audio               | 64       | 5000ms  |
| Server self/passthrough audio  | 128      | 5000ms  |
| Transcripts                    | 192      | 10000ms |
| Agent intermediate tracks      | 128      | 5000ms  |

Audio tracks carrying active speech SHOULD use priority 64.
During silence, publishers MAY raise the priority value to 192
to yield bandwidth to active speakers.

TTL bounds the staleness of cached objects. After expiry, relays
MAY discard the object. Transcript TTL is longer because text
is useful for late-join context even when slightly stale.


# Session Lifecycle {#session-lifecycle}

## Negotiation {#session-negotiation}

The client knows `<app>` and `<context_id>` (obtained out-of-band).
Session negotiation happens via WebSocket, gRPC, or HTTP before
any MOQT connection is established.

The client sends:

~~~json
{
  "context_id": "mtg-42",
  "client_identity": "<auth token or credentials>"
}
~~~

The server responds with:

~~~json
{
  "relay_url": "moq://relay.example.com:4433",
  "instance_id": "srv-a3f7c201",
  "session_id": "sess-7f2b"
}
~~~

* `instance_id` — the server's own instance ID, so the client
  knows which server namespace to subscribe to.
* `session_id` — for server-side state and auth token scoping.
  Does NOT appear in namespace tuples.

All namespaces are derived from `<app>` + `<context_id>` +
the identifiers exchanged above.

## Join Flow

After negotiation, both client and server connect to the relay
and set up subscriptions before the client publishes audio.

~~~
Client                         Relay                       Server
  |                              |                            |
  |--- WS/gRPC: negotiate ---------------------------------------->|
  |<-- {relay_url, instance_id, session_id} ------------------------|
  |                              |                            |
  |  QUIC connect                |       QUIC connect         |
  |----------------------------->|<---------------------------|
  |                              |                            |
  |                              |  SUBSCRIBE_NAMESPACE       |
  |                              |  prefix: (<app>,<ctx>,     |
  |                              |    "client","in")          |
  |                              |<---------------------------|
  |                              |                            |
  |  SUBSCRIBE_NAMESPACE         |                            |
  |  prefix: (<app>,<ctx>,       |                            |
  |    "server")                 |                            |
  |----------------------------->|                            |
  |                              |                            |
  |  PUBLISH(opus-16000-1_dev-x1)|                            |
  |  ns: (<app>,<ctx>,           |                            |
  |    "client","in","p-a1","en")|                            |
  |----------------------------->|                            |
  |                              |  PUBLISH forwarded         |
  |                              |--------------------------->|
  |                              |                            |
  |                              |  PUBLISH OK                |
  |                              |<---------------------------|
  |  PUBLISH OK                  |                            |
  |<-----------------------------|                            |
  |                              |                            |
  |  Audio objects flow          |                            |
  |----------------------------->|--------------------------->|
  |                              |                            |
  |                              |  PUBLISH(mix/en/audio)     |
  |                              |  ns: (<app>,<ctx>,         |
  |                              |    "server","srv-xxx")     |
  |                              |<---------------------------|
  |  PUBLISH forwarded           |                            |
  |<-----------------------------|                            |
  |  PUBLISH OK                  |                            |
  |----------------------------->|--------------------------->|
  |                              |                            |
  |  Server audio objects flow   |                            |
  |<-----------------------------|<---------------------------|
~~~

## Teardown

Sessions auto-terminate after 2 minutes of inactivity. On leave:

1. Client stops publishing.
2. Server removes client from pipeline.
3. If no clients remain after timeout, server stops
   publishing output tracks.


# Agentic Pipelines {#agentic}

A monolithic server bundles ASR+MT+TTS. Agents decompose this
into independent MOQT publishers/subscribers:

~~~
Client          Relay         ASR           MT            TTS
  |               |             |             |             |
  |  PUBLISH      |             |             |             |
  |  (audio)      |             |             |             |
  |-------------->|             |             |             |
  |               | PUBLISH fwd |             |             |
  |               |------------>|             |             |
  |               |             |             |             |
  |               |  PUBLISH    |             |             |
  |               |  (transcript/<lang>)      |             |
  |               |<------------|             |             |
  |               |    PUBLISH fwd            |             |
  |               |-------------------------->|             |
  |               |             |             |             |
  |               |             |  PUBLISH    |             |
  |               |             |  (translated/<lang>)      |
  |               |<------------------------------|         |
  |               |               PUBLISH fwd     |         |
  |               |---------------------------------------->|
  |               |             |             |             |
  |               |                        PUBLISH          |
  |               |                        (audio/<lang>)   |
  |               |<-----------------------------------------|
  |  PUBLISH fwd  |             |             |             |
  |<--------------|             |             |             |
  |               |             |             |             |
  | Audio objects |             |             |             |
  |<=============>|<==========>|<==========>|<============>|
~~~

Each agent connects to the relay, issues SUBSCRIBE_NAMESPACE for
its input prefix, processes objects, and publishes output. Agents
are independently scalable and replaceable.

A deployment can mix approaches: monolithic server for the hot path
plus independent agents for additional processing (moderation,
diarization, sentiment).


# Security Considerations

TODO


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments

Thanks to the IETF MoQ working group.
