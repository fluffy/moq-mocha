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
Providers (IdPs), how authorization tokens (C4M and Privacy Pass)
are issued and validated at relays, how permissions map to MOQT
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
- Token issuance via Privacy Pass {{PrivacyPassAuth}} for anonymous access
- Role-to-scope mapping for the MOCHA namespace hierarchy
- Federated identity exchange across Providers
- Token refresh and revocation

# Conventions and Definitions

See BCP 14 {{!RFC2119}} {{!RFC8174}}.

This document uses terminology defined in {{MOCHA-ARCH}}, including
Provider, Organization, Team, Channel, Device, and DevID.

The following additional terms are used:

Identity Provider (IdP):
: A service that authenticates users and asserts their identity.
  Examples include OIDC-compliant providers, enterprise SSO, and
  self-hosted identity services. The IdP answers "who is this user?"

Token Service:
: The MOCHA component that validates identity assertions from IdPs,
  determines user permissions, and issues authorization tokens
  (C4M or Privacy Pass) for relay access.

Token Issuer (PP-Issuer):
: The Privacy Pass token issuer. Signs blinded token requests so
  that tokens cannot be linked back to the requesting user.

Attester:
: A trusted entity (typically the Token Service) that vouches for
  a client's eligibility to the PP-Issuer without revealing the
  client's identity.

C4M (CAT for MOQT):
: A CWT (CBOR Web Token) that encodes a user's identity and MOQT
  namespace permissions. Defined in {{C4M}}. The token includes a
  DPoP confirmation claim (`cnf`) binding it to the client's key.

DPoP (Demonstrating Proof of Possession):
: A mechanism that binds a C4M token to the client's asymmetric
  key pair. The client generates a fresh DPoP proof for each MOQT
  action, preventing token theft and replay. See {{dpop}}.

AUTHORIZATION Parameter:
: The MOQT parameter that carries authentication credentials. It
  can appear in CLIENT_SETUP, SUBSCRIBE, SUBSCRIBE_NAMESPACE,
  PUBLISH, PUBLISH_NAMESPACE, and FETCH messages.

Relay:
: Validates authorization tokens and enforces access control for
  MOQT operations. Relays do not authenticate users directly;
  they trust tokens signed by known issuers.

# Architecture Overview

~~~
                   MOCHA Identity Architecture

  +----------+                                    +----------+
  |   IdP    |  (OIDC, Enterprise SSO, etc.)      | PP-Issuer|
  +----+-----+                                    +----+-----+
       |                                               |
       | ID Token                          blind sign  |
       v                                               v
  +----+-----+     (2) AUTHORIZATION param     +------+------+
  |  Client  |  -----------------------------> |    Relay    |
  | (Device) |   (C4M + DPoP, or PP token)     | (Validates) |
  +----+-----+                                 +------+------+
       |                                               |
       | (1) authenticate + request token              |
       v                                               |
  +----+-----------------------------------------+     |
  |            Token Service                     |     |
  |  - Validates IdP assertions                  |     |
  |  - Checks membership/permissions             |     |
  |  - Mints C4M tokens with DPoP binding        |     |
  |  - Proxies PP blind token requests           |     |
  +----------------------------------------------+     |
                                                       |
            (3) SUBSCRIBE / PUBLISH granted            |
            <------------------------------------------+
~~~
{: #fig-arch title="Identity Architecture"}

# Identity Provider Authentication {#idp-auth}

## Overview

Before obtaining authorization tokens, a user must authenticate
with an Identity Provider. MOCHA supports pluggable IdPs. The
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
    |  (2) ID Token (JWT)      |                        |
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

## Guest Authentication

For deployments allowing pseudonymous access:

~~~
  Client                             Token Service
    |                                      |
    |  POST /auth/guest                    |
    |  { "display_name": "Alice" }         |
    |------------------------------------->|
    |                                      |
    |  200 OK                              |
    |  { "user_id": "guest-7a3f",          |
    |    "session_token": "s_..." }        |
    |<-------------------------------------|
    |                                      |
~~~
{: #fig-guest title="Guest Authentication"}

Guest users receive a server-assigned pseudonymous identity.
Guest sessions MAY have reduced permissions compared to
authenticated users.

# C4M Token Issuance and Validation {#c4m}

## Overview

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
a specific channel and role. The client provides its public key
for DPoP binding:

~~~
  Client                              Token Service
    |                                       |
    |  POST /token                          |
    |  Authorization: Bearer <session_token>|
    |  {                                    |
    |    "room_id": "engineering",          |
    |    "role": "participant",             |
    |    "dpop_jwk": { <client public key> }|
    |  }                                    |
    |-------------------------------------->|
    |                                       |
    |  (1) Validate session                 |
    |  (2) Check Team/Channel membership    |
    |  (3) Determine role permissions       |
    |  (4) Mint C4M token with cnf claim    |
    |      binding to client's key          |
    |                                       |
    |  200 OK                               |
    |  { "token": "<base64 CWT>",           |
    |    "expires_at": 1749903600 }         |
    |<--------------------------------------|
    |                                       |
~~~
{: #fig-c4m-issue title="C4M Token Issuance"}

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

# Privacy Pass Token Issuance and Validation {#privacy-pass}

## Overview

Privacy Pass {{PrivacyPassAuth}} provides cryptographic
unlinkability: the relay can verify a token was legitimately
issued without correlating it to the identity that requested it.
This enables anonymous participation in channels. Illustrative
call flows are provided in {{pp-call-flows}}.

For scoped access, Partially Blind RSA tokens carry public
metadata encoding the permitted MOCHA namespaces without breaking
unlinkability. The relay uses these extensions for scope
enforcement identical to C4M tokens, but cannot determine which
user holds the token.

## MOCHA Token Issuance

Token types, issuance protocol, attester authentication, and
token wire format are defined in {{PrivacyPassAuth}}. The Token
Service acts as Attester ({{PrivacyPassAuth}}, Section 3) and
authenticates to the PP-Issuer using HTTP Message Signatures
{{RFC9421}}.

In the MOCHA context, the Token Service:

1. Validates the client's session and channel membership (as
   in the C4M flow).

2. Determines the namespace scope to encode in the token's
   public metadata (for PBRS tokens).

3. Attests to the PP-Issuer that the client is authorized for
   the requested scope.

The client presents the resulting token in the AUTHORIZATION
parameter of any MOQT control message (CLIENT_SETUP, SUBSCRIBE,
PUBLISH, etc.).

## Reverse Flow {#pp-reverse}

MOCHA deployments MAY use the Privacy Pass reverse flow
{{PP-REVERSE}} where the relay acts as both attester and issuer.
The client bootstraps with a publicly verifiable token from an
external issuer, and the relay responds with privately verifiable
tokens for subsequent operations.

The reverse flow enables continuous unlinkable authorization
without requiring an external issuer after the initial bootstrap.
Token replenishment is piggybacked on normal MOQT control message
exchanges using batched tokens {{PP-BATCH}}. The bootstrap,
continuous authorization, and batch token encoding are defined in
{{PrivacyPassAuth}} (Sections 4-5) and {{PP-REVERSE}} (Section 4).

# Roles and Token Scopes {#roles}

Roles (Owner, Moderator, Participant, Observer) are defined in
{{MOCHA-ARCH}} and assigned at the Provider, Organization, Team,
and Channel levels. The permissions associated with each role
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
enforced by the relay per {{C4M}}, Section 2.1. The same scopes
apply when using Privacy Pass with PBRS tokens — the namespace
permissions are encoded in the token's public metadata extensions
(see {{PrivacyPassAuth}}) and enforced identically by the relay.

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
      | (3) Verify signing_key via DNSSEC/TLS    |
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
  "policies": {
    "allow_inbound": true,
    "allowed_providers": ["*"],
    "blocked_providers": []
  }
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
    |                |<--------------------|                  |
    |                |                      |                  |
    | (8) Federated  |                      |                  |
    |  C4M token     |                      |                  |
    |<---------------|                      |                  |
    |                |                      |                  |
    | (9) CLIENT_SETUP { AUTHORIZATION: <C4M from B> }        |
    |-------------------------------------------------------->|
    |                |                      |                  |
    |                |                      |  (10) Validate   |
    |                |                      |  with B's key    |
    |                |                      |                  |
    | (11) SERVER_SETUP                                       |
    |<--------------------------------------------------------|
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

# Token Refresh and Revocation {#refresh}

C4M tokens are intentionally short-lived. To maintain
continuous access without requiring the client to
re-authenticate via the IdP, MOCHA provides in-session token
refresh over MOQT. For immediate access removal (e.g., user
banned or role changed), relays subscribe to a revocation
track published by the Token Service.

## In-Session Token Refresh

Once a MOQT session is active, token refresh occurs over MOQT
to avoid HTTP round-trips:

~~~
  Client                Relay              Token Service
    |                     |                      |
    | SUBSCRIBE           |                      |
    | mocha/<provider>/   |                      |
    |   tokens/<user_id>  |                      |
    |-------------------->|                      |
    |                     |--------------------->|
    |                     |                      |
    |                     |      (token nearing  |
    |                     |       expiry)        |
    |                     |                      |
    |                     |  OBJECT: new token   |
    |                     |<---------------------|
    |  new token          |                      |
    |<--------------------|                      |
    |                     |                      |
~~~
{: #fig-refresh title="In-Session Token Refresh"}

## Token Revocation

For immediate revocation (role change, ban):

~~~
  Token Service publishes to:
    Namespace: mocha/<provider>/tokens
    Track: revocations

  Revocation object:
  {
    "user_id": "alice@provider.example",
    "token_id": "<SHA-256 hash of the revoked token>",
    "revoked_at": 1749901500,
    "affected_scopes": ["mocha/provider.example/acme/eng/private/*"]
  }

  Relays subscribed to the revocation track enforce immediately.
~~~

# Service Discovery {#discovery}

TODO

# Security Considerations

TODO

# IANA Considerations

This document makes no requests of IANA.

--- back

# Privacy Pass Call Flows {#pp-call-flows}

This appendix illustrates the Privacy Pass flows for MOCHA.
The normative protocol is defined in {{PrivacyPassAuth}} and
{{PP-REVERSE}}.

## External Issuer Flow

The client authenticates with the IdP, then obtains a Privacy
Pass token via the Token Service (Attester) and external
PP-Issuer, and finally presents it to the relay.

~~~
  Client          IdP        Token Service       PP-Issuer       Relay
    |              |          (Attester)              |              |
    |              |               |                  |              |
    | (1) Authenticate             |                  |              |
    |------------->|               |                  |              |
    |              |               |                  |              |
    | (2) ID Token |               |                  |              |
    |<-------------|               |                  |              |
    |              |               |                  |              |
    | (3) POST /auth/oidc          |                  |              |
    |   { ID Token }               |                  |              |
    |----------------------------->|                  |              |
    |              |               |                  |              |
    | (4) session_token            |                  |              |
    |<-----------------------------|                  |              |
    |              |               |                  |              |
    | (5) POST /auth/privacypass/challenge            |              |
    |   { room_id, token_type }    |                  |              |
    |----------------------------->|                  |              |
    |              |               |                  |              |
    |              |               | (6) Fetch issuer |              |
    |              |               |     public key   |              |
    |              |               |----------------->|              |
    |              |               |<-----------------|              |
    |              |               |                  |              |
    | (7) TokenChallenge +         |                  |              |
    |     issuer_pub_key           |                  |              |
    |<-----------------------------|                  |              |
    |              |               |                  |              |
    | (8) Generate nonce,          |                  |              |
    |     blind token_input        |                  |              |
    |              |               |                  |              |
    | (9) POST /auth/privacypass/token-request        |              |
    |   { blinded_element }        |                  |              |
    |----------------------------->|                  |              |
    |              |               |                  |              |
    |              |               | (10) Forward     |              |
    |              |               |   + HTTP sig     |              |
    |              |               |----------------->|              |
    |              |               |                  |              |
    |              |               |  (11) Verify     |              |
    |              |               |  attester sig,   |              |
    |              |               |  sign token      |              |
    |              |               |                  |              |
    |              |               | (12) blind_sig   |              |
    |              |               |<-----------------|              |
    |              |               |                  |              |
    | (13) blind_signature         |                  |              |
    |<-----------------------------|                  |              |
    |              |               |                  |              |
    | (14) Unblind, assemble       |                  |              |
    |      final token             |                  |              |
    |              |               |                  |              |
    | (15) CLIENT_SETUP { AUTHORIZATION: token }                    |
    |-------------------------------------------------------------->|
    |              |               |                  |              |
    |              |               |                  | (16) Verify  |
    |              |               |                  | with issuer  |
    |              |               |                  | public key   |
    |              |               |                  |              |
    | (17) SERVER_SETUP (identity unknown to relay)                 |
    |<--------------------------------------------------------------|
    |              |               |                  |              |
~~~
{: #fig-pp-external title="Privacy Pass External Issuer Flow"}

## Reverse Flow

The reverse flow has three phases: bootstrap, session
establishment, and continuous authorization with token
replenishment.

### Step 1: Bootstrap

The client connects without credentials. The relay challenges
the client, who obtains a publicly verifiable token from an
external issuer.

~~~
  Client              PP-Issuer (external)          Relay
    |                      |                          |
    | (1) CLIENT_SETUP (no auth)                      |
    |------------------------------------------------>|
    |                      |                          |
    | (2) UNAUTHORIZED { TokenChallenge }             |
    |<------------------------------------------------|
    |                      |                          |
    | (3) Blind token      |                          |
    |   request            |                          |
    |--------------------->|                          |
    |                      |                          |
    | (4) blind_signature  |                          |
    |<---------------------|                          |
    |                      |                          |
    | (5) Unblind, assemble publicly                  |
    |     verifiable token (type 0x0002)              |
    |                      |                          |
~~~
{: #fig-pp-rev-step1 title="Reverse Flow Step 1: Bootstrap"}

### Step 2: Token Exchange

The client presents the bootstrap token and requests a batch
of privately verifiable tokens from the relay (now acting as
issuer).

~~~
  Client                                    Relay (Issuer)
    |                                         |
    | (1) CLIENT_SETUP {                      |
    |       AUTHORIZATION: Token (0x0002),    |
    |       GenericBatchTokenRequest           |
    |     }                                   |
    |---------------------------------------->|
    |                                         |
    |     (2) Validate bootstrap token        |
    |     (3) Issue privately verifiable      |
    |         tokens (type 0x0001/0x0005)     |
    |                                         |
    | (4) SERVER_SETUP {                      |
    |       GenericBatchTokenResponse          |
    |     }                                   |
    |<----------------------------------------|
    |                                         |
~~~
{: #fig-pp-rev-step2 title="Reverse Flow Step 2: Token Exchange"}

### Step 3: Continuous Authorization

On each subsequent operation, the client presents a privately
verifiable token and piggybacks a new batch request. The relay
validates and replenishes tokens.

~~~
  Client                                    Relay
    |                                         |
    | (1) SUBSCRIBE {                         |
    |       AUTHORIZATION: PrivateTokenAuth,  |
    |       GenericBatchTokenRequest           |
    |     }                                   |
    |---------------------------------------->|
    |                                         |
    |     (2) Validate private token          |
    |     (3) Issue new batch of tokens       |
    |                                         |
    | (4) GenericBatchTokenResponse            |
    |<----------------------------------------|
    |                                         |
    | (5) PUBLISH {                           |
    |       AUTHORIZATION: PrivateTokenAuth,  |
    |       GenericBatchTokenRequest           |
    |     }                                   |
    |---------------------------------------->|
    |                                         |
    |     (6) Validate, issue new batch       |
    |                                         |
    | (7) GenericBatchTokenResponse            |
    |<----------------------------------------|
    |                                         |
    |     (continues for session lifetime)    |
    |                                         |
~~~
{: #fig-pp-rev-step3 title="Reverse Flow Step 3: Continuous Authorization"}
