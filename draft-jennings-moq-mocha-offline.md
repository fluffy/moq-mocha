---
title: "MOCHA Ferry: Offline Storage and Synchronization"
abbrev: "mocha-offline"
category: std

docname: draft-jennings-moq-mocha-offline-00
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
 - Offline
 - Storage
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
  MOCHA-ARCH: I-D.draft-jennings-moq-mocha-arch
  MOCHA-CHAT: I-D.draft-jennings-moq-mocha-chat
  MOCHA-IDENTITY: I-D.draft-jennings-moq-mocha-identity

informative:
  MOCHA-MLS: I-D.draft-jennings-moq-mocha-mls-keying
  MOCHA-MEETINGS: I-D.draft-jennings-moq-mocha-meetings


--- abstract

This document specifies offline operation for MOCHA (MoQ Open
Communication & Hosting Architecture). It defines the Mocha
Ferry, an agent with durable storage that runs either on the
device, keeping it working when the network is unreachable, or
in the network, holding content for the user while their
devices are offline. It also defines a track property that lets
publishers declare how their tracks may be stored and replayed.

--- middle

# Introduction

MOCHA is designed so that basic chat and meetings continue to
work without access to the home server. While operations that
mutate organizational state (creating users, groups, changing
permissions or membership) require the home server to be
reachable, reading and sending messages within existing channels
operates over MoQ relays and the Ferry.

This document specifies the Mocha Ferry, an agent that stores
content of interest to a user and serves it to the user's
devices. On the device, the Ferry provides offline continuity
when upstream connectivity is lost. The home server can run
behind a NAT; the Ferry handles reconnection and resynchronization
when the network path is restored. The name follows message
ferrying in delay-tolerant networking, where a node stores
content and carries it across a gap in connectivity.

Disconnection also happens in the other direction: the
network stays up, but the user's device goes offline because
it is asleep, out of coverage, or not running. A Ferry placed
in the network rather than on the device addresses this case.
It keeps the user's subscriptions running while all of the
user's devices are offline, persists what arrives, and serves
it when a device returns, so the device can catch up and then
continue live from the same session ({{network-ferry}}).

The Ferry's store is durable, not a best-effort cache. It
survives restarts, and a Network Ferry may hold the only
remaining copy of content that relays and the Persistence
Service {{MOCHA-ARCH}} have already discarded.

A key constraint is that authorization tokens issued by the home
server will eventually expire. If the home server is offline for
too long, the device loses the ability to authenticate to relays.
The Ferry mitigates this by serving stored content locally
without requiring relay authentication.

Not all MOCHA tracks are safe to replay from storage. Message
history is immutable and can be replayed at any time, but
tracks that carry current state, such as availability, typing
indicators, or the MLS committer keep-alive {{MOCHA-MLS}},
would mislead the application if replayed after the fact. This
document therefore defines the MOCHA_STORE_POLICY track
property ({{store-policy}}), with which publishers declare how
a track may be stored and replayed.

# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology defined in {{MOCHA-ARCH}}. The
MOQT primitives (Track, Group, Subgroup, Object, Namespace,
Full Track Name, Relay) are defined in {{MoQTransport}}.

The following additional terms are used:

Ferry:
: The agent defined in this document.

Store:
: The Ferry's durable storage for objects and queued
  publications ({{storage-model}}).

Device-Local Ferry:
: A Ferry that runs on the user's device or on a local network
  appliance, and covers loss of connectivity between the
  device and the network.

Network Ferry:
: A Ferry that runs in the network, stays connected while the
  user's devices are offline, and covers loss of connectivity
  of the devices themselves ({{network-ferry}}).

Durable Interest:
: An interest that a Network Ferry keeps current whether or
  not any of the user's devices is connected
  ({{durable-interest}}).

Upstream Session:
: An MOQT session from the Ferry to a MOCHA relay, or from a
  Device-Local Ferry to a Network Ferry.

Local Session:
: An MOQT session between the Ferry and an application on a
  device that the Ferry serves.

Interest:
: A namespace prefix or object range that the local device
  has requested and that the Ferry keeps current.

Complete Group:
: A stored group for which the Ferry holds every object up to
  and including an End of Group marker, or every object up to
  the largest Object ID announced for the group.

Partial Group:
: A stored group that is not complete, for example because
  connectivity was lost while the group was being received.

# Naming and Scope {#naming-scope}

## Use of MOCHA Names

The Ferry uses MOCHA namespaces and track names exactly as
defined by the publishing specifications (for example,
{{MOCHA-CHAT}} and {{MOCHA-MEETINGS}}). It does not rewrite,
prefix, or map names. An object stored from the Upstream
Session under a given Full Track Name is served on the Local
Session under the same Full Track Name, Group ID, Subgroup ID,
and Object ID.

This keeps the Ferry transparent: an application uses the same
names whether it is connected to the Ferry or directly to a
relay. It also means a local subscriber cannot tell from the
name alone whether an object came from the store or from the
network. Tracks for which that difference matters declare it
with MOCHA_STORE_POLICY ({{store-policy}}).

## Interest Is Prefix-Based

MOCHA assigns each publishing device its own track within a
channel namespace (for example, `msg_v1_<HDevID>` in
{{MOCHA-CHAT}}). New tracks appear whenever a participant adds
a device. A list of individual tracks would miss devices that
join later, so the Ferry MUST track interest at the granularity
of namespace prefixes.

MOQT provides two prefix requests {{MoQTransport}}.
SUBSCRIBE_TRACKS delivers a PUBLISH for every track under the
prefix, including tracks created later. SUBSCRIBE_NAMESPACE
only discovers namespaces under the prefix and delivers no
objects. The Ferry:

* MUST treat each prefix received in SUBSCRIBE_TRACKS as a
  content interest, and MUST use SUBSCRIBE_TRACKS upstream to
  keep it current.

* MUST treat each prefix received in SUBSCRIBE_NAMESPACE as a
  discovery interest. It forwards the request upstream and
  remembers the namespaces reported, but does not by itself
  load tracks under the prefix.

When interest prefixes overlap, the longest matching prefix
determines the policy (such as retention) applied to a track.

## Storage Keys {#storage-keys}

The Ferry MUST key stored content on the Full Track Name as a
sequence of namespace fields plus the track name, each encoded
with an explicit length, followed by Group ID, Subgroup ID,
and Object ID. Namespace fields and track names are arbitrary
byte sequences, so implementations MUST NOT key on a joined
string form (such as fields separated by "/"), because
distinct names can join to the same string.

If an implementation uses a hash of the Full Track Name as a
storage key, the hash MUST be collision-resistant, or the
implementation MUST store the full name with each entry and
compare it on lookup.

## Scope of Service

The Ferry serves only the device or devices belonging to its
configured user. It acts as a relay for that user alone and
MUST NOT serve content to other users.

When serving from its store, the Ferry MUST NOT serve a Full
Track Name unless it is covered by a subscribe scope of the
most recent C4M token the Ferry held for that user
{{MOCHA-IDENTITY}}. This check applies even after the token
has expired. It preserves the access decision the home server
made last, so that losing connectivity never widens what a
device can read.

# The Mocha Ferry

The Mocha Ferry is an agent with a durable store that operates
as both a subscriber and a publisher in the MOQT network. The
Ferry has three functional roles: Listen, Load, and Distribute.

A Ferry is deployed in one of two modes:

* A Device-Local Ferry runs on a user's device or on a local
  network appliance. It keeps the device working when the
  network or the home server is unreachable.

* A Network Ferry runs in the network on behalf of one user.
  It keeps the user's content current when the user's devices
  are unreachable ({{network-ferry}}).

Both modes use the same naming, store policy, storage model,
and retention rules defined in this document. Because a Ferry
does not rewrite names ({{naming-scope}}), the modes can be
chained: a Device-Local Ferry MAY use a Network Ferry as its
upstream, in which case the Network Ferry appears to it as a
relay.

The Ferry MUST keep its Upstream Sessions and Local Sessions
separate. Only objects received on an Upstream Session, or
published by the local device for forwarding upstream, are
written to the store ({{storage-loops}}).

## Listen

The Listen role builds the Ferry's interest set by observing
what the local device requests.

The Ferry:

* Listens for all SUBSCRIBE_TRACKS and SUBSCRIBE_NAMESPACE
  requests from the local device and adds their namespace
  prefixes to the interest list ({{naming-scope}}).

* Listens for all SUBSCRIBE and FETCH requests from the local
  device and adds the referenced tracks and object ranges to
  the interest set.

The interest list represents the set of content the local user
cares about. It is built passively from observed device behavior.

## Load

The Load role acquires content from the network and populates
the store.

The Ferry:

* Sends SUBSCRIBE_TRACKS upstream for each content interest
  prefix, and SUBSCRIBE_NAMESPACE for each discovery interest
  prefix.

* Issues FETCH requests for objects in the interest set that are
  not yet stored.

* Resubscribes to all active subscriptions on network change
  (e.g., switching from Wi-Fi to cellular, reconnecting after
  an outage).

* Stores received content according to each track's
  MOCHA_STORE_POLICY ({{store-policy}}).

## Distribute

The Distribute role serves stored content to the local device
and publishes locally-produced content upstream.

The Ferry:

* Answers a local FETCH from the store when the store holds
  every requested object, whether or not upstream connectivity
  is available. MOQT objects are immutable, so a stored object
  is never out of date. If the store does not hold every
  requested object and upstream is reachable, the Ferry fetches
  the missing objects upstream and stores them.

* Forwards local SUBSCRIBE requests upstream when connectivity
  is available. When it is not, the Ferry serves the
  subscription from the store, subject to the track's
  MOCHA_STORE_POLICY. A local SUBSCRIBE_TRACKS is handled the
  same way: when upstream is unreachable, the Ferry sends a
  PUBLISH for each stored track under the prefix whose policy
  is not None.

* Queues locally-published content and forwards it to upstream
  relays when connectivity is available ({{offline-queue}}).

# Store Policy Track Property {#store-policy}

The MOCHA_STORE_POLICY track property tells a Ferry whether the
objects of a track may be stored and replayed from storage.
Publishers SHOULD include this property in the PUBLISH message
(publisher-initiated flow) or SUBSCRIBE_OK (subscriber-initiated
flow), in the same way as MOCHA_COMPRESSION {{MOCHA-CHAT}}.
Other MOCHA components that store objects, such as the
Persistence Service {{MOCHA-ARCH}}, MAY also honor it.

The following values are defined:

| Value | Name    | Meaning |
|:===== |:======= |:======= |
| 0     | None    | Do not store. Never replay. |
| 1     | Latest  | Store only the most recent object. |
| 2     | Persist | Store all objects, subject to retention. |
| 3     | Latest Group | Store only the most recent group. |

None (0):
: The Ferry MUST NOT write objects of the track to its store
  and MUST NOT replay them. Local subscriptions to the track
  are only served while upstream connectivity is available.

Latest (1):
: The Ferry retains only the object with the largest location
  on the track and MAY serve it to a local subscriber or FETCH
  as the last known state. Applications interpret staleness
  using timestamps in the object payload.

Persist (2):
: The Ferry retains all objects of the track, subject to its
  retention policy ({{retention}}), and serves them from the
  store.

Latest Group (3):
: The Ferry retains every object of the most recent group on
  the track and serves that group in full as the last known
  state. Once it holds the first object of a newer group, it
  discards older groups. This policy is for tracks whose
  objects depend on earlier objects in the same group, such as
  a group that starts with a snapshot followed by incremental
  updates. Latest would keep only the final update, which is
  useless without the objects before it.

If the property is absent, the Ferry MUST treat the track as
None (0). Storage is therefore opt-in: a track is never
replayed from storage unless its publisher has declared that
this is safe. If the property is present with an unknown
value, the Ferry MUST treat the track as None (0).

## Recommended Values for MOCHA Tracks

MOCHA publishers SHOULD set MOCHA_STORE_POLICY as follows:

| Track | Specification | Policy |
|:===== |:============= |:====== |
| `msg_v1_<HDevID>` (chat messages) | {{MOCHA-CHAT}} | Persist |
| `user_<HDevID>` (roster) | {{MOCHA-CHAT}} | Latest |
| Channel info tracks | {{MOCHA-CHAT}} | Latest |
| `availability_v1_<HDevID>` | {{MOCHA-CHAT}} | None |
| `typing_v1_<HDevID>` | {{MOCHA-CHAT}} | None |
| `mls_v1_<HDevID>` (proposals, commits) | {{MOCHA-MLS}} | Persist |
| `keypackage_v1_<HDevID>` | {{MOCHA-MLS}} | Latest |
| `keepalive_v1_<HDevID>` | {{MOCHA-MLS}} | None |
| Meeting media tracks | {{MOCHA-MEETINGS}} | None |
| Meeting catalog tracks | {{MOCHA-MEETINGS}} | Latest Group |

The keep-alive track is None because a stored keep-alive
carries a valid MAC for its epoch. Replaying it would make an
absent committer appear live and suppress committer failover
{{MOCHA-MLS}}.

Meeting catalog tracks are Latest Group because a catalog
group can carry incremental updates after its first object
{{MOCHA-MEETINGS}}.

Availability is None because replaying a stored "online"
state while the device is offline is false. An application
that wants to show "last seen" while offline can derive it
from stored Persist tracks such as chat messages.

# Storage Model {#storage-model}

The store MUST be durable: objects and queued publications
written to it survive termination and restart of the Ferry.
The storage backend is an implementation choice. A
Device-Local Ferry typically uses local disk; a Network Ferry
may use a database or an object store. Any backend MUST meet
the requirements of this section.

## Storage Unit

The unit of storage is the (Full Track Name, Group ID,
Subgroup ID) tuple, holding the objects of that subgroup in
Object ID order. Each stored group is either a Complete Group
or a Partial Group. The Ferry MUST record which, because
reconnection depends on it ({{reconnection}}).

## Preserving Object Properties

Stored objects MUST be served exactly as received. The Ferry
MUST preserve, for each object:

* The payload bytes.

* The object status (for example End of Group, End of Track,
  or Object Does Not Exist).

* The publisher priority.

* Any object extension headers.

* Any time-to-live or delivery timeout, including the
  distinction between "none" and zero.

MOCHA payloads are protected end to end {{MOCHA-MLS}}. The Ferry
stores and serves the ciphertext as received and does not
need decryption keys to operate.

## Immutability and Duplicates

The Ferry MUST NOT modify a stored object. When the Ferry
receives an object it already holds, for example during gap
filling after reconnection, it keeps the stored copy. If the
duplicate's payload or status differs from the stored copy,
the publisher has violated MOQT object immutability; the Ferry
SHOULD log the conflict and MUST continue to serve the first
copy.

## Crash Recovery

Mobile operating systems can terminate a Device-Local Ferry at
any time, and a Network Ferry can fail or be restarted. The
Ferry MUST be able to recover a consistent store after
abrupt termination. Objects that were fully written MUST be
recoverable. A group whose writing was interrupted MUST be
recovered as a Partial Group, so that reconnection fills in
the rest. Implementations typically achieve this with
per-record integrity checks and by discarding an incomplete
final record.

# Interest Management

## Building the Interest List

The interest list is populated by observing the device's MOQT
activity:

* Each SUBSCRIBE_TRACKS adds a content interest prefix, and
  each SUBSCRIBE_NAMESPACE a discovery interest prefix, to the
  interest list.

* Each SUBSCRIBE adds a Full Track Name to the interest list.

* Each FETCH adds a specific (Full Track Name, object range)
  to the interest set.

* Interests are deduplicated by namespace prefix, Full Track
  Name, or object range.

## Expiring Old Interests

Over time the interest list accumulates stale entries for
channels the user no longer visits or content that is no longer
relevant. The Ferry MUST provide a mechanism to expire old
interests.

Implementations SHOULD use a least-recently-used policy: if
a namespace has not been accessed by the local device within a
configurable period, it is removed from the interest list and
the Ferry stops subscribing to it upstream. Expiring an
interest makes the stored content eligible for eviction but
does not by itself remove it; removal is governed by the
retention policy ({{retention}}).

A Network Ferry MUST NOT expire a Durable Interest because the
user's devices have not accessed it; the absence of device
activity is the condition a Network Ferry exists to cover.
Durable Interests expire as described in {{durable-interest}}.

## Preventing Storage Loops {#storage-loops}

The Ferry MUST NOT store content that it served to the local
device from its own store, since doing so would create
feedback loops where the Ferry observes its own responses as
new content and stores them redundantly.

Because the Ferry writes to the store only objects received on
an Upstream Session or published by the local device, and
never objects it sends on a Local Session, this loop cannot
occur. Implementations MUST NOT route objects sent on a Local
Session through the store write path.

# Retention and Eviction {#retention}

Interest expiry decides what the Ferry keeps subscribing to.
Retention decides what it keeps in its store. The two are
separate policies.

The Ferry:

* MUST discard an object once its time-to-live, if any, has
  elapsed.

* SHOULD support per-namespace limits on age and on total
  storage size.

* When storage is exhausted, SHOULD evict in this order:
  objects past their time-to-live; then objects of expired
  interests; then the least recently used groups of current
  interests.

* MUST NOT evict entries in the offline publication queue
  ({{offline-queue}}) to reclaim space.

Eviction removes whole groups where possible, so that a group
is either fully stored or absent. A group that is partly
evicted MUST be marked as a Partial Group. For tracks with
policy Latest Group, eviction MUST remove whole groups.

# Offline Publication Queue {#offline-queue}

Messages that the local device publishes while upstream
connectivity is unavailable are held in a publication queue.

The Ferry:

* MUST keep queued publications in its store, so that they
  survive termination and restart of the Ferry.

* MUST forward queued publications upstream in the order they
  were published on each track.

* MUST retry forwarding until the publication is accepted
  upstream, the local application cancels it, or the
  publishing user's authorization for the track is revoked.

* MUST NOT silently drop a queued publication. When forwarding
  is blocked by an expired token ({{token-expiry}}), the
  publication stays queued.

* SHOULD report two states to the local application for each
  publication: stored locally, and accepted upstream. This
  lets a chat client show messages as pending or sent.

* SHOULD make the number of queued publications visible to
  the local application.

The Ferry MUST NOT renumber a queued publication: it keeps the
Group ID and Object ID assigned when it was created. For chat,
the Group ID is the NTP minute of creation {{MOCHA-CHAT}}, so
a message written offline reaches upstream with a Group ID
for a minute that has already passed.

This needs no special handling in MOQT. Each device publishes
chat messages on a track that only it writes
(`msg_v1_<HDevID>`), and the queue is forwarded in order, so a
queued object still has a larger location than every object
already on that track and is delivered normally. Receivers,
however, MUST NOT discard a message because its Group ID is in
the past, and MUST order messages using the previous-message
linkage in {{MOCHA-CHAT}}, not arrival time.

# Network Reconnection {#reconnection}

When the Ferry detects a network change or reconnection after
an outage:

1. It resends SUBSCRIBE_TRACKS and SUBSCRIBE_NAMESPACE for
   all active prefixes in the interest list.

2. For each stored track with policy Persist, it subscribes
   with a Location Filter starting at the Next Object and a
   FILL_PARAMETERS parameter {{MoQTransport}} whose fill range
   covers the objects it missed. For a track whose last stored
   group is a Complete Group, the fill range starts at the
   next group. For a Partial Group, it starts at the object
   after the last one held. The fill range ends at the
   publisher's Largest Object, so the Ferry needs no knowledge
   of how a track assigns Group IDs. For a track announced by
   PUBLISH in response to SUBSCRIBE_TRACKS, the Ferry supplies
   the same filter and fill when accepting the PUBLISH.

3. For each stored track with policy Latest, it fetches the
   most recent object; with policy Latest Group, the most
   recent group.

4. It forwards publications from the offline publication
   queue ({{offline-queue}}) to upstream relays.

5. If authorization tokens have expired, it attempts to refresh
   them from the home server. If the home server is still
   unreachable, the Ferry continues serving stored content
   locally but cannot authenticate to upstream relays.

When a Device-Local Ferry uses a Network Ferry as its upstream,
the FETCH requests in step 2 are answered by the Network Ferry,
which holds content received while the device was offline.
The procedure a device follows when it connects to a Network
Ferry directly is described in {{sync}}.

# Authorization Token Expiry {#token-expiry}

Authorization tokens (C4M tokens per {{MOCHA-IDENTITY}}) are
issued by the home server and have a finite lifetime. The Ferry
operates in degraded mode when tokens expire:

* Local access to the store continues without tokens, limited
  to the scopes of the most recent token ({{naming-scope}}).

* Publishing to upstream relays requires valid tokens,
  publications remain in the offline publication queue until a
  valid token is available.

* The Ferry SHOULD proactively refresh tokens before expiry when
  the home server is reachable.

* The Ferry SHOULD warn the local application when tokens are
  nearing expiry and the home server is unreachable.

# Network-Resident Ferry {#network-ferry}

A Network Ferry runs in the network on behalf of a single user.
It holds Upstream Sessions to MOCHA relays continuously, and
accepts Local Sessions from the user's devices when they are
online. While no device is connected, it continues to receive
and store content for the user's Durable Interests. When a
device reconnects, the device synchronizes from the Network
Ferry and continues live operation over the same session.

A Network Ferry MAY be operated by the user's home Provider or
by the user. One Network Ferry serves all of a user's devices.

## Relationship to the Persistence Service

The Persistence Service {{MOCHA-ARCH}} stores channel history
for a Provider. A Network Ferry differs from it in three ways:

Per-user rather than per-organization:
: A Network Ferry follows everything its user is interested
  in, across Providers and federated organizations
  {{MOCHA-IDENTITY}}. A reconnecting device synchronizes from
  one place instead of from the Persistence Service of each
  Provider involved.

Driven by the user's interest rather than retention policy:
: A Network Ferry keeps what its user cared about, subject to
  its own retention ({{retention}}), independent of how long
  a Provider retains channel history.

Always online:
: A Network Ferry can act while the user's devices cannot. It
  can wake a device when content arrives
  ({{wake-up}}) and can accept publications from a device
  that then goes offline ({{outbox}}).

A Network Ferry MAY obtain missing content from a Persistence
Service with FETCH, like any other upstream.

## Durable Interest {#durable-interest}

A Device-Local Ferry learns interest by observing its device
and expires interest that the device no longer uses. A Network
Ferry cannot rely on observation alone, because the interest
must outlive the device's session.

The Network Ferry:

* MUST record a Durable Interest for each prefix that a device
  of its user requests with SUBSCRIBE_TRACKS, and each track it
  requests with SUBSCRIBE, on a Local Session, and MUST record
  which device registered it.

* MAY additionally derive Durable Interests from the subscribe
  scopes of the user's authorization ({{network-ferry-auth}}),
  for example so that a newly joined channel is followed before
  any device has subscribed to it.

* MUST NOT remove a Durable Interest when a Local Session
  closes. Ending a session is not a withdrawal of interest.

* MUST remove a device's registration of a Durable Interest
  when that device cancels the request {{MoQTransport}} while
  its Local Session remains open, and MUST stop following the
  interest when no registrations remain.

* MUST stop following a Durable Interest that is no longer
  covered by the subscribe scopes of its current token, for
  example because the user left the channel or the token was
  revoked.

* SHOULD expire a device's registrations when that device has
  not connected within a configurable period, so that storage
  is not consumed indefinitely on behalf of a lost device.
  This period SHOULD be long compared to expected offline
  periods.

When a Durable Interest stops being followed, the stored
content becomes eligible for eviction as described in
{{retention}}.

## Credentials {#network-ferry-auth}

A Network Ferry subscribes upstream while none of the user's
devices is online, so it cannot use a device's token. It
authenticates with its own C4M token {{MOCHA-IDENTITY}}:

* The token MUST be issued to the Network Ferry and bound to
  the Network Ferry's own key, not to a device key.

* The token's subscribe scopes MUST NOT exceed the subscribe
  scopes the user would be granted.

* The token MUST NOT carry publish scopes, except as needed
  for the outbox ({{outbox}}). When present, publish scopes
  MUST be limited to the tracks of the user's own devices.

* The Network Ferry MUST refresh its token before expiry, and
  MUST subscribe to the Token Service revocation track
  {{MOCHA-IDENTITY}} and act on revocations that affect its
  token or its user.

* For channels hosted by other Providers, the Network Ferry
  obtains tokens through federated token exchange
  {{MOCHA-IDENTITY}}.

The Network Ferry MUST authenticate each Local Session as
belonging to its user before serving it. The scope rule in
{{naming-scope}} applies: the Network Ferry serves a device
only content covered by its user's current subscribe scopes.

## Synchronization and Resumption {#sync}

A reconnecting device knows the last location it holds on
each track. The Network Ferry does not need to keep a
per-device synchronization position.

On reconnection, the device:

1. Sends SUBSCRIBE_TRACKS for its interest prefixes, which
   refreshes its Durable Interest registrations and delivers a
   PUBLISH for each track, including tracks created while it
   was offline.

2. For each Persist track, obtains the objects after its last
   held location and then continues with live objects. A
   SUBSCRIBE with a Location Filter starting at the Next Object
   and FILL_PARAMETERS {{MoQTransport}} does this in one
   request, without duplicates. For a track the device has never seen,
   it starts from a position chosen by the application, such
   as the most recent groups.

3. For each Latest track, subscribes and receives the most
   recent object; for each Latest Group track, the most recent
   group.

The Network Ferry answers these requests from its store and,
for anything it does not hold, from upstream, as described in
the Distribute role. Live objects that arrive while the device
is synchronizing are delivered on the subscription; the device
discards duplicates by location.

## MLS Continuity

A device that was offline must process every MLS Commit it
missed, in order, before it can decrypt content from later
epochs {{MOCHA-MLS}}. MLS tracks are Persist
({{store-policy}}), so the Network Ferry stores them.

* The Network Ferry SHOULD retain MLS tracks for at least as
  long as the longest offline period it supports for a device.

* A device that finds missed handshake messages unavailable
  cannot advance its epoch and MUST rejoin the group as
  described in {{MOCHA-MLS}}.

* The Network Ferry stores ciphertext only. It MUST NOT be a
  member of any MLS group and MUST NOT hold MLS epoch secrets.

## Outbox {#outbox}

A device can hand publications to its Network Ferry and go
offline immediately, rather than waiting for upstream
acceptance. The Network Ferry then holds them in its offline
publication queue ({{offline-queue}}) and forwards them
upstream under its own token.

Objects in the outbox are end-to-end protected by the
publishing device {{MOCHA-MLS}}. The Network Ferry forwards them
unchanged and MUST NOT originate objects on a device's track.

The Network Ferry SHOULD report the upstream acceptance state of
each outbox publication to the device on its next connection.

## Wake-Up Notifications {#wake-up}

A Network Ferry MAY notify a user's device when new content
arrives for a Durable Interest, using a platform push service.
The user SHOULD be able to choose which interests trigger a
notification.

A push service is a third party. A notification SHOULD carry
no content and no namespace or track names; its purpose is to
prompt the device to connect and synchronize ({{sync}}).

## Media

The recommended store policies in {{store-policy}} apply to a
Network Ferry unchanged. Meeting media tracks are None, so a
Network Ferry does not persist live media. Recording of
meetings is outside the scope of this document.

# Security Considerations

TODO 

# IANA Considerations

TODO

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
