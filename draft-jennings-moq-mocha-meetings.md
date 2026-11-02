---
title: "MOCHA Meetings: Real-Time Conferencing over MoQ Transport"
abbrev: "mocha-meetings"
category: std

docname: draft-jennings-moq-mocha-meetings-latest
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
 - Conferencing
 - Meetings
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
  MSF: I-D.draft-ietf-moq-msf
  LOC: I-D.draft-ietf-moq-loc
  SecureObjects: I-D.draft-jennings-moq-secure-objects
  DTS: I-D.draft-wilaw-moq-dts4moq

informative:
  RFC3550:
  RFC8825:

--- abstract

This document specifies how multi-party audio and video
meetings are conducted using MOCHA (MoQ Open Communication &
Hosting Architecture). It defines namespace conventions, media
track design, catalog distribution, and participant flows for
real-time conferencing over MoQ Transport (MOQT). The
specification uses Top-N Track Filters for active speaker
selection and DTS (Dynamic Track Switching) for per-source
quality adaptation, enabling meetings to scale from small
group calls to large conferences while minimizing downstream
bandwidth.

--- middle

# Introduction

Real-time meetings are a core use case for MOCHA. This
document specifies how participants join, publish, and
subscribe to audio and video media in a meeting using MOQT's
publish/subscribe primitives.

Each participant publishes media under a namespace that
encodes the meeting, rendition, and source identity. This
design enables two orthogonal relay mechanisms: Top-N Track
Filters select WHICH sources to forward (the N loudest
speakers), and DTS/SSTS select WHAT quality to forward per
source (based on available bandwidth). Together they reduce
downstream bandwidth from O(all participants x all qualities)
to O(N sources x 1 quality each), with all switching handled
at the relay.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses terminology and architecture defined in
{{MOCHA-ARCH}}, including Provider, Organization, Team,
Channel, and Device. The MOQT transport primitives
(Track, Group, Object, Namespace, Relay) are defined in
{{MoQTransport}}. DTS and Switching Sets are defined in
{{DTS}}.

The following additional terms are used:

MeetingID:
: A unique identifier for a meeting instance within a
  channel. A new MeetingID is assigned each time a meeting
  is started in a channel.

SourceID:
: An identifier for a media source (participant device)
  within a meeting. Used as a namespace tuple element to
  scope tracks per source.

RenditionID:
: An identifier for a specific media encoding variant
  (e.g., "720p", "360p", "180p"). Used as a namespace
  tuple element to scope tracks per quality level.

Catalog:
: An MSF {{MSF}} catalog track describing the available
  media renditions published by a device.

Top-N Track Filter:
: A SUBSCRIBE_TRACKS parameter that instructs the relay
  to forward only the N tracks with the highest value of
  a specified object property (LOUDNESS). See {{top-n}}.

DTS (Dynamic Track Switching):
: A relay mechanism that selects one rendition per source
  within a switching set based on available bandwidth.
  See {{dts-ssts}}.

SSTS (Switching Set Track Selection):
: The subscriber-side assignment of tracks into switching
  sets, enabling DTS to operate per-source.

LOUDNESS:
: An object property carried on video objects indicating
  the current voice activity level of the source.
  Publishers start a new group when they begin speaking.

Active Speaker:
: The participant currently producing the most significant
  audio activity, as determined by the LOUDNESS object
  property and Top-N Track Filter selection.

# Meeting Namespace {#meeting-namespace}

## Identifier Derivation {#id-derivation}

The SourceID used in namespace tuples is derived per
{{MOCHA-IDENTITY}}, Section "Identifier Derivation".

## Media Namespace

The media namespace encodes the meeting, rendition, and
source identity as namespace tuples. This design ensures
that each namespace contains exactly one track per source
at a given rendition, enabling SUBSCRIBE_TRACKS with
prefix matching to operate on a single rendition across
all sources.

Media tracks use the following namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "media",
   <TeamID>, <ChannelID>, <MeetingID>,
   <RenditionID>, <SourceID>)
~~~

The track name is the media type:

~~~
  Track: "video"
  Track: "audio"
~~~

For example, Alice publishing three video qualities and
audio in the "standup" channel of team "eng" at "acme":

~~~
  NS: ("mocha_v1", "example.com", "acme", "media",
       "eng", "standup", "mtg-2024-05-15", "720p", "alice")
  Track: "video"

  NS: ("mocha_v1", "example.com", "acme", "media",
       "eng", "standup", "mtg-2024-05-15", "360p", "alice")
  Track: "video"

  NS: ("mocha_v1", "example.com", "acme", "media",
       "eng", "standup", "mtg-2024-05-15", "180p", "alice")
  Track: "video"

  NS: ("mocha_v1", "example.com", "acme", "media",
       "eng", "standup", "mtg-2024-05-15", "audio", "alice")
  Track: "opus48k"
~~~

This structure enables prefix-based subscriptions. A
SUBSCRIBE_TRACKS with prefix (..., "mtg-2024-05-15", "720p")
matches all sources' 720p video tracks — exactly one track
per source — making Top-N track filtering deterministic
regardless of object arrival order.

## Catalog Namespace {#catalog-namespace}

The catalog namespace carries metadata tracks that describe
each device's available media renditions. By separating the
catalog from media, subscribers can discover participants
and their capabilities without subscribing to any media
streams. Each device publishes exactly one catalog track
in this namespace, enabling other participants to learn
what audio/video tracks are available for subscription.

Catalog tracks use a separate namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "catalog",
   <TeamID>, <ChannelID>, <MeetingID>, <SourceID>)
~~~

## Catalog Track Names

Each device publishes a catalog describing its available
renditions:

~~~
  Track: "catalog"
~~~

For example:

~~~
  NS: ("mocha_v1", "example.com", "acme", "catalog",
       "eng", "standup", "mtg-2024-05-15", "alice")
  Track: "catalog"
~~~

# Media Track Design {#media-design}

## Audio Tracks

Each participant publishes a single audio track per
rendition. The audio codec and encoding parameters are
specified by the RenditionID and described in the device's
catalog (see {{catalog-format}}).

### Audio Object Mapping

Each audio frame maps to a single MOQT Object:

* Group ID increments per frame (no inter-frame dependencies)
* Object ID is 0 within each group
* Objects MAY be sent via QUIC Datagrams for lowest latency

## Video Tracks

Video publishers provide one or more rendition tracks
(simulcast). Each rendition represents a different
resolution or quality level.

### Video Object Mapping

~~~
  NS: (..., "720p", "alice")  Track: "video"
    |
    +-- Group 1 (GOP / keyframe boundary)
    |   +-- Object 0: I-frame  [LOUDNESS=82]
    |   +-- Object 1: P-frame  [LOUDNESS=82]
    |   +-- Object 2: P-frame  [LOUDNESS=80]
    |   ...
    |
    +-- Group 2 (next GOP)
    |   +-- Object 0: I-frame  [LOUDNESS=90]
    |   ...
~~~

* Each encoded video frame is packaged as a LOC container
  and placed in a single MOQT Object
* Each video object carries a LOUDNESS property indicating
  the source's current voice activity level; this enables
  Top-N filtering at the relay without inspecting audio
* Group ID increments at each keyframe boundary (GOP). The
  first Group ID is the lower 32 bits of the current wall
  clock time in milliseconds. Publishers MUST start a new
  group when they begin speaking (transition from silent to
  active) to provide a clean switch point
* Object IDs are sequential within each group
* Objects are sent via QUIC Streams for reliable delivery

### Quality Variants

Publishers advertise available quality variants via the
catalog. Typical variants include:

| RenditionID | Resolution | Frame Rate | Bitrate Range |
|:============|:===========|:===========|:==============|
| 1080p       | 1920x1080  | 30 fps     | 2-4 Mbps      |
| 720p        | 1280x720   | 30 fps     | 1-2 Mbps      |
| 360p        | 640x360    | 30 fps     | 300-600 Kbps  |
| 180p        | 320x180    | 15 fps     | 100-200 Kbps  |

## Top-N Track Filters {#top-n}

Top-N Track Filters select WHICH sources to forward. The
subscriber issues SUBSCRIBE_TRACKS with a TOP_TRACKS_FILTER
parameter, and the relay evaluates LOUDNESS metadata carried
on video objects to identify the N loudest sources. Only
those sources' tracks are forwarded downstream.

Each publisher carries a LOUDNESS property on its video
objects indicating current voice activity level. Publishers
start a new group when they begin speaking, providing a
clean switch point for the relay.

The subscriber sends one SUBSCRIBE_TRACKS per rendition:

