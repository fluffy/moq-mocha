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

informative:
  RFC3550:
  RFC8825:

--- abstract

This document specifies how multi-party audio and video
meetings are conducted using MOCHA (MoQ Open Communication &
Hosting Architecture). It defines namespace conventions, media
track design, catalog distribution, and participant flows for
real-time conferencing over MoQ Transport (MOQT). This
specification supports both simulcast and scalable video codec
configurations, enabling meetings ranging from small group
calls to large-scale conferences.

--- middle

# Introduction

Real-time meetings are a core use case for MOCHA. This
document specifies how participants join, publish, and
subscribe to audio and video media in a meeting using MOQT's
publish/subscribe primitives.

Each participant's device publishes its media on per-device
tracks within a meeting namespace. A catalog track provides
metadata about available media renditions, enabling
subscribers to discover and select appropriate streams.
Active speaker detection and relay-based fan-out allow
meetings to scale efficiently.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses terminology and architecture defined in
{{MOCHA-ARCH}}, including Provider, Organization, Team,
Channel, and Device. The MOQT transport primitives
(Track, Group, Object, Namespace, Relay) are defined in
{{MoQTransport}}. HClientID and HDevID are defined in
{{MOCHA-IDENTITY}}.

The following additional terms are used:

MeetingID:
: A unique identifier for a meeting instance within a
  channel. A new MeetingID is assigned each time a meeting
  is started in a channel.

RenditionID:
: An identifier for a specific media encoding variant
  (e.g., resolution, bitrate, codec profile).

Catalog:
: An MSF {{MSF}} catalog track describing the available
  media renditions published by a device.

Active Speaker:
: The participant currently producing the most significant
  audio activity, as determined by the audio_level object
  property and Top-N Track Filter selection.

# Meeting Namespace {#meeting-namespace}

## Identifier Derivation {#id-derivation}

The HClientID and HDevID used in namespace tuples and
track names are derived per {{MOCHA-IDENTITY}},
Section "Identifier Derivation".

## Media Namespace

The media namespace groups all real-time audio and video tracks
for a single meeting instance. All devices participating in
the same meeting publish and subscribe to tracks under this
common namespace, allowing the relay to perform fan-out,
caching, and track filtering scoped to that meeting.

Media tracks for a meeting use the following namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "media",
   <TeamID>, <ChannelID>, <MeetingID>, <HClientID>)
~~~

For example, a meeting in the "standup" channel of team
"engineering" at organization "acme":

~~~
  ("mocha_v1", "example.com", "acme", "media",
   "engineering", "standup", "meeting-2024-05-15-1600",
   <HClientID>)
~~~

## Media Track Names

Each device publishes media on tracks within the meeting
namespace. Track names encode the media type, rendition,
and device identity:

Audio tracks:

~~~
  audio-<RenditionID>-<HDevID>
~~~

Video tracks:

~~~
  video-<RenditionID>-<HDevID>
~~~

HClientID and HDevID are derived per {{id-derivation}}.

For example, if Alice's device has HDevID "alice-laptop-1"
and publishes two video renditions:

~~~
  Namespace: ("mocha_v1", "example.com", "acme", "media",
              "engineering", "standup", "meeting-2024-05-15-1600",
              <HClientID>)

  Track: audio-opus48k-<HDevID>
  Track: video-720p-<HDevID>
  Track: video-360p-<HDevID>
~~~

The RenditionID values used in track names (e.g., "opus48k",
"720p", "360p") correspond to rendition entries in the
device's catalog (see {{catalog-format}}). Subscribers
consult the catalog to discover which renditions are
available before subscribing to specific media tracks.

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
   <TeamID>, <ChannelID>, <MeetingID>, <HClientID>)
~~~

HClientID and HDevID are derived per {{id-derivation}}.

## Catalog Track Names

Each device publishes a catalog describing its available
renditions:

~~~
  catalog-<HDevID>
~~~

For example:

~~~
  Namespace: ("mocha_v1", "example.com", "acme", "catalog",
              "engineering", "standup", "meeting-2024-05-15-1600",
              <HClientID>)
  Track: catalog-<HDevID>
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
  Track: video-720p-alice-laptop-1
    |
    +-- Group 1 (GOP / keyframe boundary)
    |   +-- Object 0: I-frame
    |   +-- Object 1: P-frame
    |   +-- Object 2: P-frame
    |   ...
    |
    +-- Group 2 (next GOP)
    |   +-- Object 0: I-frame
    |   ...
~~~

* Each encoded video frame is packaged as a LOC container
  and placed in a single MOQT Object
* Group ID increments at each keyframe boundary (GOP). The
  first Group ID is the lower 32 bits of the current wall
  clock time in milliseconds
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

## Active Speaker Detection via Top-N Track Filters

Active speaker detection leverages MOQT's Top-N Track Filter
mechanism on SUBSCRIBE_TRACKS. Rather than requiring a
separate speaking indicator track, each audio track includes
an audio_level object property (0-127) representing the
current voice activity level.

Subscribers issue a SUBSCRIBE_TRACKS with a TRACK_FILTER
parameter selecting the top-N audio tracks by audio_level.
The relay monitors audio_level properties across all audio
tracks in the namespace and automatically promotes tracks
that enter the top-N (sending PUBLISH messages for them)
and demotes tracks that fall out (ceasing delivery). A
configurable timeout (TrackFilterTimeout, default 1 second)
provides damping to avoid rapid switching during brief
interjections.

