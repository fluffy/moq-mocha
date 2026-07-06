# Abtract 

This specification defined the teminology and architecture used in Mocha. 

# Goals 

Form a meeting and messaging system for users across a set of federated providers. Allow users to chat with and have audio vidoe meetings using only MoQ Relays to trasfer the data. While operations creating new organzations, users, devices, teams, or channels will require or changing permissions or membership will require the Provider to be reachable, basic chat and meetings work to without access to the Provider. All content is end to end encryped. 


# Terminology 

* User: Represnets a humany or bot user. Has an identity on a single Orginatzation in Provider. Users are identified by an E.164 phone number at the Provider they use. A users phone nuber can be changed but they have a unique UserID at that provider that can not be changed. 

* Device: Device a users uses to interact with Mocha. A device has a  a single users. User can have multiple devices. 

* DevID: 64 bit identifier for device and statistially unique within a single Team. 

* Provider: A logical entity that provides identity, policy, and membership for a set of users and organizations. Providers are identified by a DNS name. 

* Origanization: A logical entity owned by a Provider containingn multiple Users and teams. 

* Team: A grouping of users with an Organzation. A team is owned by a single Organization but may have users from multiple organizations across multuple providers. 

* Channel: A named location inside a Team that has an roughly chronilogical conversation. Same ideas as a Room in other systems. 

* Message: A small chunk of content in a Room sent at a point in time. Can be any media type. 

* Translation: A conversion of a message to another media format or language. 

* Thread: An order set of Message that share a common base Message that was not part of any other Thread. Threads are not nested. 

* Reaction: A glyph sent in reaction to a Message. 

* Meeting: An audio video conference call. This happens in the context a of a single Channel but a Channel can have many meeting over time or concurently. 

* Roles: Organizations, Teams, Channel and Meetings all have Users that can take on different roles which defined their permisiions. Roles include: Owner, Moderator, Partipant, Observer

* GroupID: The 32 bit MoQ Group. The ID is the NTP time in minutes for creation time of data inside this group. 

* ObjectID: The 64 bit integer indientifing an MoQ object inside a group. 

# Protocol Layer Cake

MOCHA is designed as a layered protocol stack. Each layer depends on the layers below it:

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

All infomation between Devices for Channels and Meetings is echanged over a network of MoQ relays.  Each Device also subscribes for information about it's configureation includding from it's Provider. 

## Persistence Service

The Persistence Service provides durable storage for MOCHA data beyond the lifetime of relay caches. It stores token revocation records, user roles and membership state, chat message history, channel metadata, and meeting recordings. The Persistence Service participates in MOQT as both a subscriber (ingesting live data from channel namespaces and the Token Service revocation track) and a publisher (serving historical data to clients and relays via FETCH requests that exceed relay cache retention).