~~~
  SUBSCRIBE_TRACKS {
    Prefix: (..., "mtg-2024-05-15", "720p")
    TOP_TRACKS_FILTER (0x29) {
      Property: LOUDNESS
      MaxTopTracks: 4
    }
  }
  SUBSCRIBE_TRACKS {
    Prefix: (..., "mtg-2024-05-15", "360p")
    TOP_TRACKS_FILTER { Property: LOUDNESS, MaxTopTracks: 4 }
  }
  SUBSCRIBE_TRACKS {
    Prefix: (..., "mtg-2024-05-15", "180p")
    TOP_TRACKS_FILTER { Property: LOUDNESS, MaxTopTracks: 4 }
  }
~~~

Each prefix matches exactly one track per source at that
rendition. Top-N is evaluated independently per rendition,
resulting in the same N sources selected consistently across
all subscriptions regardless of object arrival order.

When a track's object arrives, the relay evaluates its
LOUDNESS. If the track proves top-N membership, the relay
sends a PUBLISH message downstream. When a source falls out
of the top-N (another source becomes louder), the relay
sends REQ_UPDATE with FWD=0 for the demoted source and
PUBLISH with FWD=1 for the promoted source.

## DTS and SSTS {#dts-ssts}

DTS (Dynamic Track Switching) and SSTS (Switching Set Track
Selection) select WHAT quality to forward per source. While
Top-N decides which sources are active, DTS picks one
rendition per source based on available bandwidth.

Top-N evaluates first, then DTS operates within the
selected sources.

### Switching Sets

After the relay sends PUBLISH messages for a source's
tracks (one per rendition), the subscriber responds with
PUBLISH_OK including a SWITCHING-SET-ASSIGNMENT. Each
switching set groups the renditions of a single source:

~~~
  Set 1 (alice): 720p (thr=2000) | 360p (thr=1000) | 180p (thr=500)
  Set 2 (bob):   720p (thr=2000) | 360p (thr=1000) | 180p (thr=500)
  Set 3 (carol): 720p (thr=2000) | 360p (thr=1000) | 180p (thr=500)
  Set 4 (dave):  720p (thr=2000) | 360p (thr=1000) | 180p (thr=500)
~~~

The relay runs the DTS allocation algorithm to pick ONE
track per switching set based on the bandwidth budget. DTS
switches at group boundaries, ensuring no decoder disruption.

### Congestion Adaptation

DTS provides layered congestion defense:

1. Publisher priority: network drops higher-bitrate streams
   first (720p at pri=3 dropped before 180p at pri=1).

2. DTS algorithm: relay re-runs bandwidth allocation,
   switching each source to a lower rendition.

3. Rank: if bandwidth is still insufficient, lower-rank
   switching sets are starved first (least-loud speakers
   lose quality or get dropped).

No subscriber signaling is needed — the relay handles
adaptation autonomously.

## Top-N and DTS: Example {#top-n-dts-example}

Consider a 10-person meeting where the subscriber displays
a 2x2 grid (top-4 speakers). Each publisher sends 3
simulcast video renditions: 720p (2 Mbps), 360p (1 Mbps),
180p (500 kbps).

### Step 1: Subscription

The subscriber sends 3 SUBSCRIBE_TRACKS (one per rendition)
each with TOP_TRACKS_FILTER{LOUDNESS, MaxTopTracks=4}.

### Step 2: Top-N Evaluation

Objects arrive interleaved from all publishers. The relay
evaluates LOUDNESS per track:

~~~
  C: L=90, H: L=88, A: L=82, E: L=78
  D: L=72, F: L=75, B: L=60, ...
~~~

Top-4 by LOUDNESS: C, H, A, E. The relay sends PUBLISH
for these 4 sources' tracks (12 total: 4 sources x 3
renditions).

### Step 3: DTS Quality Selection

The subscriber assigns switching sets and the relay picks
one quality per set. With a budget of ~1500 kbps per
source, DTS selects 360p for each:

~~~
  Forwarded: 360p/C, 360p/H, 360p/A, 360p/E
  Total: ~4 Mbps downstream
~~~

Without Top-N + DTS, the subscriber would receive all 30
video tracks (~60 Mbps).

### Step 4: Speaker Change

When J starts speaking (L=95, starting a new group), the
relay re-evaluates and J displaces E (L=78). The relay
sends REQ_UPDATE{E, FWD=0} and PUBLISH{J, FWD=1}. DTS
assigns J to the switching set that E vacated. The switch
is group-aligned, giving the subscriber's decoder a clean
keyframe.

### Only 2 Speakers Active

