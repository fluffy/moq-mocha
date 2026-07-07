---
title: "MOCHA Architecture: Terminology and System Design"
abbrev: "mocha-arch"
category: info

docname: draft-jennings-moq-mocha-arch-00
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
 - Architecture
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
  MOCHA-IDENTITY: I-D.draft-jennings-moq-mocha-identity
  MOCHA-CHAT: I-D.draft-jennings-moq-mocha-chat
  MOCHA-MEETINGS: I-D.draft-jennings-moq-mocha-meetings
  MOCHA-MLS: I-D.draft-jennings-moq-mocha-mls-keying

informative:


--- abstract

This specification defines the terminology and architecture used
in MOCHA (MoQ Open Communication & Hosting Architecture).

--- middle

# Introduction

MOCHA forms a meeting and messaging system for users across a
set of federated providers. It allows users to chat with and
have audio video meetings using only MoQ Relays to transfer the
data. While operations creating new organizations, users,
devices, teams, or channels, or changing permissions or
membership, require the Provider to be reachable, basic chat
and meetings work without access to the Provider. All content
is end to end encrypted.

# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

# Terminology

User:
: Represents a human or bot user. Has an identity on a single
  Organization in a Provider. Users are identified by an E.164
  phone number at the Provider they use. A user's phone number
  can be changed but they have a unique UserID at that provider
  that cannot be changed.

Device:
: A device a user uses to interact with MOCHA. A device belongs
  to a single user. A user can have multiple devices.

DevID:
: A 64 bit identifier for a device, statistically unique within
  a single Team.

Provider:
: A logical entity that provides identity, policy, and
  membership for a set of users and organizations. Providers
  are identified by a DNS name.

Organization:
: A logical entity owned by a Provider containing multiple
  Users and Teams.

Team:
: A grouping of users within an Organization. A team is owned
  by a single Organization but may have users from multiple
  organizations across multiple providers.

Channel:
: A named location inside a Team that has a roughly
  chronological conversation. Same concept as a Room in other
  systems.

Message:
: A small chunk of content in a Channel sent at a point in
  time. Can be any media type.

Translation:
: A conversion of a message to another media format or
  language.

Thread:
: An ordered set of Messages that share a common base Message
  that was not part of any other Thread. Threads are not
  nested.

Reaction:
: A glyph sent in reaction to a Message.

Meeting:
: An audio video conference call. This happens in the context
  of a single Channel but a Channel can have many meetings
  over time or concurrently.

Roles:
: Organizations, Teams, Channels, and Meetings all have Users
  that can take on different roles which define their
  permissions. Roles include: Owner, Moderator, Participant,
  Observer.

GroupID:
: The 32 bit MoQ Group. The ID is the NTP time in minutes for
  creation time of data inside this group.

ObjectID:
: The 64 bit integer identifying an MoQ object inside a group.

# Protocol Layer Cake

MOCHA is designed as a layered protocol stack. Each layer
depends on the layers below it:

~~~
  ┌─────────────────────────────────────────────────────────────┐
  │  Layer 5: Identity                                          │
  │  (Provider-assigned identities, E.164 phone numbers,        │
  │   UserID, DevID, MLSParticipantID)                          │
  └─────────────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Layer 4: Authorization                                     │
  │  (C4M tokens, DPoP binding, token revocation,               │
  │   roles: Owner, Moderator, Participant, Observer)           │
  └─────────────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Layer 3: Keying & Encryption                               │
  │  (MLS key management, MoQ Secure Objects,                   │
  │   per-epoch key derivation, E2EE)                           │
  └─────────────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Layer 2: Message Format                                    │
  │  (Chat messages, reactions, meetings media/catalog,         │
  │   translations, threads, edits/deletes)                     │
  └─────────────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Layer 1: MoQ Transport                                     │
  │  (MOQT over QUIC/WebTransport, pub/sub, namespaces,         │
  │   relay caching, fan-out, TLS 1.3 hop-by-hop)              │
  └─────────────────────────────────────────────────────────────┘
~~~

# Architecture

All information between Devices for Channels and Meetings is
exchanged over a network of MoQ relays. Each Device also
subscribes for information about its configuration including
from its Provider.

## Persistence Service

The Persistence Service provides durable storage for MOCHA
data beyond the lifetime of relay caches. It stores token
revocation records, user roles and membership state, chat
message history, channel metadata, and meeting recordings.
The Persistence Service participates in MOQT as both a
subscriber (ingesting live data from channel namespaces and
the Token Service revocation track) and a publisher (serving
historical data to clients and relays via FETCH requests that
exceed relay cache retention).

# Security Considerations

MOCHA relies on end-to-end encryption via MLS key management
(Layer 3) to protect message content from intermediaries
including relays. Authorization tokens (Layer 4) control
access to channels and meetings. The relay network provides
hop-by-hop TLS 1.3 encryption for transport protection.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
