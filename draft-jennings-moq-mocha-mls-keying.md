---
title: "MOCHA MLS Keying: End-to-End Encryption Key Management over MoQ"
abbrev: "mocha-mls-keying"
category: std

docname: draft-jennings-moq-mocha-mls-keying-latest
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
 - MLS
 - E2EE
 - Key Management
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
  MOCHA-IDENTITY: I-D.draft-jennings-moq-mocha-identity
  MOCHA-KEY-BINDING: I-D.draft-jennings-moq-mocha-key-binding
  MOCHA-PAB: I-D.draft-jennings-moq-mocha-pab
  MOCHA-CHAT: I-D.draft-jennings-moq-mocha-chat
  MLS: RFC9420
  SecureObjects: I-D.draft-jennings-moq-secure-objects
  MLS-PARTIAL: I-D.draft-ietf-mls-partial
  MLS-VIRTUAL-CLIENTS: I-D.draft-ietf-mls-virtual-clients

informative:

--- abstract

This document specifies MLS key management for MOCHA (MoQ Open
Communication & Hosting Architecture). It defines how MLS groups
are created, how members are added and removed, how key material
is distributed over MOQT tracks, and how the MLS Delivery
Service is realized in a fully distributed manner using MOQT
publish/subscribe, including committer election, failover, and
fork resolution. For large channels, Partial MLS light clients
avoid requiring every member to store the full group state. An
optional profile represents each user as a single MLS virtual
client shared by the user's devices.

--- middle

# Introduction

Keying for end-to-end encryption in MOCHA is provided by the
Messaging Layer Security (MLS) protocol {{MLS}}. MLS requires a
Delivery Service to distribute protocol messages (KeyPackages,
Commits, Welcome messages) between group members. Traditional
MLS deployments use a centralized server for this role.

This document specifies how MOCHA realizes the MLS Delivery
Service in a fully distributed manner using MOQT's
publish/subscribe primitives. Each member publishes MLS
protocol messages on its own MOQT track, and other members
subscribe to receive them. Ordering is provided by a committer
chosen from among the group members, so the Delivery Service
does not depend on any central server.

For large channels where full MLS group membership becomes
impractical, this document specifies the use of Partial MLS
{{MLS-PARTIAL}} light clients, which participate in a group
without storing the full ratchet tree.

Credentials used within MLS embed the binding chain defined in
{{MOCHA-KEY-BINDING}}, which links device signing keys to user
keys and IdP-issued identity assertions. Trust decisions are
anchored in each member's Personal Address Book {{MOCHA-PAB}},
enabling offline credential validation.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses terminology defined in {{MOCHA-ARCH}} and
{{MLS}}. The following additional terms are used:

MLS Group:
: An MLS group instance associated with a MOCHA channel. All
  members of the channel who participate in E2EE are members
  of this group.

KeyPackage:
: An MLS structure published by a device to advertise its
  willingness to be added to a group. Contains the device's
  public keys and supported ciphersuites.

Commit:
: An MLS message that advances the group epoch. Commits
  carry proposals such as Add, Remove, Update, and
  GroupContextExtensions.

Welcome:
: An MLS message sent to a newly added member containing the
  group state needed to participate from the current epoch.

Epoch:
: A version of the MLS group state. Each Commit advances the
  epoch by one.

Committer:
: The designated group member responsible for collecting
  Proposals and publishing Commits. Only the committer
  advances the group epoch.

Full Client, Light Client:
: A full client stores the complete ratchet tree of a group.
  A light client {{MLS-PARTIAL}} does not; see
  {{partial-mls}}.


# MLS Namespace and Track Structure {#mls-naming}

## MLS Namespace

MLS protocol messages for a channel use the following
namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "mls", <TeamID>,
   <ChannelID>, <MLSParticipantID>)
~~~

KeyPackage advertisements use a separate namespace:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "keypackage", <TeamID>,
   <ChannelID>, <MLSParticipantID>)
~~~

The MLSParticipantID identifies a participant within the
channel's MLS group. It is the participant's HClientID
({{MOCHA-IDENTITY}}) computed for the namespace in which it
appears. With the virtual client profile ({{virtual-clients}}),
the participant is the user's virtual client; otherwise all of
a user's devices share the user's MLSParticipantID and are
distinguished by HDevID in the track name.

## Track Names

Each device publishes MLS protocol messages (Proposals,
Commits, Welcomes) on its own track within the MLS
namespace:

~~~
  mls_v1_<HDevID>
~~~

Each device publishes its KeyPackage on its own track within
the keypackage namespace:

~~~
  keypackage_v1_<HDevID>
~~~

The HDevID is defined in {{MOCHA-IDENTITY}}. Relays enforce,
through the authorization tokens of {{MOCHA-IDENTITY}}, that
only the owning device publishes on a track named with its
HDevID.

## Object Semantics

Each MLS protocol message is published as a single MOQT
object. On the `mls_v1_`, `keepalive_v1_`, `annot_v1_`, and
`tree_v1_` tracks, the MOQT Group ID equals the MLS epoch in
which the object was created: Proposals and keep-alives for
epoch N, and the Commit that moves the group from epoch N to
N+1 together with its Welcome and annotations, are published
in MOQT group N. Objects within a group are numbered
sequentially.

On a `keypackage_v1_` track, each new KeyPackage starts a new
MOQT group, and only the latest object is current.

Proposals and Commits SHOULD be sent as MLS PrivateMessage, so
that relays do not learn the content of membership changes.
KeyPackages and Welcomes use their own MLS wire formats.

## Object Format

Each MOQT object on an `mls_v1_`, `keypackage_v1_`, or
`annot_v1_` track carries a 1-byte type prefix followed by
the raw message bytes:

~~~
  +------+-----------------------------+
  | Type | MLS Message (raw bytes)     |
  | (1B) |                             |
  +------+-----------------------------+
~~~

The Type field values are:

| Value | Description          |
|-------|----------------------|
| 0x01  | Proposal             |
| 0x02  | Commit               |
| 0x03  | Welcome              |
| 0x04  | KeyPackage           |
| 0x05  | Light client annotation ({{partial-mls}}) |

This prefix allows receivers to quickly dispatch messages,
including PrivateMessage handshake messages whose content type
is encrypted. For types 0x01 through 0x04, the MLS Message
field contains the unmodified MLSMessage struct as defined in
{{MLS}}, Section 6. For type 0x05, it contains a light client
annotation as defined in {{MLS-PARTIAL}}. A receiver MUST
discard an object whose Type does not match the decoded
message. Objects on `keepalive_v1_` and `tree_v1_` tracks use
the formats defined in {{keep-alive}} and {{upgrade}}.


# Distributed Delivery Service {#delivery-service}

## Design

Traditional MLS requires a Delivery Service that orders
Commits and ensures all members see the same sequence of
group operations. In MOCHA, this role is split between the
MOQT relay network, which delivers messages, and a committer
chosen from among the group members, which orders them.

MOQT relays already provide:

- Reliable delivery of published objects to all subscribers
- Caching for late-joining members
- Fan-out to all subscribers of a track

By assigning each member its own MLS track within the channel
namespace, the relay network delivers MLS messages without any
server ordering them or having access to plaintext key
material. Relays can still delay, drop, or withhold messages;
see {{security}}.

## Propose/Commit Separation {#propose-commit}

MLS requires that Commits are applied in a consistent order
by all members. MOCHA uses a leader-follower model to
achieve this without a centralized server:

- Any member MAY publish Proposals (Add, Remove, Update) on
  its own MLS track at any time.
- A single designated member (the "committer") collects
  pending Proposals and publishes Commits that advance the
  epoch.
- All other members ("followers") subscribe to the
  committer's track and apply Commits as they arrive.

Members other than the current committer MUST NOT publish
Commits, except as part of committer failover
({{committer-failover}}). In normal operation this yields
exactly one Commit per epoch. Network partitions and
concurrent failover can still produce conflicting Commits for
the same epoch; {{forks}} specifies how members detect and
resolve them.

The committer MAY batch multiple pending Proposals into a
single Commit to minimize epoch churn. For example, if
several Add or Remove Proposals arrive in quick succession,
the committer can coalesce them into one Commit that
advances the epoch once rather than producing a separate
epoch transition for each Proposal.

## Committer Extension {#committer-ext}

The identity of the current committer is part of the group
state, carried in a GroupContext extension:

~~~
  struct {
    opaque committer_id<V>;
  } MochaCommitter;
~~~

The `committer_id` is the committer's HDevID or, when the
committer is a virtual client ({{virtual-clients}}), its
MLSParticipantID. This document calls this value a member's
committer identifier. The extension type is mocha_committer
(TBD3). Groups MUST list mocha_committer in the
`required_capabilities` extension so that every member
supports it.

Because the extension is in the GroupContext, all members
agree on the committer in each epoch, and a new member learns
it from the Welcome.

Full clients signal that they are full clients with an empty
LeafNode extension of type mocha_full_client (TBD4). A member
that changes between full and light client updates its leaf
to add or remove this extension. Because the extension is in
the LeafNode, all full clients agree on which members are
eligible to become committer ({{committer-failover}}).

## Committer Selection {#committer-selection}

### Initial Committer

The Provider designates the initial committer when the
channel is created. The channel metadata returned by the
Provider includes the creator's HDevID as the initial
committer. Since channel creation requires the Provider to
be reachable, the Provider serializes this assignment, and
no race condition is possible. The creator sets the
mocha_committer extension to its own HDevID when it creates
the group.

### Group Re-Bootstrap {#rebootstrap}

If the MLS group requires re-creation (all members have
left, or the group state is explicitly reset), the first
authorized member to join and observe no active keep-alive
assumes the committer role and creates a new group.

Each bootstrap MUST use a fresh `group_id`, so that state from
different bootstraps is never confused:

~~~
  group_id = SHA-256("MOCHA 1.0 group" || channel_namespace ||
                     random_32_bytes)
~~~

where `channel_namespace` is the encoded MLS namespace tuple
prefix up to and including the ChannelID.

If two members bootstrap concurrently, members observe more
than one group for the channel, each identified by the
`group_id` in its keep-alives ({{keep-alive}}). Considering
only groups for which a keep-alive has been received within
twice its advertised interval, all members MUST converge on
the group whose `group_id` is lexicographically lowest. A member in any other group for the
channel, including its bootstrapper, MUST leave that group
and join the winning group by publishing a KeyPackage. A
bootstrapper whose group loses stops acting as committer.

Only devices that the Provider has authorized to publish in
the channel's MLS namespace can bootstrap, which limits this
procedure to channel members.

### Committer Transfer

The current committer MAY designate a successor by
committing a GroupContextExtensions proposal that sets
mocha_committer to the successor's identifier. Because a
GroupContextExtensions proposal replaces the full extension
list, the proposal MUST carry all other GroupContext
extensions unchanged. The successor takes over from the
epoch in which the Commit takes effect. A committer that
intends to leave the group SHOULD transfer the role before
leaving, since MLS does not allow a member to commit its own
removal.

## Committer Failover {#committer-failover}

If the committer becomes unavailable (temporary disconnection
or permanent departure), the group must elect a new committer
to continue processing Proposals.

Unavailability is detected when:

- No valid keep-alive ({{keep-alive}}) is received within
  twice the interval advertised in the most recent valid
  keep-alive.
- A Remove Proposal targeting the committer, from a proposer
  authorized to remove it ({{authorization}}), has not been
  committed within twice the keep-alive interval. This lets
  the group displace a committer that is live but refuses to
  commit, and the new committer then commits the Remove.

When unavailability is detected, the remaining members
elect a new committer using the following deterministic rule:

1. The eligible members are those, other than the current
   committer, that hold a leaf in the current epoch whose
   LeafNode carries the mocha_full_client extension, and that
   have a roster entry with status "active" ({{MOCHA-CHAT}}).
2. The eligible member with the lowest committer identifier
   ({{committer-ext}}, lexicographic ordering) becomes the new
   committer.
3. The new committer publishes a Commit that includes all
   pending Proposals and a GroupContextExtensions proposal
   setting mocha_committer to itself. The Commit MUST include
   an UpdatePath, which refreshes the new committer's own
   leaf.

Full clients accept a failover Commit only from the member
selected by this rule. Because all full clients evaluate the
same rule over the same group state and roster, they arrive at
the same new committer without explicit coordination, unless
they are partitioned ({{forks}}). Light clients cannot
evaluate the rule, since they do not hold all leaves; they
accept a valid failover Commit and learn the new committer
from the GroupContext (see {{light-trust}}). The keep-alive interval SHOULD be
long enough to tolerate transient network disruptions but
short enough to avoid blocking group operations.

## Fork Detection and Resolution {#forks}

A fork occurs when members apply different Commits for the
same epoch, for example when a partition causes each side to
elect its own committer. A member detects a fork when it
observes two different valid Commits for the same epoch of the
same group, or a valid keep-alive for its current epoch from
a committer other than the one recorded in its group state.

On detecting a fork, members resolve it deterministically:

1. The branch whose Commit at the first divergent epoch was
   produced by the committer recorded in the GroupContext of
   the preceding epoch wins.
2. If neither or both satisfy this condition, the branch whose
   committer has the lowest committer identifier wins.

Members on the losing branch MUST discard the state of the
losing branch. A member that retains its state for the last
common epoch returns to it and applies the winning branch's
Commits from that point. Otherwise, it rejoins by publishing
a new KeyPackage; the winning committer removes its stale leaf
and adds it again. Members SHOULD retain the state of the
previous epoch for a short period (for example, a few
keep-alive intervals) to allow the first option, and MUST
delete it afterwards to preserve forward secrecy.

Proposals and application messages sent in losing-branch
epochs are not valid on the winning branch; clients SHOULD
re-send them after rejoining.

## Follower Behavior

Followers subscribe to all MLS tracks in the channel
namespace via SUBSCRIBE_TRACKS. They:

1. Publish Proposals on their own MLS tracks when they
   wish to add, remove, or update members.
2. Apply Commits only from the committer recorded in the
   current GroupContext, or from the member selected by
   failover ({{committer-failover}}).
3. Ignore all other Commits, except as evidence of a fork
   ({{forks}}).

## Committer Keep-Alive {#keep-alive}

The committer periodically publishes keep-alive objects to
prove liveness and current epoch membership. All members
MUST subscribe to the committer's keep-alive track within the
committer's MLS namespace:

~~~
  keepalive_v1_<HDevID>
~~~

### Keep-Alive Object Format

~~~json
{
  "group_id": "<MLS group_id, hex-encoded>",
  "epoch": 5,
  "ts": 1715788800,
  "interval": 10,
  "nonce": "<16 bytes, hex-encoded>",
  "mac": "<HMAC output, hex-encoded>"
}
~~~

The `group_id` identifies the MLS group, so that concurrent
bootstraps can be told apart ({{rebootstrap}}) by members and
by devices that are not yet members. The `interval` field is
the keep-alive interval in seconds. New joiners that FETCH the
latest object learn the failover timeout from it.

The MAC is computed as:

~~~
  keepalive_key = MLS-Exporter("MOCHA 1.0 keepalive", "",
                               KDF.Nh)

  mac = HMAC(keepalive_key,
             group_id ||
             big_endian_uint64(epoch) ||
             big_endian_uint64(ts) ||
             big_endian_uint64(interval) ||
             nonce_bytes)
~~~

where MLS-Exporter is the exporter of the current epoch
({{MLS}}, Section 8.5), and HMAC and KDF.Nh are those of the
group's ciphersuite. A dedicated exporter label keeps this key
separate from the keys MLS uses internally.

### Verification

Upon receiving a keep-alive object, each member:

1. Reads the group_id, epoch, ts, interval, and nonce
   fields, and checks that group_id is that of its group.
2. Computes the HMAC using the keepalive_key for the
   indicated epoch.
3. Compares the computed MAC against the received mac field.
4. If verification succeeds and `ts` is later than that of
   the last valid keep-alive, the committer is confirmed as
   live and in possession of current epoch secrets.
5. If verification fails, the member MUST discard the object.
   An invalid keep-alive MUST NOT by itself trigger failover,
   since a relay or other party could inject one.

A valid MAC shows that the keep-alive was produced by a member
of the current epoch. That it was produced by the committer
follows from relay enforcement of publish authorization on the
committer's track ({{mls-naming}}).

### Timing

The committer MUST publish a keep-alive at the interval it
advertises. Members trigger failover if no valid keep-alive is
received within twice the advertised interval
({{committer-failover}}).

The nonce field (random bytes generated per keep-alive)
ensures each object produces a unique MAC, and members reject
keep-alives whose `ts` does not advance, so that relays cannot
replay a previously observed keep-alive to mask a committer's
failure.

## KeyPackage Distribution

Each device publishes its current KeyPackage on its
keypackage track within the channel's keypackage namespace:

~~~
  Namespace: ("mocha_v1", <Provider>, <OrgID>, "keypackage",
              <TeamID>, <ChannelID>, <MLSParticipantID>)
  Track: keypackage_v1_<HDevID>
~~~

The committer MUST subscribe to the KeyPackage namespace
(SUBSCRIBE_TRACKS) to be notified when new devices
publish KeyPackages. Upon receiving a new KeyPackage, the
committer performs the authorization checks of
{{authorization}} and, if they pass, includes an Add for it
in its next Commit.

Other members MAY also subscribe to the KeyPackage namespace
and publish Add Proposals on their own MLS tracks, but
only the committer produces the Commit that incorporates
the Proposal.

KeyPackages are rotated by publishing a new object on the
track. Each KeyPackage is single-use; after being consumed
in an Add, the device MUST publish a fresh KeyPackage.
Committers MUST use only the latest object on a KeyPackage
track, since relays may serve older cached objects.

## Welcome Message Delivery

When the committer issues a Commit that adds new members, it
publishes the Welcome message as an object on its own MLS
track, in the same MOQT group as the Commit. A joining device
subscribes to the channel's MLS namespace (SUBSCRIBE_TRACKS)
before publishing its KeyPackage, and identifies its Welcome
by finding the KeyPackageRef of its own KeyPackage in the
Welcome's `secrets` ({{MLS}}). No
recipient identifier is exposed to relays.


# Group Lifecycle {#lifecycle}

## Group Creation

When a channel is created, the creator (designated as the
initial committer by the Provider) initializes a new MLS
group:

1. Creator creates an MLS group at epoch 0 with a fresh
   `group_id` ({{rebootstrap}}), a GroupContext containing
   the mocha_committer extension set to its own HDevID, and
   a `required_capabilities` extension listing
   mocha_committer, mocha_full_client, and the credential
   types in use. The creator's LeafNode carries the
   mocha_full_client extension.
2. Creator publishes its initial keep-alive on track
   `keepalive_v1_<HDevID>` within its MLS namespace:
   ("mocha_v1", \<Provider\>, \<OrgID\>, "mls",
   \<TeamID\>, \<ChannelID\>, \<MLSParticipantID\>).
3. Creator subscribes (SUBSCRIBE_TRACKS) to the keypackage
   namespace to receive KeyPackages from new joiners.
4. Creator subscribes (SUBSCRIBE_TRACKS) to the MLS
   namespace to receive Proposals from other members.

Creating an MLS group does not require a Commit; the first
Commit is issued when the first members are added.

## Authorization {#authorization}

The committer MUST check that each change it commits is
authorized:

- Add: the KeyPackage's credential is valid
  ({{credentials}}), and the KeyPackage was published under
  the channel's keypackage namespace, which relays permit only
  for devices the Provider has authorized for the channel
  ({{MOCHA-IDENTITY}}). The committer SHOULD also check the
  new member's user against channel membership policy where it
  is available, and MUST NOT add a device whose key is revoked
  ({{MOCHA-KEY-BINDING}}).
- Remove: the proposer is removing one of its own user's
  leaves, the proposer is permitted by channel policy to
  remove other members (for example, as a channel
  administrator), or the target's device key has been
  revoked.
- Update: the proposer is updating its own leaf, and any
  changed credential satisfies {{credential-rotation}}.

The committer MUST NOT commit a Proposal that fails these
checks. Followers SHOULD apply the same checks to each Commit
and SHOULD alert the user if the committer commits a change
that fails them, since that indicates a misbehaving committer.
When the Provider is unreachable, the committer uses the most
recent channel membership information it has.

## Adding Members

To add a member to the channel:

1. The new device subscribes (SUBSCRIBE_TRACKS) to the
   channel's MLS namespace.
2. The new device publishes (PUBLISH) its KeyPackage on track
   `keypackage_v1_<HDevID>` within the keypackage namespace.
3. The committer, subscribed to the keypackage namespace,
   receives the new KeyPackage via the relay.
4. The committer performs the checks of {{authorization}} and
   creates an MLS Commit that adds the new device.
5. The committer publishes the Commit and the Welcome message
   on track `mls_v1_<HDevID>` within its MLS namespace.
6. All existing members, subscribed to the MLS namespace,
   receive the Commit and advance to the new epoch. Members
   MUST cache received Proposals locally until the
   corresponding Commit from the committer is received.
7. The new member identifies its Welcome, initializes its
   group state at the new epoch, and learns the committer
   from the mocha_committer extension.
8. The new member subscribes (SUBSCRIBE) to the committer's
   keep-alive track `keepalive_v1_<committer-HDevID>`.

Alternatively, any member subscribed to the keypackage
namespace MAY create an Add Proposal and publish it on its
own track `mls_v1_<HDevID>`. The committer incorporates such
Proposals into its next Commit, subject to
{{authorization}}.

## Removing Members

To remove a member:

1. The remover creates an MLS Remove Proposal for the target
   member's leaf index and publishes (PUBLISH) it on track
   `mls_v1_<HDevID>` within the MLS namespace.
2. The committer, subscribed to the MLS namespace, receives
   the Remove Proposal, performs the checks of
   {{authorization}}, and creates an MLS Commit incorporating
   it.
3. The committer publishes the Commit on its track
   `mls_v1_<HDevID>` within the MLS namespace.
4. All remaining members, subscribed to the MLS namespace,
   receive the Commit and advance to the new epoch. Members
   MUST cache received Proposals locally until the
   corresponding Commit from the committer is received.
5. The removed member can no longer decrypt messages sent
   in subsequent epochs.

A Remove of the current committer is committed by its
successor: either the committer transfers the role first, or
the group fails over as described in {{committer-failover}}.

## Key Update

Any member can rotate its own key material:

1. The member creates an MLS Update Proposal with fresh key
   material and publishes (PUBLISH) it on track
   `mls_v1_<HDevID>` within the MLS namespace.
2. The committer, subscribed to the MLS namespace, receives
   the Update Proposal and creates an MLS Commit
   incorporating it.
3. The committer publishes the Commit on its track
   `mls_v1_<HDevID>` within the MLS namespace.
4. All members, subscribed to the MLS namespace, receive the
   Commit and advance to the new epoch with updated key
   material. Members MUST cache received Proposals locally
   until the corresponding Commit from the committer is
   received.

The committer refreshes its own leaf by including an
UpdatePath in a Commit rather than by an Update Proposal.


# Light Clients for Large Channels {#partial-mls}

For channels with many members (hundreds or thousands), full
MLS becomes expensive: every member must store the ratchet
tree and process every Commit, whose cost grows with group
size. Partial MLS {{MLS-PARTIAL}} defines light clients, which
participate in a group without storing the full ratchet tree.
A light client stores only its own leaf and the information
needed to process Commits along its own path.

This section specifies how light clients operate in MOCHA. The
data structures exchanged with light clients, and how light
clients validate them, are defined in {{MLS-PARTIAL}}.

## Full and Light Clients

Each device chooses whether to act as a full client or a light
client in each channel group, based on its resources and on
deployment policy, and signals the choice with the
mocha_full_client LeafNode extension ({{committer-ext}}).
Devices SHOULD act as light clients in channels whose
membership exceeds a deployment-configured threshold (for
example, 100 members), unless deployment policy designates
them as full clients.

The committer MUST be a full client. Deployments SHOULD ensure
that several members of each large channel act as full
clients, so that committer failover ({{committer-failover}})
has eligible candidates. A light client can become a full
client at any time by obtaining the full ratchet tree
({{upgrade}}) and then updating its leaf to carry the
mocha_full_client extension.

Light clients can send and receive application messages and
publish Proposals as usual. They do not produce Commits.

## Annotated Commits

A light client cannot process a plain Commit, because it
lacks the tree information needed to verify the Commit and
locate the path secret encrypted to it. {{MLS-PARTIAL}}
addresses this with annotations: information produced by a
full client that lets a specific light client process the
Commit and verify it against the group's tree hash.

In MOCHA, the committer produces annotations for each Commit.
Since part of an annotation is specific to each receiving
light client, annotations are published on per-recipient
tracks, so that each light client receives only its own. These
tracks, and the tree track of {{upgrade}}, use a namespace
separate from the MLS namespace, so that members subscribing
to the MLS namespace with SUBSCRIBE_TRACKS do not receive
them:

~~~
  Namespace: ("mocha_v1", <Provider>, <OrgID>, "mls-light",
              <TeamID>, <ChannelID>)
  Track:     annot_v1_<HDevID>_<leaf_index>
~~~

where `HDevID` is the committer's and `leaf_index` is the
receiving light client's leaf index. Each object has Type
0x05 ({{mls-naming}}), is published in the MOQT group of the
epoch of the corresponding Commit, and contains the
annotation for that recipient as defined in {{MLS-PARTIAL}}.
The Commit itself is published on the committer's
`mls_v1_` track as usual.

A light client subscribes to the MLS namespace as other
members do, and with SUBSCRIBE to its own annotation track for
the current committer. It processes each Commit using the
corresponding annotation. After committer transfer or
failover, light clients subscribe to the new committer's
annotation track.

## Joining as a Light Client

A device joining as a light client is added in the same way
as a full client ({{lifecycle}}). The committer publishes,
alongside the Welcome, the additional information that
{{MLS-PARTIAL}} requires for a light client to initialize its
state without the full ratchet tree, on the new member's
annotation track.

## Credential Validation by Light Clients

A light client does not hold the leaves, and therefore the
credentials, of most members. It MUST validate the credential
of each member whose messages it processes, using the sender's
leaf and membership proof as provided under {{MLS-PARTIAL}},
before presenting that member as verified ({{credentials}}).
A light client cannot by itself confirm the full membership of
the group; clients that need to display the complete member
list SHOULD obtain the full tree.

## Upgrading to a Full Client {#upgrade}

A light client becomes a full client by obtaining the ratchet
tree for the current epoch and verifying it against the tree
hash in the GroupContext. The committer publishes the current
ratchet tree on the track `tree_v1_<HDevID>` in the
"mls-light" namespace ({{partial-mls}}), one object per epoch, so that light clients can
upgrade without contacting another member. The object is
encrypted with the AEAD of the group's ciphersuite, using as
key and nonce the first AEAD.Nk and next AEAD.Nn bytes of
`MLS-Exporter("MOCHA 1.0 tree", "", AEAD.Nk + AEAD.Nn)`, so
that relays do not learn the group's membership. Since one
object is encrypted per epoch, the nonce is not reused. Only
the latest object is current.

## Status

{{MLS-PARTIAL}} is an early working group draft, and its
formats and processing rules may change. This section will be
updated to track it.


# Virtual Clients for Multi-Device Users {#virtual-clients}

By default, each device of a user is a separate leaf in every
channel's MLS group. A user with several devices therefore
occupies several leaves, and adding or removing a device
requires an Add or Remove Commit in every channel the user
belongs to.

MLS Virtual Clients {{MLS-VIRTUAL-CLIENTS}} allows a set of
devices to jointly operate a single MLS leaf. This section
defines an OPTIONAL profile in which each MOCHA user is
represented in channel groups by one virtual client, emulated
by that user's devices. Deployments that do not implement this
profile continue to use one leaf per device.

## Terminology Mapping

| Virtual Clients term         | MOCHA equivalent                  |
|------------------------------|-----------------------------------|
| Virtual client               | A user, as a single leaf per channel |
| Emulator client              | One of the user's devices         |
| Emulation group              | A per-user MLS group of the user's devices |
| Higher-level group           | A channel MLS group ({{lifecycle}}) |

## Goals

This profile primarily targets:

- **Smaller channel trees:** the number of leaves in a channel
  group equals the number of users rather than the number of
  devices, reducing Commit size and group state download cost
  and deferring the point at which Partial MLS
  ({{partial-mls}}) is needed.

- **Cheap device onboarding:** a new device joins its user's
  emulation group once and obtains the state of every channel
  the user belongs to, without an Add Commit in each channel.

- **Fewer blanks:** removing one device does not blank the
  user's leaf or direct path in channel groups.

Hiding the number of a user's devices from other channel
members is not a goal of this profile as specified; see
{{vc-challenges}}.

## Emulation Group

Each user participating in this profile maintains one
emulation group whose members are the user's devices. Only
devices holding a valid, unrevoked device binding
({{MOCHA-KEY-BINDING}}) for the user's user key are admitted.

The emulation group uses the same track structure and
Delivery Service model as a channel group ({{mls-naming}},
{{delivery-service}}), under a per-user namespace:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "mls-emu", <HClientID>)
~~~

Each device publishes emulation group messages on its own
track `mls_v1_<HDevID>` within this namespace. The
committer, failover, and keep-alive procedures of
{{delivery-service}} apply, except that the emulation group
has no channel roster, so every device holding a leaf with the
mocha_full_client extension is eligible in failover. The ordering provided by
the emulation group committer is the ordering that
{{MLS-VIRTUAL-CLIENTS}} requires for virtual client actions
such as KeyPackage uploads.

Emulation group messages MUST NOT be published under any
channel namespace. The GroupInfo carrying the
NewEmulatorClientState MUST only be delivered inside a Welcome
and MUST NOT be published on any track, since it contains the
secret state of every channel the user belongs to.

## Virtual Client Credential {#vc-credential}

The mocha_binding_chain credential requires the LeafNode
signature key to equal a device key, which does not hold for a
virtual client. This profile defines an additional credential
type:

~~~
  credential_type = mocha_virtual_client (TBD2)

  struct {
    IdentityBinding identity_binding;
    VirtualClientBinding virtual_client_binding;
  } MochaVirtualClientCredential;

  struct {
    uint16 sig_alg;
    opaque user_key<V>;
    uint16 client_sig_alg;
    opaque client_key<V>;
    uint64 issued_at;
    opaque signature<V>;
  } VirtualClientBinding;
~~~

The IdentityBinding is as defined in {{MOCHA-KEY-BINDING}}.
The `client_key` is a signature key whose private half is
shared among the user's devices through the emulation group.
The `client_key` MUST equal the `signature_key` in the
LeafNode, and `client_sig_alg` MUST be the signature scheme of
the group's ciphersuite.

The `signature` is computed by the user key using the
MochaLabeledContent construction of {{MOCHA-KEY-BINDING}},
with label "VirtualClientBinding" and content:

~~~
  struct {
    uint16 sig_alg;
    opaque user_key<V>;
    uint16 client_sig_alg;
    opaque client_key<V>;
    uint64 issued_at;
  } VirtualClientBindingTBS;
~~~

Validation follows {{credentials}}, with the
VirtualClientBinding taking the place of the DeviceBinding.
Trust remains anchored in the user key held in the PAB, so
offline validation is unaffected.

A new `client_key` and VirtualClientBinding MUST be issued
whenever a device is removed from the emulation group
({{vc-remove-device}}). Because the removed device still holds
the previous `client_key`, only the VirtualClientBinding with
the highest `issued_at` for a given user key is valid. A member
that has seen a VirtualClientBinding for a user key MUST reject
any credential carrying a VirtualClientBinding for the same
user key with a lower `issued_at`, including in KeyPackages.

## Participation in Channel Groups

The virtual client is the member of each channel group. Any of
the user's devices MAY act for it. The following rules apply:

1. A device publishes Proposals, Commits, and KeyPackages for
   the virtual client on its own per-device tracks, as
   defined in {{mls-naming}}. Two devices MUST NOT publish on
   the same track.

2. A device MUST deliver every message it sends for the
   virtual client to the other devices in the emulation group,
   as required by {{MLS-VIRTUAL-CLIENTS}}. Because the other
   devices subscribe to the channel's MLS namespace, publishing
   on the channel track satisfies this requirement.

3. Track encryption keys ({{key-derivation}}) include the
   track name, and tracks remain per device, so two devices
   acting for the same leaf never derive the same key and
   nonce. The KID leaf_index identifies the virtual client's
   leaf ({{key-derivation}}).

4. If a virtual client holds the committer role, the
   emulation group determines which device acts as committer.
   The mocha_committer extension ({{committer-ext}}) and the
   failover rule in {{committer-failover}} identify the virtual
   client by its MLSParticipantID rather than by HDevID.

## Adding a Device

A new device is added to the emulation group by one of the
user's existing devices with an MLS Add and Welcome. For each
channel group, the provisioning device SHOULD use state
transfer (Variant A of {{MLS-VIRTUAL-CLIENTS}}). It is
invisible to channel members, preserves the epoch chain, and
can complete directly between the user's own devices without
the Provider being reachable.

External commit (Variant B) MAY be used when no existing
device is online, at the cost of a visible leave-and-rejoin in
each affected channel.

Before the new device sends messages in a channel group, the
virtual client's leaf in that group MUST be rotated to an
emulation group epoch that includes the new device.

## Removing a Device {#vc-remove-device}

To remove a device (for example, when it is lost or
compromised), a remaining key-holding device of the user
({{MOCHA-KEY-BINDING}}) performs, in order:

1. Publishes a new binding chain that removes the device's
   binding and includes a device revocation for it
   ({{MOCHA-KEY-BINDING}}).
2. Replaces all outstanding virtual client KeyPackages by
   publishing new objects on its keypackage tracks.
3. Commits the Remove in the emulation group.
4. Issues a new `client_key` and VirtualClientBinding
   ({{vc-credential}}).
5. Updates the virtual client's leaf, with the new
   credential, in every channel group the user belongs to.

## Challenges {#vc-challenges}

This profile introduces trade-offs that deployments need to
weigh. The following list describes each, what
{{MLS-VIRTUAL-CLIENTS}} says about it, and what remains open
for MOCHA.

Compromise scope:
: All devices hold the full secret state of the virtual
  client. Compromising one device compromises the user in
  every channel, whereas with per-device leaves only that
  device's leaf is affected. Post-compromise security is
  regained only when the emulation group itself is updated.
  {{MLS-VIRTUAL-CLIENTS}} acknowledges both and requires
  devices to trust each other fully; it offers no
  mitigation.

Forward secrecy:
: Forward secrecy in each channel is bounded by the user's
  slowest device to delete consumed keys. {{MLS-VIRTUAL-CLIENTS}}
  recommends deleting retained state as soon as possible,
  notes that retained operation secrets are not needed
  unless state transfer is used, and recommends updating the
  leaf soon after an external commit or group creation.

Non-atomic removal:
: A removed device keeps valid keys for each channel until
  that channel's leaf has been updated, and a KeyPackage that
  escapes replacement lets it process a Welcome. The
  procedure needs another of the user's devices to be online.
  {{MLS-VIRTUAL-CLIENTS}} asks that this window be minimized
  and that long-lived credentials be revoked. For MOCHA, the
  device revocation and the new VirtualClientBinding make
  the removed device's credential invalid. However, MOQT relays
  may still serve cached KeyPackage objects in response to
  FETCH, so committers MUST only use the latest KeyPackage
  object on a track and MUST reject KeyPackages whose
  credential carries a superseded VirtualClientBinding.

Per-device control:
: Channel members and administrators see only the user, so
  they cannot remove a single stale or stolen device and
  cannot attribute a message to a device. Only the user's own
  devices can remove a device. {{MLS-VIRTUAL-CLIENTS}}
  describes this as inherent to the design. Deployments
  where administrators must be able to evict individual
  devices SHOULD NOT use this profile.

Device count visibility:
: Because each device publishes on its own HDevID-named
  track, relays and channel members subscribed to the MLS
  namespace can still count a user's devices and see which
  device is active. Hiding this would require the devices of
  a user to share a publisher identity for a single track,
  which MOQT does not currently support. Separately,
  {{MLS-VIRTUAL-CLIENTS}} notes that the DerivationInfo
  component marks a leaf as virtual and that a shared
  epoch_id links a user's leaves across groups. In MOCHA,
  the credential already identifies the user, so this link
  reveals little that is not already visible.

Dependencies:
: {{MLS-VIRTUAL-CLIENTS}} is an early, Informational draft and
  depends on the Safe Exporter of the MLS extensions framework.
  Its secret derivation, reuse guard, and state transfer
  formats may change.


# Key Derivation for Message Encryption {#key-derivation}

MLS-derived keys are used with MoQ Secure Objects
{{SecureObjects}} to encrypt MOCHA message payloads.

## Exporter Secret to Track Key

Per-track encryption keys are derived from the MLS exporter
({{MLS}}, Section 8.5) of each epoch, as recommended for
application use of MLS secrets:

~~~
  struct {
    opaque namespace_elements<V>;  /* each element as <V> */
    opaque track_name<V>;
  } FullTrackName;

  track_secret = MLS-Exporter("MOCHA 1.0 track",
                              FullTrackName, KDF.Nh)

  track_key   = ExpandWithLabel(track_secret, "key", "",
                                AEAD.Nk)
  track_nonce = ExpandWithLabel(track_secret, "nonce", "",
                                AEAD.Nn)
~~~

FullTrackName encodes the complete MOQT namespace tuple and
track name, each element as a length-prefixed vector, so that
distinct tracks never derive the same key. ExpandWithLabel,
KDF, and AEAD are as defined in {{MLS}} for the group's
ciphersuite. The track_key and track_nonce are the key and
base nonce used with {{SecureObjects}}.

Every member of the epoch can derive the key of every track.
Track encryption therefore provides confidentiality against
non-members but does not authenticate which member sent an
object; see {{security}}.

## Key ID (KID) Encoding

The KID carried in MoQ Secure Objects encodes the MLS epoch
and sender leaf index, allowing receivers to select the
correct decryption key:

~~~
  KID = (epoch << 24) | leaf_index
~~~

The KID is encoded as a MoQ varint (64 bits). This yields
40 bits for the epoch (practically unlimited) and 24 bits
for the leaf_index, supporting up to 16,777,215 members in
a single MLS group.

Receivers use the epoch to identify which exporter secret to
derive keys from. The leaf_index identifies the sender's
position in the tree, which receivers use to look up the
sender's credential. Since track keys do not depend on the
leaf index, the KID's leaf_index is not authenticated, and
receivers MUST NOT rely on it to authenticate the sender.


# Credentials {#credentials}

MLS requires each member to present a Credential in their
LeafNode that binds their identity to their MLS signature key.
MOCHA uses the mocha_binding_chain credential type as the
primary credential mechanism, and the mocha_virtual_client
type ({{vc-credential}}) for virtual clients.

The structure and validation of mocha_binding_chain, including
the PAB lookup and the handling of unverified members, are
specified normatively in {{MOCHA-KEY-BINDING}} and are not
repeated here. Members MUST validate every LeafNode they
receive (via a Commit, Welcome, or KeyPackage) as specified
there.

## Offline Validation

Because the credential is self-contained in the LeafNode and
the PAB lookup is local, existing group members who already
have each other's user keys in their PABs verify credentials
without network access. This is critical for MOCHA's offline
operation goal: the home server need not be reachable for
existing group members to verify each other's MLS messages.

## Device Revocation

When a member learns from a verified binding chain that a
device key used as a LeafNode `signature_key` in a group has
been revoked or is no longer listed ({{MOCHA-KEY-BINDING}}),
it SHOULD publish a Remove Proposal for that leaf as described
in {{lifecycle}}. The committer MUST NOT commit Proposals
from a leaf whose device key it knows to be revoked.

## KeyPackage Credentials

KeyPackages published on the keypackage track ({{mls-naming}})
MUST include a mocha_binding_chain credential, or a
mocha_virtual_client credential when the virtual client
profile is used ({{virtual-clients}}). This allows the
committer and other members to verify the identity of a new
joiner before the Add Commit is issued.

## Credential Rotation {#credential-rotation}

When a user rotates their user key ({{MOCHA-KEY-BINDING}}),
each of their devices MUST publish new KeyPackages with an
updated credential containing the new identity_binding and
device_binding. In each group, each device also publishes an
Update Proposal carrying a LeafNode with the new credential,
which the committer commits as described in {{lifecycle}}.

{{MLS}} requires the application to confirm that a changed
credential in an Update represents the same client. Members
MUST accept the new credential only if they have verified a
key rotation statement from the old user key to the new one,
and the new binding chain, as specified in
{{MOCHA-KEY-BINDING}}. Peers update their PAB entry by the
same mechanism.


# Security Considerations {#security}

## Relays

Relays cannot read MLS secrets or application data, and
cannot forge MLS messages or keep-alives. They can, however,
delay, drop, reorder, or withhold objects, and serve stale
cached objects. Consequences include:

- Withholding a Commit from some members leaves them in an
  older epoch, unable to decrypt newer messages. Members detect
  this when they receive objects encrypted for a later epoch.
- Withholding keep-alives causes members to start failover.
  Failover is deterministic ({{committer-failover}}) and forks
  are resolved ({{forks}}), so this degrades availability but
  does not split the group permanently.
- Serving old KeyPackages could cause a consumed or superseded
  KeyPackage to be used; committers use only the latest object
  on a KeyPackage track.

Relays observe metadata: which devices publish in a channel
(through HDevID-named tracks), message timing and sizes, and
the epochs in which they occur. Sending handshake messages as
PrivateMessage ({{mls-naming}}) hides their content but not
their existence.

## Committer

The committer orders all changes to the group. A misbehaving
committer can delay or refuse to commit Proposals, for example
to keep a removed member in the group, but cannot forge
Proposals from other members, and any change it commits is
visible to all members. Followers check each Commit against
{{authorization}}. A committer that is live but refuses to
commit is displaced when an authorized member proposes its
removal and the Proposal is not committed in time
({{committer-failover}}); deployments SHOULD allow channel
administrators to make such Proposals.

## Compromised Members

Any member of an epoch can derive every track key for that
epoch ({{key-derivation}}). A compromised or malicious member
can therefore decrypt all traffic and produce objects that
other members accept as coming from any track. Applications
that require sender authentication within the group MUST
authenticate content separately, for example with a signature
by the sender's device key ({{MOCHA-KEY-BINDING}}). MLS
provides post-compromise security for future epochs once the
compromised member is removed or updates its leaf.

## Group State Consistency

During a partition, members may advance on different branches
of group state. {{forks}} ensures that members converge, at
the cost of messages sent on the losing branch. Bootstrapping
with a fresh `group_id` ({{rebootstrap}}) ensures that state
from different bootstraps is never mixed.

## Light Client Trust {#light-trust}

Light clients rely on annotations from the committer and
verify them against the group's tree hash ({{MLS-PARTIAL}}).
They do not validate the credentials of all members and so
cannot confirm the full membership of the group; see
{{partial-mls}}. They also cannot check that a failover
Commit came from the member selected by the failover rule, so
a malicious full client could claim the committer role toward
light clients. Full clients detect such a Commit, and light
clients follow the fork resolution of {{forks}} when it
conflicts with the branch the full clients follow.

## Virtual Client Profile

The virtual client profile shares a user's leaf secrets among
all of the user's devices. Its security trade-offs are
discussed in {{vc-challenges}}.


# IANA Considerations

## MLS Credential Type

IANA is requested to add the following entry to the "MLS
Credential Types" registry established by {{MLS}}:

| Value | Name | Recommended | Reference |
|:------|:-----|:------------|:----------|
| TBD2 | mocha_virtual_client | N | This document |

Implementations prior to assignment use the private-use value
0xF002.

## MLS Extension Type

IANA is requested to add the following entry to the "MLS
Extension Types" registry established by {{MLS}}:

| Value | Name | Message(s) | Recommended | Reference |
|:------|:-----|:-----------|:------------|:----------|
| TBD3 | mocha_committer | GC | N | This document |
| TBD4 | mocha_full_client | LN | N | This document |

Implementations prior to assignment use a value from the
private-use range.

## MLS Exporter Labels

IANA is requested to add the following entries to the "MLS
Exporter Labels" registry established by {{MLS}}:

| Label | Recommended | Reference |
|:------|:------------|:----------|
| "MOCHA 1.0 track" | N | This document |
| "MOCHA 1.0 keepalive" | N | This document |
| "MOCHA 1.0 tree" | N | This document |

--- back
