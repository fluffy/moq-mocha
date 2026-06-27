---
title: "MOCHA Personal Address Book"
abbrev: "mocha-pab"
category: std

docname: draft-jennings-moq-mocha-pab-latest
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
 - Address Book
 - Contacts
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
  RFC7095:
  RFC8259:

informative:

--- abstract

This document defines how a set of devices owned by a common
user synchronize a Personal Address Book (PAB) over MOQT. The
address book maps identities to human-readable names and contact
information, enabling users to maintain a trusted contact list
that is consistent across all their devices.

--- middle

# Introduction

MOCHA users need to maintain a personal directory of contacts
— mapping user identities (e.g., alice@provider.example) to
display names and other metadata they trust. This address book
must synchronize across all devices belonging to the same user
without requiring a centralized contacts server.

This document specifies:

- The MOQT namespace and track structure for address book data
- The contact card format (based on jCard {{RFC7095}})
- The synchronization model across a user's devices
- Group and object organization for efficient retrieval

# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology defined in {{MOCHA-ARCH}},
including Provider, Organization, UserID, Device, and DevID.

The following additional terms are used:

Personal Address Book (PAB):
: A per-user collection of contact cards synchronized across
  all of the user's devices via MOQT.

Contact Card:
: A single entry in the address book, encoded as a jCard
  {{RFC7095}} object. Each card maps an identity to a
  human-readable name and optional metadata.

# Namespace and Track Structure {#namespace}

## Namespace {#pab-namespace}

The MOQT tracks carrying address book contact cards use the
following namespace tuple:

~~~
  ("mocha_v1", <Provider>, <OrgID>, "pab", <UserID>)
~~~

Where:

- `Provider`: The user's home provider domain
- `OrgID`: The organization the user belongs to
- `pab`: Literal string identifying this as a Personal Address
  Book namespace
- `UserID`: The identity of the user who owns this address book

## Track Name

Each device publishes its address book updates on its own track:

~~~
  Track Name: ("msg", <DevID>)
~~~

Where `DevID` identifies the publishing device. Each device
maintains its own track to avoid write conflicts between
devices.

## Example

~~~
  Namespace: ("mocha_v1", "example.com", "acme", "pab", "alice")
  Track:     ("msg", "device-a3f2")
~~~

This represents the address book updates published by Alice's
device "device-a3f2" within the "acme" organization on
provider "example.com".

# Group and Object Organization {#groups}

## Groups {#pab-groups}

There is one group per calendar month. This provides a natural
time-based partitioning that allows devices to efficiently
fetch recent changes without downloading the entire history.

Groups are numbered sequentially, with the group ID derived
from the month:

~~~
  Group ID = (year - 2024) * 12 + (month - 1)

  Example: June 2026 = (2026 - 2024) * 12 + (6 - 1) = 29
~~~

## Objects

Each object within a group represents a single contact card
operation (add, update, or delete). Objects are sequentially
numbered within each group.

~~~
  Namespace: ("mocha_v1", "example.com", "acme", "pab", "alice")
  Track:     ("msg", "device-a3f2")
  Group:     29 (June 2026)
  Object:    0, 1, 2, ... (sequential operations)
~~~

# Contact Card Format {#card-format}

## Overview

Each object payload is a JSON {{RFC8259}} document containing
a contact card operation. The card data follows the jCard
format defined in {{RFC7095}}.

## Operation Types

~~~
{
  "op": "add" | "update" | "delete",
  "id": "<contact identity>",
  "timestamp": <unix timestamp>,
  "card": <jCard object (for add/update)>
}
~~~

## Example: Adding a Contact

~~~
{
  "op": "add",
  "id": "bob@example.com",
  "timestamp": 1751000000,
  "card": ["vcard", [
    ["version", {}, "text", "4.0"],
    ["fn", {}, "text", "Bob Smith"],
    ["email", {}, "text", "bob@example.com"]
  ]]
}
~~~

## Example: Deleting a Contact

~~~
{
  "op": "delete",
  "id": "bob@example.com",
  "timestamp": 1751100000
}
~~~

# Synchronization Model {#sync}

## Device Publishing

Each device publishes contact card operations to its own track.
When a user adds, modifies, or deletes a contact on any device,
that device publishes a new object to the current month's group
on its track.

## Device Subscribing

Each device subscribes to the tracks of all other devices
belonging to the same user. This is achieved by subscribing to
the PAB namespace:

~~~
  SUBSCRIBE_NAMESPACE ("mocha_v1", <Provider>, <OrgID>,
                       "pab", <UserID>)
~~~

The relay delivers objects from all device tracks within this
namespace, allowing each device to learn of changes made on
other devices.

## Conflict Resolution

When multiple devices publish conflicting operations (e.g.,
both update the same contact), the operation with the latest
timestamp wins. If timestamps are equal, the operation from
the lexicographically greater DevID takes precedence.

## Initial Sync

A newly added device subscribes starting from the earliest
available group to reconstruct the full address book state.
The device applies all operations in group and object order,
with later operations overriding earlier ones for the same
contact identity.

# Authorization {#auth}

Access to a user's PAB namespace is restricted to that user's
own devices. The Token Service issues C4M tokens (or Privacy
Pass tokens) scoped to the user's PAB namespace:

~~~
  publish:   ["mocha_v1/<provider>/<org>/pab/<user_id>/msg/<own-device>"]
  subscribe: ["mocha_v1/<provider>/<org>/pab/<user_id>/*"]
~~~

Each device can only publish to its own track but can subscribe
to all tracks within the user's PAB namespace.

# Security Considerations

TODO

# IANA Considerations

This document makes no requests of IANA.

--- back