This approach eliminates the need for dedicated speaking
indicator tracks, as the relay performs active speaker
selection natively using the audio_level property on the
media tracks themselves.

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
      "name": "audio-opus48k-alice-laptop-1",
      "namespace": ["mocha_v1", "example.com", "acme",
                    "media", "engineering", "standup",
                    "meeting-2024-05-15-1600"],
      "packaging": "loc",
      "role": "audio",
      "codec": "opus",
      "samplerate": 48000,
      "channelConfig": "mono",
      "label": "Alice Smith"
    },
    {
      "name": "video-720p-alice-laptop-1",
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
      "name": "video-360p-alice-laptop-1",
      "packaging": "loc",
      "role": "video",
      "codec": "avc1.64001f",
      "width": 640,
      "height": 360,
      "framerate": 30,
      "bitrate": 500000,
      "altGroup": 1,
      "label": "Alice Smith"
    }
  ]
}
~~~

Tracks that share an altGroup value represent alternate
renditions of the same content (simulcast). Subscribers
select one track from each alternate group based on
available bandwidth and layout requirements.

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
sequence occurs. In this example, the catalog namespace is
("mocha_v1", "example.com", "acme", "catalog", "engineering",
"standup", "meeting-2024-05-15-1600") and the media namespace
is ("mocha_v1", "example.com", "acme", "media", "engineering",
"standup", "meeting-2024-05-15-1600").

~~~
  Alice                       Relay                  Bob (existing)
    |                           |                        |
    |  1. SUBSCRIBE_NAMESPACE   |                        |
    |     catalog namespace     |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  2. NAMESPACE             |                        |
    |     (catalog-bob-desktop) |                        |
    |<--------------------------|                        |
    |                           |                        |
    |  3. SUBSCRIBE             |                        |
    |     catalog-bob-desktop   |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  4. Catalog object        |                        |
    |     (Bob's renditions)    |                        |
    |<--------------------------|                        |
    |                           |                        |
    |  5. SUBSCRIBE_TRACKS      |                        |
    |     media namespace       |                        |
    |     (Top-N audio_level)   |                        |
    |-------------------------->|                        |
    |                           |                        |
    |  6. PUBLISH (from relay)  |                        |
    |     audio-opus48k-bob-desktop                      |
    |     video-720p-bob-desktop|                        |
    |<--------------------------|                        |
    |                           |                        |
    |  7. PUBLISH media         |                        |
    |     audio-opus48k-alice-laptop-1                   |
    |     video-720p-alice-laptop-1                      |
    |     catalog-alice-laptop-1|                        |
    |-------------------------->|                        |
    |                           |                        |
    |  8. Media objects flow    |                        |
    |<------------------------->|<---------------------->|
    |                           |                        |
~~~

The join sequence in detail:

1. Alice sends SUBSCRIBE_NAMESPACE for the catalog
   namespace to discover existing participants.

2. The relay responds with NAMESPACE messages for each
   catalog track currently published in the meeting
   (e.g., catalog-bob-desktop).

3. Alice sends SUBSCRIBE for each discovered catalog
   track to learn available renditions.

4. Alice receives MSF catalog objects describing each
   participant's media tracks and renditions.

5. Alice sends SUBSCRIBE_TRACKS for the media namespace
   with a Top-N Track Filter on audio_level. This requests
   the relay to establish subscriptions for the top-N
   active audio/video tracks.

6. The relay responds by sending PUBLISH messages for the
   matching tracks (e.g., Bob's audio and video), then
   begins delivering media objects.

7. Alice publishes her own media tracks and catalog,
   making her media available to other participants.

8. Bidirectional media flows through the relay.

## Active Speaker Switching {#active-speaker}

Active speaker detection uses the audio_level object
property on audio tracks. The relay monitors these values
and selects the top-N tracks for delivery based on the
SUBSCRIBE_TRACKS filter:

~~~
  Alice             Relay              Bob's View
    |                 |
    | audio_level: 95 |                 +-----------+
    |---------------->|  Top-N select   | Alice(AS) |
    |                 |  by audio_level | Carol     |
    | Bob level: 20   |                 | Dave      |
    |                 |                 +-----------+
    |                 |
    | audio_level: 110|
    |---------------->|  Alice exceeds  +-----------+
    |                 |  threshold for  |*Alice*    |
    |                 |  >1 sec         | Bob       |
    |                 |                 | Carol     |
    |                 |                 +-----------+
~~~

Subscribers use SUBSCRIBE_TRACKS with a Top-N Track Filter
to request the top-N media streams ranked by audio_level:

~~~
  SUBSCRIBE_TRACKS {
    Prefix: ("mocha_v1", "example.com", "acme", "media",
             "engineering", "standup", "meeting-2024-05-15-1600")
    Parameters:
      TRACK_FILTER: {
        Property: audio_level
        MaxTracksSelected: 4
      }
  }
~~~

The relay selects the top-N audio tracks and delivers only
those to the subscriber. When the active speaker changes,
the relay sends PUBLISH for the newly promoted track and
ceases delivery of the demoted track.

## Screen Sharing

A participant initiates screen sharing by publishing
additional video rendition tracks and updating their
catalog:

~~~
  Alice (sharing)              Relay              Others
    |                            |                  |
    | Update catalog:            |                  |
    |  + video-screen-1080p-alice-laptop-1          |
    |--------------------------->|                  |
    |                            |                  |
    | Publish screen video       |                  |
    |--------------------------->|  Notify catalog  |
    |                            |  subscribers     |
    |                            |----------------->|
    |                            |                  |
    |                            |  SUBSCRIBE       |
    |                            |  video-screen-*  |
    |                            |<-----------------|
    |                            |                  |
    | Screen frames              |  Fan-out         |
    |--------------------------->|----------------->|
    |                            |                  |
~~~

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

## Track Filtering

Relays support track filtering based on properties like
speaking level, enabling active speaker selection without
requiring subscribers to receive all tracks and filter
locally.

# Security Considerations

TODO

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO
