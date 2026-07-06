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
  MOCHA-CHAT: I-D.draft-jennings-moq-mocha-chat
  MLS: RFC9420
  SecureObjects: I-D.draft-jennings-moq-secure-objects
  MLS-PARTIAL: I-D.draft-ietf-mls-partial

informative:

--- abstract

This document specifies MLS key management for MOCHA (MoQ Open
Communication & Hosting Architecture). It defines how MLS groups
are created, how members are added and removed, how key material
is distributed over MOQT tracks, and how the MLS Delivery
Service is realized in a fully distributed manner using MOQT
publish/subscribe. For large channels, Partial MLS is used to
avoid requiring all members to process every membership change.

--- middle

# Introduction

Keying for end-to-end encryption in MOCHA is provided by the
Message Layer Security (MLS) protocol {{MLS}}. MLS requires a
Delivery Service to distribute protocol messages (KeyPackages,
Commits, Welcome messages) between group members. Traditional
MLS deployments use a centralized server for this role.

This document specifies how MOCHA realizes the MLS Delivery
Service in a fully distributed manner using MOQT's
publish/subscribe primitives. Each member publishes MLS
protocol messages on its own MOQT track, and other members
subscribe to receive them. This eliminates the single point
of failure and trust concentration inherent in a centralized
Delivery Service.

For large channels where full MLS group membership becomes
impractical, this document specifies the use of Partial MLS
{{MLS-PARTIAL}}, which allows subgroups of members to
participate without processing every commit in the full group.

Credentials used within MLS (signing keys, identity
assertions) are obtained via the mechanisms defined in
{{MOCHA-IDENTITY}}.


# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

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
  carry Add, Remove, or Update proposals.

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

The MLSParticipantID is derived from the participant's MLS
credential and uniquely identifies a participant within the
channel's MLS group.

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

## Object Semantics

Each MLS protocol message (Proposal, Commit, or Welcome)
is published as a single MOQT object. Objects within a group
are sequentially numbered. The MOQT Group ID corresponds to
the MLS epoch number, providing natural alignment between
MLS state transitions and MOQT group boundaries.

## Object Format

Each MOQT object on an MLS track carries a 1-byte type
prefix followed by the raw MLS message bytes:

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

This prefix allows receivers to quickly dispatch messages
without parsing the MLS wire format. The MLS Message field
contains the unmodified MLSMessage struct as defined in
{{MLS}}, Section 6.


# Distributed Delivery Service {#delivery-service}

## Design

Traditional MLS requires a Delivery Service that orders
Commits and ensures all members see the same sequence of
group operations. In MOCHA, this role is distributed across
the MOQT relay network.

The key insight is that MOQT relays already provide:

- Reliable delivery of published objects to all subscribers
- Caching for late-joining members
- Fan-out to all subscribers of a track

By assigning each member its own MLS track within the channel
namespace, the relay network collectively acts as the Delivery
Service without any single entity controlling message ordering
or having access to plaintext key material.

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

This separation ensures exactly one Commit per epoch
transition with no conflicts or rollbacks.

The committer MAY batch multiple pending Proposals into a
single Commit to minimize epoch churn. For example, if
several Add or Remove Proposals arrive in quick succession,
the committer can coalesce them into one Commit that
advances the epoch once rather than producing a separate
epoch transition for each Proposal.

## Committer Selection {#committer-selection}

### Initial Committer

The Provider designates the initial committer when the
channel is created. The channel metadata returned by the
Provider includes the creator's HDevID as the initial
committer. Since channel creation requires the Provider to
be reachable, the Provider serializes this assignment —
no race condition is possible.

### Group Re-Bootstrap

If the MLS group requires re-creation (all members have
left, or the group state is explicitly reset), the first
authorized member to join and observe no active keep-alive
track assumes the committer role and creates a new epoch 0.
If two members join simultaneously and both attempt epoch 0,
relays deliver both but members MUST accept only the first
valid Welcome they process and ignore subsequent attempts.
The losing bootstrapper observes the established group and
falls back to follower behavior.

Because this fallback uses Provider-authorized channel
membership rather than always electing the lowest HDevID,
the attack surface is reduced — only devices the Provider
has authorized for the channel can assume the committer
role.

### Committer Transfer

The current committer MAY designate a successor by including
a committer_transfer extension in a Commit. The successor's
HDevID is recorded in the group state.

## Committer Failover {#committer-failover}

If the committer becomes unavailable (temporary disconnection
or permanent departure), the group must elect a new committer
to continue processing Proposals.

Unavailability is detected when:

- No valid keep-alive ({{keep-alive}}) is received within
  the deployment-configured timeout (e.g., 30 seconds).
- The committer is explicitly removed from the group.

When unavailability is detected, the remaining members
elect a new committer using the following deterministic rule:

1. All active members (those with a currently published
   roster entry with status "active") are eligible.
2. If the departing committer's last Commit contained a
   committer_transfer extension, the designated HDevID
   becomes the new committer. Otherwise, the eligible
   member with the lowest HDevID (lexicographic ordering)
   becomes the new committer.
3. The new committer publishes a Commit containing all
   pending Proposals plus its own Update proposal to
   signal committer succession.

This ensures all members independently arrive at the same
new committer without explicit coordination. The timeout
SHOULD be long enough to tolerate transient network
disruptions but short enough to avoid blocking group
operations.

## Follower Behavior

Followers subscribe to all MLS tracks in the channel
namespace via SUBSCRIBE_NAMESPACE. They:

1. Publish Proposals on their own MLS tracks when they
   wish to add, remove, or update members.
2. Apply Commits only from the current committer's track.
3. Ignore Commits published by non-committer members
   (except during failover succession).

## Committer Keep-Alive {#keep-alive}

The committer periodically publishes keep-alive objects to
prove liveness and current epoch membership. All members
MUST subscribe to the committer's keep-alive track:

~~~
  keepalive_v1_<HDevID>
~~~

### Track Properties

The committer sets the following Track Property when
publishing to the keep-alive track:

- **keepalive_interval**: The expected interval in seconds
  between keep-alive objects. Members use this value as the
  basis for failover timeout detection.

### Keep-Alive Object Format

~~~json
{
  "epoch": 5,
  "ts": 1715788800,
  "interval": 10,
  "nonce": "<16 bytes, hex-encoded>",
  "mac": "<HMAC output, hex-encoded>"
}
~~~

The `interval` field is the keep-alive interval in seconds,
matching the Track Property value. This allows new joiners
who FETCH the latest object to learn the timeout without
having received the original SUBSCRIBE Track Properties.

The MAC is computed as:

~~~
  mac = HMAC(confirmation_key,
             big_endian_uint64(epoch) ||
             big_endian_uint64(ts) ||
             nonce_bytes)
~~~

where confirmation_key is the MLS confirmation key for the
current epoch ({{MLS}}, Section 8). All group members hold
this key as part of the MLS key schedule.

### Verification

Upon receiving a keep-alive object, each member:

1. Reads the epoch, ts, and nonce fields from the object.
2. Computes the HMAC using its own confirmation_key for the
   indicated epoch.
3. Compares the computed MAC against the received mac field.
4. If verification succeeds, the committer is confirmed as
   live and in possession of current epoch secrets.
5. If verification fails, the member SHOULD treat the
   committer as potentially compromised or out of sync and
   initiate failover ({{committer-failover}}).

### Timing

The committer MUST publish a keep-alive at the interval
advertised in the keepalive_interval Track Property and
the object's interval field. Members trigger failover if
no valid keep-alive is received within twice the advertised
interval ({{committer-failover}}).

The nonce field (random bytes generated per keep-alive)
ensures each object produces a unique MAC, preventing relays
from replaying a previously observed keep-alive within the
same epoch.

## KeyPackage Distribution

Each device publishes its current KeyPackage on its
keypackage track within the channel's keypackage namespace:

~~~
  Namespace: ("mocha_v1", <Provider>, <OrgID>, "keypackage",
              <TeamID>, <ChannelID>, <MLSParticipantID>)
  Track: keypackage_v1_<HDevID>
~~~

The committer MUST subscribe to the KeyPackage namespace
(SUBSCRIBE_NAMESPACE) to be notified when new devices
publish KeyPackages. Upon receiving a new KeyPackage, the
committer creates an Add Proposal and includes it in its
next Commit.

Other members MAY also subscribe to the KeyPackage namespace
and publish Add Proposals on their own MLS tracks, but
only the committer produces the Commit that incorporates
the Proposal.

KeyPackages are rotated by publishing a new object on the
track. Each KeyPackage is single-use; after being consumed
in an Add Proposal, the device MUST publish a fresh
KeyPackage.

## Welcome Message Delivery

When a member issues a Commit that adds a new participant,
it publishes the Welcome message as an object on its own
MLS track. The Welcome is addressed to the new member by
including the target HDevID in an object property, allowing
the new member to identify which object contains its Welcome.


# Group Lifecycle {#lifecycle}

## Group Creation

When a channel is created, the creator (designated as the
initial committer by the Provider) initializes a new MLS
group:

1. Creator generates an MLS group with a fresh group ID
   derived from the channel namespace.
2. Creator publishes (PUBLISH) its KeyPackage on track
   `keypackage_v1_<HDevID>` within the keypackage namespace:
   ("mocha_v1", \<Provider\>, \<OrgID\>, "keypackage",
   \<TeamID\>, \<ChannelID\>, \<MLSParticipantID\>).
3. Creator publishes the initial Commit (epoch 0 to epoch 1)
   on track `mls_v1_<HDevID>` within the MLS namespace:
   ("mocha_v1", \<Provider\>, \<OrgID\>, "mls",
   \<TeamID\>, \<ChannelID\>, \<MLSParticipantID\>).
4. Creator publishes its initial keep-alive on track
   `keepalive_v1_<HDevID>` within the MLS namespace.
5. Creator subscribes (SUBSCRIBE_NAMESPACE) to the keypackage
   namespace to receive KeyPackages from new joiners.
6. Creator subscribes (SUBSCRIBE_NAMESPACE) to the MLS
   namespace to receive Proposals from other members.

## Adding Members

To add a member to the channel:

1. The new device publishes (PUBLISH) its KeyPackage on track
   `keypackage_v1_<HDevID>` within the keypackage namespace.
2. The committer, subscribed to the keypackage namespace,
   receives the new KeyPackage via the relay.
3. The committer creates an MLS Commit containing an Add
   Proposal referencing the received KeyPackage. Only the
   current committer MUST produce Commits.
4. The committer publishes the Commit and the Welcome message
   on track `mls_v1_<HDevID>` within the MLS namespace.
5. All existing members, subscribed to the MLS namespace,
   receive the Commit and advance to the new epoch. Members
   MUST cache received Proposals locally until the
   corresponding Commit from the committer is received.
6. The new member subscribes (SUBSCRIBE) to the committer's
   track `mls_v1_<committer-HDevID>`, receives the Welcome,
   and initializes its group state at the new epoch.
7. The new member subscribes (SUBSCRIBE_NAMESPACE) to the
   MLS namespace to receive future Proposals and Commits.
8. The new member subscribes (SUBSCRIBE) to the committer's
   keep-alive track `keepalive_v1_<committer-HDevID>`.

Alternatively, any member subscribed to the keypackage
namespace MAY create an Add Proposal and publish it on its
own track `mls_v1_<HDevID>`. The committer incorporates such
Proposals into its next Commit.

## Removing Members

To remove a member:

1. The remover creates an MLS Remove Proposal for the target
   member's leaf index and publishes (PUBLISH) it on track
   `mls_v1_<HDevID>` within the MLS namespace.
2. The committer, subscribed to the MLS namespace, receives
   the Remove Proposal and creates an MLS Commit
   incorporating it. Only the current committer MUST produce
   Commits.
3. The committer publishes the Commit on its track
   `mls_v1_<HDevID>` within the MLS namespace.
4. All remaining members, subscribed to the MLS namespace,
   receive the Commit and advance to the new epoch. Members
   MUST cache received Proposals locally until the
   corresponding Commit from the committer is received.
5. The removed member can no longer decrypt messages sent
   in subsequent epochs.

## Key Update

Any member can rotate its own key material:

1. The member creates an MLS Update Proposal with fresh key
   material and publishes (PUBLISH) it on track
   `mls_v1_<HDevID>` within the MLS namespace.
2. The committer, subscribed to the MLS namespace, receives
   the Update Proposal and creates an MLS Commit
   incorporating it. Only the current committer MUST produce
   Commits.
3. The committer publishes the Commit on its track
   `mls_v1_<HDevID>` within the MLS namespace.
4. All members, subscribed to the MLS namespace, receive the
   Commit and advance to the new epoch with updated key
   material. Members MUST cache received Proposals locally
   until the corresponding Commit from the committer is
   received.


# Partial MLS for Large Channels {#partial-mls}

For channels with many members (hundreds or thousands), full
MLS becomes impractical because every member must download
and process every Commit. Partial MLS {{MLS-PARTIAL}} allows
members to participate in the group without tracking the
full ratchet tree.

## When to Use Partial MLS

Channels SHOULD use Partial MLS when the member count exceeds
a deployment-configured threshold (e.g., 100 members).
Channels MAY start with full MLS and transition to Partial
MLS as membership grows.

## Partial MLS Operation

In Partial MLS mode:

1. A subset of members (the "core group") maintains the full
   MLS tree state and processes all Commits.

2. Other members ("partial members") hold only their own
   leaf state and the path to the root. They can encrypt
   and decrypt application messages using the group's
   application secret, but do not process every Commit.

3. When a partial member needs to send a Commit (e.g., to
   update its own keys), it publishes a partial Commit that
   the core group integrates into the full tree.

4. The core group periodically publishes "sync" objects that
   provide partial members with updated path secrets, allowing
   them to remain in sync without processing the full Commit
   history.

## Sync Track

A dedicated sync track carries periodic state updates for
partial members:

~~~
  sync_v1_<HDevID>
~~~

Core group members publish sync objects containing the path
secrets needed by partial members to derive current
application keys.


# Key Derivation for Message Encryption {#key-derivation}

MLS-derived keys are used with MoQ Secure Objects
{{SecureObjects}} to encrypt MOCHA message payloads.

## Application Secret to Track Key

The MLS application secret for each epoch is used to derive
per-track encryption keys:

~~~
  track_key = HKDF-Expand(application_secret,
                          "mocha_track" || track_name,
                          key_length)

  track_nonce = HKDF-Expand(application_secret,
                            "mocha_nonce" || track_name,
                            nonce_length)
~~~

## Key ID (KID) Encoding

The KID carried in MoQ Secure Objects encodes the MLS epoch
and sender leaf index, allowing receivers to select the
correct decryption key:

~~~
  KID = (epoch << 24) | leaf_index
~~~

The KID is encoded as a MoQ Varint (64 bits). This yields
40 bits for the epoch (practically unlimited) and 24 bits
for the leaf_index, supporting up to 16,777,215 members in
a single MLS group. Channels exceeding this limit use
Partial MLS ({{partial-mls}}).

Receivers use the epoch to identify which application secret
to derive keys from, and the leaf index to identify the
sender's position in the tree.


# Credentials {#credentials}

MLS requires each member to present a credential binding
their identity to their signing key. In MOCHA, credentials
are obtained through the identity system defined in
{{MOCHA-IDENTITY}}.

The MLS credential contains:

- The member's userId (as used in MOCHA message sender fields)
- The member's signing public key
- A signature from the Provider's Token Service binding the
  userId to the public key

Credential validation follows the trust model established
by the Provider's identity infrastructure. Members MUST
verify credentials against the Provider's public key before
accepting any MLS protocol message.


# Security Considerations

TODO


# IANA Considerations

This document makes no requests of IANA.

--- back