When only C (L=90) and A (L=82) are speaking and others
are silent (L<10), each rendition subscription still
selects the same 2 sources. The subscriber gets 2 active
grid slots; the remaining 2 are empty until more
participants speak.

# Catalog Format {#catalog-format}

The meeting catalog uses the MOQT Streaming Format (MSF)
catalog structure. Each device publishes an MSF catalog
track describing its available media renditions. The
catalog track MUST have a track name of "catalog" within
the catalog namespace.

## Catalog Object Structure

Each catalog object is a JSON document following the MSF
catalog schema. The catalog describes the tracks being
published by the device, their codecs, qualities, and
relationships:

~~~json
{
  "version": 1,
  "generatedAt": 1715788800000,
  "tracks": [
    {
      "name": "opus48k",
      "namespace": ["mocha_v1", "example.com", "acme",
                    "media", "eng", "standup",
                    "mtg-2024-05-15", "audio", "alice"],
      "packaging": "loc",
      "role": "audio",
      "codec": "opus",
      "samplerate": 48000,
      "channelConfig": "mono",
      "bitrate": 48000,
      "label": "Alice Smith"
    },
    {
      "name": "video",
      "namespace": ["mocha_v1", "example.com", "acme",
                    "media", "eng", "standup",
                    "mtg-2024-05-15", "720p", "alice"],
      "packaging": "loc",
      "role": "video",
      "codec": "avc1.64001f",
      "width": 1280,
      "height": 720,
      "framerate": 30,
      "bitrate": 2000000,
      "altGroup": 1,
      "label": "Alice Smith"
    },
    {
      "name": "video",
      "namespace": ["mocha_v1", "example.com", "acme",
                    "media", "eng", "standup",
                    "mtg-2024-05-15", "360p", "alice"],
      "packaging": "loc",
      "role": "video",
      "codec": "avc1.64001f",
      "width": 640,
      "height": 360,
      "framerate": 30,
      "bitrate": 500000,
      "altGroup": 1,
      "label": "Alice Smith"
    },
    {
      "name": "video",
      "namespace": ["mocha_v1", "example.com", "acme",
                    "media", "eng", "standup",
                    "mtg-2024-05-15", "180p", "alice"],
      "packaging": "loc",
      "role": "video",
      "codec": "avc1.64001f",
      "width": 320,
      "height": 180,
      "framerate": 15,
      "bitrate": 200000,
      "altGroup": 1,
      "label": "Alice Smith"
    }
  ]
}
~~~

Tracks that share an altGroup value represent alternate
renditions of the same content (simulcast). Each rendition
lives in its own namespace, enabling prefix-based
SUBSCRIBE_TRACKS with Top-N filtering per rendition. DTS
uses switching sets to select one rendition per source
based on available bandwidth.

## Catalog Updates

When a device changes its published renditions (e.g., starts
screen sharing or changes video quality), it publishes a new
catalog object in the next group. Delta updates MAY be used
for incremental changes. Subscribers to the catalog track
receive the updated rendition list and adjust their media
subscriptions accordingly.

# Basic Meeting Flow {#meeting-flow}

## Meeting Creation

A meeting is created when the first participant initiates
a call in a channel. The MeetingID is generated and
associated with the channel. Meeting creation is signaled
via a system message on the channel's chat track as defined
in {{MOCHA-CHAT}}, allowing all channel participants to
discover and join the meeting.

## Joining a Meeting {#join-flow}

When a participant joins an existing meeting, the following
sequence occurs. In this example, the catalog namespace
prefix is (..., "catalog", "eng", "standup", "mtg-2024-05-15")
and the media namespace prefix is
(..., "media", "eng", "standup", "mtg-2024-05-15").

