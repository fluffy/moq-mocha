---
title: "MOCHA Identity: Authentication, Authorization, and Federation"
abbrev: "mocha-identity"
category: std

docname: draft-jennings-moq-mocha-identity-latest
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
 - Identity
 - Authorization
 - Federation
 - Privacy Pass
 - C4M
 - DPoP
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
  C4M: I-D.draft-ietf-moq-c4m
  PrivacyPassAuth: I-D.draft-ietf-moq-privacy-pass-auth
  PP-REVERSE: I-D.draft-meunier-privacypass-reverse-flow
  PP-BATCH: I-D.draft-ietf-privacypass-batched-tokens
  RFC8392:
  RFC9421:
  OIDC:
    title: "OpenID Connect Core 1.0"
    date: 2014
    target: https://openid.net/specs/openid-connect-core-1_0.html

informative:

--- abstract

This document specifies identity, authentication, authorization,
and federation for MOCHA (MoQ Open Communication & Hosting
Architecture). It defines how users authenticate with Identity
Providers (IdPs), how acess tokens
are issued, how permissions map to MOQT
namespaces, and how users from one Provider obtain access to
resources on a federated Provider.

--- middle

# Introduction

MOCHA requires participants to prove their identity and obtain
authorization before publishing or subscribing to tracks on MOQT
relays. This document specifies the identity and authorization
architecture, covering:

- Identity Provider (IdP) authentication (OIDC, guest, future methods)
- Token issuance via C4M {{C4M}} for identified access with DPoP binding
- Role-to-scope mapping for the MOCHA namespace hierarchy
- Federated identity exchange across Providers
- Token refresh and revocation

# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology defined in {{MOCHA-ARCH}}, including
Provider, Organization, Team, Channel, Device, and DevID.

The following identifiers are defined in this document:

HClientID:
: A hashed client identifier derived via HMAC. Used in
  namespace tuples and track names instead of the raw
  clientID. See {{id-derivation}}.

HDevID:
: A hashed device identifier derived via HMAC. Used in
  track names instead of the raw devID. See {{id-derivation}}.

The following additional terms are used:

Identity Provider (IdP):
: A service that authenticates users and asserts their identity.
  Examples include OIDC-compliant providers, enterprise SSO, and
  self-hosted identity services. The IdP answers "who is this user?"

Provider Token Service:
: The MOCHA component that validates identity assertions from IdPs,
  determines user permissions, and issues access tokens
  (C4M) for relay access.

Access Tokens: A C4M (CAT for MOQT)) that encodes a
  user's  MOQT namespace permissions. Defined in
  {{C4M}}. When used in Mocha, the token includes a DPoP confirmation
  claim (`cnf`) binding it to the client's key.

ID Token: : An OIDC (OpenID Connect) token that provides the
  authenticated identity of the user.

DPoP (Demonstrating Proof of Possession):
: A mechanism that binds a C4M token to the client's asymmetric
  key pair. The client generates a fresh DPoP proof for each MOQT
  action, preventing token theft and replay. See {{dpop}}.

Authorization Parameter:
: The MOQT parameter that carries authentication credentials. It
  can appear in CLIENT_SETUP, SUBSCRIBE, SUBSCRIBE_NAMESPACE,
  PUBLISH, PUBLISH_NAMESPACE, and FETCH messages.

Relay:
: Validates access tokens and enforces access control for
  MOQT operations. Relays do not authenticate users directly;
  they trust tokens signed by known issuers.

# Architecture Overview

The client does a normal OpenID SSO login flow to authenticate to the
Provider Token Service to get an ID Token and an Access Token with DPoP.

For each Org at the Provider, the Provider keeps tracks of:

* all of the users in the Org,

* for each users in its Org, all of their devices,

* all the teams in its Org,

* for each team, all of  users (in the Org or other Providers/Orgs) and their roles,

* for each team, all of its channels,

* for each of the channels, its users (in this Org or other Providers/Orgs) and their roles,

* for each of the channels, all of the meetings.


The Provider Token Service takes an ID Token and generates Access
Token(s) with the required scopes for Relays for all the Mocha
services. The Provider Token Service coordiates with the CDNs providing
the Relays so the relays can validate the Access Tokens.

The exact scopes needed for Access Tokens are defined in the
specifications that define the Mocha service. The Client will have a
seperate Access Token for each of:

* team the User is part of,

* channel they can access,

* meeting they can access. 


Clients use the aproperate Access Token(s) when sending requests to the Relays.

A Client that is acting as User on a different provider can also ask for
Access Tokens for this resources in this Provider. This allows federated
actions where a Client on Provider A can participated in meetings or
channels that are owned by an Org on Provier B.


# Identifers

Mocha Providers MUST be inditifered by a unique DNS name.

UserIDs are reprsented by a Web DID <TODO ref> that has the constrained form:
```
  did:web:<Provider>:<Org>:<User>
```

In many places, a hashed form of the UserID is used called HUserID. This
is hashed in the context of particar Team, Channel, or Meeting and
includes the full name of the Team, Channel, or Meeting in the hash so
that IDs can not be correlated across different contexts.


Devices are identified by a DevID that is hash of the public key they
use for DPoP.  There is a hashed version of DevID, called HDevID, that
is hasehd like HUserID.


# Identity Provider Authentication {#idp-auth}


Before obtaining authorization tokens, a user must authenticate
with an Identity Provider. The
Token Service validates the IdP assertion and establishes a
session.

## OIDC Authentication

~~~
  Client                   IdP (OIDC)             Token Service
    |                          |                        |
    |  (1) OAuth/OIDC flow     |                        |
    |  (browser redirect or    |                        |
    |   native sign-in)        |                        |
    |------------------------->|                        |
    |                          |                        |
    |  (2) ID Token            |                        |
    |<-------------------------|                        |
    |                          |                        |
    |  (3) POST /auth/oidc     |                        |
    |      Authorization: Bearer <ID Token>             |
    |-------------------------------------------------->|
    |                          |                        |
    |                          |  (4) Verify signature  |
    |                          |      via JWKS fetch    |
    |                          |<-----------------------|
    |                          |----------------------->|
    |                          |                        |
    |                          |  (5) Validate claims:  |
    |                          |      iss, aud, exp,    |
    |                          |      email_verified    |
    |                          |                        |
    |  (6) 200 OK                                      |
    |      { user_id, session_token }                  |
    |<--------------------------------------------------|
    |                          |                        |
~~~
{: #fig-oidc title="OIDC Authentication Flow"}

The Token Service MUST validate the ID Token per {{OIDC}}
Section 3.1.3.7 (issuer, audience, expiry, signature via JWKS).


# C4M Token Issuance and Validation {#c4m}

C4M (CAT for MOQT) {{C4M}} is the primary authorization mechanism
for identified access. MOCHA deployments MUST use DPoP
(Demonstrating Proof of Possession) to bind C4M tokens to the
client's key pair, preventing token theft and replay.

Token format and CWT claims (Section 2), DPoP claims and proof
generation (Section 3.1), and DPoP proof process and token
binding flow (Section 3.2) from {{C4M}} MUST be used for
MOCHA authorization.

## Token Request {#dpop}

After authentication, the client requests a C4M token scoped to
a specific team, channel, meeting and role. The client provides its public key
for DPoP binding. The Token Service validates the session,
checks membership, determines role permissions, and mints a
C4M token with a `cnf` claim binding it to the client's key.

The complete HTTP issuance flow is specified in
{{http-issuance}}.

The Token Service maps the user's role and channel membership
to `moq_scopes` within the C4M token. The scopes use the MOCHA
namespace hierarchy:

~~~
  moq_scopes: {
    publish: ["mocha/<provider>/<org>/<team>/<channel>/participant/*"],
    subscribe: ["mocha/<provider>/<org>/<team>/<channel>/*"]
  }
~~~

## Scope Enforcement

Scope enforcement follows the positional namespace matching
rules defined in {{C4M}}, Section 2.1. The relay evaluates
each namespace field against the corresponding match object
in the token's `moqt` claim. Actions not explicitly permitted
by the token are denied by default.

For example, a MOCHA participant token might encode scopes
that permit publishing under their device prefix within a
channel, while allowing subscription to all tracks in that
channel:

~~~
  Token scope (publish): mocha/provider.example/acme/eng/general/
                         participant/<device-id>/...
  Token scope (subscribe): mocha/provider.example/acme/eng/general/...

  PUBLISH request:  mocha/provider.example/acme/eng/general/
                    participant/device-a3f2/messages/live
  Result: PERMITTED (matches publish scope)

  SUBSCRIBE request: mocha/provider.example/acme/eng/general/
                     moderator/device-b1c4/messages/live
  Result: PERMITTED (matches subscribe scope)
~~~

# Roles and Token Scopes {#roles}

Roles (Owner, Moderator, Participant, Observer)
are  assigned at the  Organization,
Team,  Channel and Meetings levels. The permissions associated with each role
(publish, subscribe, moderation, management) are application
policy specified in {{MOCHA-ARCH}}.

This document defines how the Token Service translates a user's
role into C4M token scopes. When a user requests a token, the
Token Service determines their role for the target channel and
encodes the corresponding namespace permissions:

~~~
  User: alice@provider.example
  Team: acme/eng
  Channel: general
  Role: participant

  Resulting C4M scopes:
    publish:   ["mocha/provider.example/acme/eng/general/participant/*"]
    subscribe: ["mocha/provider.example/acme/eng/general/*"]

  User: bob@provider.example
  Role: moderator

  Resulting C4M scopes:
    publish:   ["mocha/provider.example/acme/eng/general/moderator/*",
                "mocha/provider.example/acme/eng/general/moderation/*"]
    subscribe: ["mocha/provider.example/acme/eng/general/*"]
~~~

The scope patterns follow the MOCHA namespace hierarchy and are
enforced by the relay per {{C4M}}, Section 2.1.


# Federation {#federation}

Federation allows a user authenticated with one Provider to
participate in Teams and Channels hosted by a different Provider.
The home Provider vouches for the user's identity via a signed
assertion, and the remote Provider issues a locally-valid C4M
token after verifying trust and applying its federation policies.

## Federation Model

~~~
  +------------------+                    +------------------+
  | Provider A       |                    | Provider B       |
  | (home)           |                    | (remote)         |
  |                  |    Federation      |                  |
  | Token Service A  |<------------------>| Token Service B  |
  | Relay A          |    Trust           | Relay B          |
  | Users: Alice     |                    | Users: Bob       |
  +------------------+                    +------------------+

  Alice (home: A) wants to join a Channel on Provider B.
~~~
{: #fig-federation title="Federated Providers"}

## Federation Establishment

Providers establish federation trust via mutual discovery and
key exchange:

~~~
  Provider A                                Provider B
      |                                          |
      | (1) GET /.well-known/mocha-federation    |
      |----------------------------------------->|
      |                                          |
      | (2) { realm, federation_relays,          |
      |       signing_key, policies }            |
      |<-----------------------------------------|
      |                                          |
      | (3) Verify signing_key via TLS           |
      |                                          |
      | (4) Store Provider B's public key        |
      |     in trusted federation peers          |
      |                                          |
      | (5) Provider B does same for A           |
      |<---------------------------------------->|
      |                                          |
~~~
{: #fig-fed-establish title="Federation Establishment"}

The well-known document:

~~~
GET https://provider-b.example/.well-known/mocha-federation

{
  "realm": "provider-b.example",
  "federation_relays": ["moqt://relay.provider-b.example:443"],
  "signing_key": "<Ed25519 public key, base64>",
  "token_exchange_endpoint": "https://auth.provider-b.example/federation/exchange",
}
~~~

## Federated Token Exchange {#fed-exchange}

When Alice (Provider A) wants to access a Channel on Provider B,
a token exchange occurs:

~~~
  Alice        Token Service A       Token Service B        Relay B
    |                |                      |                  |
    | (1) Request    |                      |                  |
    |  federated     |                      |                  |
    |  access to     |                      |                  |
    |  provider-b/   |                      |                  |
    |  org/team/ch   |                      |                  |
    |--------------->|                      |                  |
    |                |                      |                  |
    |                | (2) Create signed    |                  |
    |                |  federation assertion|                  |
    |                |  (Ed25519):          |                  |
    |                |  { sub, home_realm,  |                  |
    |                |    target, roles,    |                  |
    |                |    exp }             |                  |
    |                |                      |                  |
    |                | (3) POST /federation/exchange           |
    |                |   { assertion, requested_scope }        |
    |                |--------------------->|                  |
    |                |                      |                  |
    |                |       (4) Verify assertion signature    |
    |                |           against Provider A's          |
    |                |           known public key              |
    |                |                      |                  |
    |                |       (5) Check federation policies:    |
    |                |           - Provider A allowed?         |
    |                |           - Target channel permits      |
    |                |             remote users?               |
    |                |           - Role mapping valid?         |
    |                |                      |                  |
    |                |       (6) Mint C4M token with           |
    |                |           Provider B's signing key      |
    |                |                      |                  |
    |                | (7) { token: <C4M>, expires_at }        |
    |                |<---------------------|                  |
    |                |                      |                  |
    | (8) Federated  |                      |                  |
    |  C4M token     |                      |                  |
    |<---------------|                      |                  |
    |                |                      |                  |
    | (9) CLIENT_SETUP { AUTHORIZATION: <C4M from B> }         |
    |--------------------------------------------------------->|
    |                |                      |                  |
    |                |                      |  (10) Validate   |
    |                |                      |  with B's key    |
    |                |                      |                  |
    | (11) SERVER_SETUP                                        |
    |<---------------------------------------------------------|
    |                |                      |                  |
~~~
{: #fig-fed-exchange title="Federated Token Exchange"}

The home Provider creates an Ed25519-signed assertion containing
the user's identity (issuer, subject), the target Provider and
namespace, the user's home roles, a validity window (issued-at
and expiry), and a fresh nonce to prevent replay. The assertion
is scoped to the remote Provider via an audience claim.

The remote Provider validates the signature against the home
Provider's known public key, checks the assertion has not
expired, and maps the home roles to local permissions according
to its federation policy. The remote Provider MAY downgrade
roles (e.g., a home "moderator" becomes a remote "participant").

# Token Service {#token-service}

The Token Service is the MOCHA component responsible for
minting, refreshing, and revoking C4M access tokens.

The Token Service interacts with other components via two
mechanisms:

HTTP API:
: Clients call the Token Service HTTP endpoint to mint
  initial C4M tokens (before a MOQT session exists).
  See {{http-issuance}}.

MOQT Tracks:
: The Token Service publishes on MOQT tracks that relays
  and clients subscribe to:

  - Per-client refresh track: delivers rotated tokens to
    each client.
  - Revocation track: notifies relays and clients of
    revoked tokens.

TODO details 

## HTTP Token Issuance {#http-issuance}

Before establishing a MOQT session, a client obtains a C4M
token from the Token Service via its HTTP API. The client
presents its session credentials (obtained after IdP
authentication per {{idp-auth}}) along with its DPoP public
key:

~~~
  Client                              Token Service
    |                                       |
    |  POST /token                          |
    |  Authorization: Bearer <session_token>|
    |  {                                    |
    |    "org_id": "acme",                  |
    |    "team_id": "engineering",          |
    |    "channel_id": "general",           |
    |    "role": "participant",             |
    |    "dpop_jwk": { <client public key> }|
    |  }                                    |
    |-------------------------------------->|
    |                                       |
    |  (1) Validate session                 |
    |  (2) Check membership and role        |
    |  (3) Mint C4M token with DPoP binding |
    |                                       |
    |  200 OK                               |
    |  { "token": "<base64 CWT>",           |
    |    "expires_at": 1749903600 }         |
    |<--------------------------------------|
    |                                       |
~~~
{: #fig-http-issue title="HTTP Token Issuance"}

The Token Service maps the user's role and channel membership
to `moq_scopes` within the C4M token per {{roles}}.


# Security Considerations

TODO

# IANA Considerations

This document makes no requests of IANA.

--- back

