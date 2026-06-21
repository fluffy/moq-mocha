---
title: "MOCHA Chat: Text Messaging over MoQ Transport"
abbrev: "mocha-chat"
category: std

docname: draft-jennings-moq-mocha-chat-latest
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
 - Chat
 - Messaging
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
  JSON: RFC8259

informative:
  MIMI-CONTENT:
    title: "More Instant Messaging Interoperability (MIMI) Content Format"
    date: 2024
    author:
    - fullname: R. Mahy
    target: https://datatracker.ietf.org/doc/draft-ietf-mimi-content/

--- abstract

This document specifies the text messaging functionality for MOCHA
(MoQ Open Communication & Hosting Architecture). It defines how
participants send and receive text messages in channels using
MoQ Transport (MOQT) publish/subscribe primitives. Each device
publishes messages on its own track within a channel namespace,
enabling decentralized message production with relay-based
fan-out. This specification covers message naming, format,
causal ordering, delivery, roster management, and channel
discovery for text-based chat.

--- middle

# Introduction

Text messaging is the most fundamental operation in any
communication system. This document specifies how MOCHA enables
real-time text exchange between participants in channels using
MOQT's publish/subscribe model.

MOCHA Chat assigns each publishing device its own MOQT track
within a channel namespace. Messages are grouped by NTP time
in minutes, enabling straightforward time-based history
retrieval and relay caching. Causal ordering is maintained
through a previous-message linkage that forms a DAG across
the per-device tracks in a channel.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "SHOULD", and "MAY" in this
document are to be interpreted as described in BCP 14
{{!RFC2119}} {{!RFC8174}}.

This document uses terminology and architecture defined in
{{MOCHA-ARCH}}, including Provider, Organization, Team, Channel,
Device, DevID, GroupID, and ObjectID. The MOQT transport
primitives (Track, Group, Object, Namespace, Relay) are defined
in {{MoQTransport}}.

The following additional terms are used:

Provider:
: The service provider operating the MOCHA deployment. Identified
  by a domain name or similar unique string.

OrgID:
: The organization identifier within a provider's deployment.

TeamID:
: An identifier for a team or workspace within an organization.

ChannelID:
: An identifier for a specific channel within a team.

DevID:
: A unique identifier for a publishing device. Each device that
  can send messages has its own DevID and publishes on its own
  track.

MsgID:
: A globally unique identifier for a message, formed by hashing
  the full track name concatenated with the Group ID and
  Object ID.

# Security Considerations

TODO

# IANA Considerations

TODO

--- back