~~~
  Alice                       Relay                  Bob (existing)
    |                           |                        |
    |  1. SUBSCRIBE_NAMESPACE   |                        |
    |     catalog prefix        |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  2. NAMESPACE             |                        |
    |     (...,"catalog","bob") |                        |
    |<--------------------------|                        |
    |                           |                        |
    |  3. SUBSCRIBE             |                        |
    |     Track: "catalog"      |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  4. Catalog object        |                        |
    |     (Bob's renditions)    |                        |
    |<--------------------------|                        |
    |                           |                        |
    |  5. 3x SUBSCRIBE_TRACKS   |                        |
    |     (...,"720p")+TOP_N{4} |                        |
    |     (...,"360p")+TOP_N{4} |                        |
    |     (...,"180p")+TOP_N{4} |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  6. PUBLISH (top-N)       |                        |
    |     (...,"720p","bob")    |                        |
    |     (...,"360p","bob")    |                        |
    |     (...,"180p","bob")    |                        |
    |<--------------------------|                        |
    |                           |                        |
    |  7. PUBLISH_OK + SET      |                        |
    |     {bob=1}               |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  8. PUBLISH media (alice) |                        |
    |     (...,"720p","alice")  |                        |
    |     (...,"360p","alice")  |                        |
    |     (...,"180p","alice")  |                        |
    |     + catalog             |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  9. Media objects (DTS    |                        |
    |     selected quality)     |                        |
    |<------------------------->|<---------------------->|
    |                           |                        |
~~~

The join sequence in detail:

1. Alice sends SUBSCRIBE_NAMESPACE for the catalog
   namespace prefix to discover existing participants.

2. The relay responds with NAMESPACE messages for each
   catalog track currently published in the meeting.

3. Alice sends SUBSCRIBE for each discovered catalog
   track to learn available renditions.

4. Alice receives MSF catalog objects describing each
   participant's media tracks and renditions.

5. Alice sends 3 SUBSCRIBE_TRACKS (one per rendition
   prefix) each with TOP_TRACKS_FILTER on LOUDNESS.

6. The relay evaluates LOUDNESS and sends PUBLISH for
   the top-N sources' tracks across all renditions.

7. Alice responds with PUBLISH_OK including switching
   set assignments. DTS selects one quality per source.

8. Alice publishes her own media tracks and catalog,
   making her media available to other participants.

9. Bidirectional media flows through the relay, with
   DTS selecting the appropriate quality per source.

## Active Speaker Switching {#active-speaker}

Active speaker switching uses the Top-N Track Filter and
DTS mechanisms described in {{top-n}} and {{dts-ssts}}.
The following shows a speaker change where J (L=95)
displaces E (L=78):

~~~
  Subscriber          Relay                Publishers
    |                   |
    |                   |<-- obj (...,720p,J) L=95 (new group)
    |                   |
    |                   |  Re-eval top-4:
    |                   |  J=95, C=90, H=88, A=82
    |                   |  E=78 displaced
    |                   |
    |  REQ_UPDATE       |
    |  (...,*,E) FWD=0  |
    |<------------------|
    |                   |
    |  PUBLISH          |
    |  (...,*,J) FWD=1  |
    |<------------------|
    |                   |
    |  PUBLISH_OK +     |
    |  SET{J replaces E}|
    |------------------>|
    |                   |
    |  Objects:         |
    |  360p,J (DTS)     |
    |<------------------|
    |                   |
~~~

The promoted source (J) started a new group when it began
speaking, so the relay forwards from a clean group boundary
with a keyframe. The demoted source (E) completes its
current group before forwarding stops — no truncated data.

## Screen Sharing

A participant initiates screen sharing by publishing
additional tracks under a "screen" rendition namespace
and updating their catalog:

~~~
  Alice (sharing)              Relay              Others
    |                            |                  |
    | Update catalog:            |                  |
    |  + (...,"1080p","alice")   |                  |
    |    Track: "screen"         |                  |
    |--------------------------->|                  |
    |                            |                  |
    | Publish screen video       |                  |
    |--------------------------->|  Notify catalog  |
    |                            |  subscribers     |
    |                            |----------------->|
    |                            |                  |
    |                            |  SUBSCRIBE       |
    |                            |  (...,"1080p","alice")
    |                            |  Track: "screen" |
    |                            |<-----------------|
    |                            |                  |
    | Screen frames              |  Fan-out         |
    |--------------------------->|----------------->|
    |                            |                  |
~~~

Screen sharing tracks are not subject to Top-N filtering;
subscribers explicitly subscribe to them via direct
SUBSCRIBE after discovering them through catalog updates.

## Late Join {#late-join}

When a participant joins a meeting already in progress, it
needs to synchronize to the current media state. The
SUBSCRIBE_OK response includes the current live edge (the
latest Group ID being published), which the subscriber uses
to determine where to begin playback.

For video tracks, the subscriber chooses between two
strategies based on how far into the current group the
publisher has progressed:

1. FETCH from current group: If the publisher is not too
   far into the current group (i.e., few objects have been
   published since the last keyframe), the subscriber
   issues a FETCH request starting from the beginning of
   the current group (Object 0). The relay delivers the
   keyframe and subsequent frames from its cache, allowing
   the subscriber to catch up quickly and join the live
   stream mid-GOP.

2. DYNAMIC_GROUPS: If the publisher is deep into the
   current group (many frames since the last keyframe),
   fetching from the beginning would require downloading
   a large backlog. Instead, the subscriber sends a
   DYNAMIC_GROUPS to the publisher, which triggers the
   publisher to end the current group and start a new one
   with a fresh keyframe. The subscriber then receives
   the new keyframe immediately as live media.

The live edge information from SUBSCRIBE_OK (the current
Group ID and Object ID) allows the subscriber to determine
how far into the current group the publisher is and choose
the appropriate strategy.

~~~
  Case 1: FETCH (publisher early in current group)

  Subscriber                  Relay                Publisher
    |                           |                      |
    |  SUBSCRIBE video track    |                      |
    |-------------------------->|                      |
    |                           |                      |
    |  SUBSCRIBE_OK             |                      |
    |  (live edge: G=5, O=2)   |                      |
    |<--------------------------|                      |
    |                           |                      |
    |  Decision: O=2, only 2    |                      |
    |  frames in, FETCH is fast |                      |
    |                           |                      |
    |  FETCH (G=5, O=0)        |                      |
    |-------------------------->|                      |
    |                           |                      |
    |  Object G=5/O=0 (I-frame)|                      |
    |<--------------------------|                      |
    |  Object G=5/O=1 (P-frame)|                      |
    |<--------------------------|                      |
    |  Object G=5/O=2 (P-frame)|  (live continues)    |
    |<--------------------------|<---------------------|
    |                           |                      |
    |  (now in sync, receives   |                      |
    |   live objects G=5/O=3...) |                      |
    |<--------------------------|<---------------------|
    |                           |                      |

  Case 2: DYNAMIC_GROUPS (publisher deep in current group)

  Subscriber                  Relay                Publisher
    |                           |                      |
    |  SUBSCRIBE video track    |                      |
    |-------------------------->|                      |
    |                           |                      |
    |  SUBSCRIBE_OK             |                      |
    |  (live edge: G=5, O=47)  |                      |
    |<--------------------------|                      |
    |                           |                      |
    |  Decision: O=47, too far  |                      |
    |  to fetch from O=0       |                      |
    |                           |                      |
    |  DYNAMIC_GROUPS        |                      |
    |-------------------------->|--------------------->|
    |                           |                      |
    |                           |  Publisher ends G=5  |
    |                           |  Starts G=6 with    |
    |                           |  fresh keyframe     |
    |                           |<---------------------|
    |                           |                      |
    |  Object G=6/O=0 (I-frame)|                      |
    |<--------------------------|                      |
    |  Object G=6/O=1 (P-frame)|                      |
    |<--------------------------|<---------------------|
    |                           |                      |
    |  (in sync from fresh      |                      |
    |   keyframe, minimal delay)|                      |
    |<--------------------------|<---------------------|
    |                           |                      |
~~~

## Leaving a Meeting

When a participant leaves a meeting:

1. The device stops publishing media objects on its tracks.
2. The device publishes a final catalog object with an empty
   rendition list or a "departed" status.
3. The relay detects track inactivity and stops forwarding.
4. Other participants observe the catalog update and
   unsubscribe from the departed device's media tracks.

# Relay Behavior {#relay}

## Fan-Out

When multiple participants subscribe to the same media track,
the relay performs fan-out delivery from a single upstream
subscription. This prevents duplicate traffic to the
original publisher.

## Caching

Relays cache recent video keyframes (the most recent complete
group) to enable fast joins. When a new subscriber arrives,
the relay delivers the cached keyframe immediately, providing
instant video without waiting for the next keyframe from the
publisher.

## Self-Suppression

Relays MUST NOT deliver a participant's own media back to
them. The relay supresses self-delivery based on
FullTrackName and Track alias.

## Top-N Track Filtering and DTS

Relays implement Top-N Track Filtering ({{top-n}}) and
DTS/SSTS ({{dts-ssts}}) to select which sources and what
quality to forward. The relay evaluates LOUDNESS metadata
on incoming objects to maintain the top-N set, sends
PUBLISH/REQ_UPDATE to adjust the subscriber's view, and
runs the DTS bandwidth allocation algorithm to pick one
rendition per switching set. All switching occurs at group
boundaries for clean decoder transitions.

# Security Considerations

TODO

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO
