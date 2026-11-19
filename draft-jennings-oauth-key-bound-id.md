---
title: "OAuth 2.0 Key-Bound Identity Assertions"
abbrev: "Key-Bound Identity Assertions"
category: std

docname: draft-jennings-oauth-key-bound-id-00
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - OAuth
 - OpenID Connect
 - Proof of Possession
 - Identity
 - End-to-End Encryption
 - AI Agents
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"

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
  RFC6755:
  RFC7515:
  RFC7517:
  RFC7519:
  RFC7638:
  RFC7800:
  RFC8414:
  RFC8693:
  RFC8725:
  RFC9449:
  OIDC:
    title: "OpenID Connect Core 1.0"
    date: 2014
    target: https://openid.net/specs/openid-connect-core-1_0.html
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0"
    date: 2014
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  SD-JWT: I-D.draft-ietf-oauth-selective-disclosure-jwt
  STATUS-LIST: I-D.draft-ietf-oauth-status-list

informative:
  RFC5280:
  RFC8823:
  RFC9420:
  OID4VCI:
    title: "OpenID for Verifiable Credential Issuance 1.0"
    date: 2025
    target: https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html
  MATRIX:
    title: "Matrix Specification: Cross-Signing"
    target: https://spec.matrix.org/latest/client-server-api/#cross-signing
    date: false
  WEBAUTHN:
    title: "Web Authentication: An API for accessing Public Key Credentials Level 2"
    date: 2021
    target: https://www.w3.org/TR/webauthn-2/
  SD-JWT-VC: I-D.draft-ietf-oauth-sd-jwt-vc
  KEYTRANS: I-D.draft-ietf-keytrans-architecture
  ABCA: I-D.ietf-oauth-attestation-based-client-auth
  WIMSE-CREDS: I-D.ietf-wimse-workload-creds
  CIBA:
    title: "OpenID Connect Client-Initiated Backchannel Authentication Flow - Core 1.0"
    date: 2021
    target: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
  OID4VP:
    title: "OpenID for Verifiable Presentations 1.0"
    date: 2025
    target: https://openid.net/specs/openid-4-verifiable-presentations-1_0.html
  PACE: I-D.nandakumar-agentproto-pace
  PACE-AUTHZ: I-D.nandakumar-agentproto-pace-authz
  PRINCIPAL-GRANTS:
    title: "Principal-Confirmed Delegation Grants"
    date: 2026-11
    author:
    - ins: S. Nandakumar
    - ins: C. Jennings
    seriesinfo:
      Internet-Draft: draft-nandakumar-oauth-principal-confirmed-grants-00
    target: https://datatracker.ietf.org/doc/draft-nandakumar-oauth-principal-confirmed-grants/


--- abstract

OpenID Connect ID Tokens assert that a user authenticated to a
particular client, but they do not bind that assertion to a key
held by the user, and they are bearer tokens that are unsafe to
disclose to third parties. Many applications need an assertion
from the user's identity provider that a long-lived public key
belongs to an authenticated user, which can be published to and
verified by arbitrary parties long after issuance: end-to-end
encrypted messaging, to know whose key a group member holds, and
systems in which software agents act for a user, to tell the
user's own approval or signed statement from one an agent made.
This document defines the
Key-Bound Identity Assertion (KBIA), a JWT issued by an OAuth 2.0
authorization server that binds a user's subject identifier to a
public key whose possession the user has proven. KBIAs are
obtained using OAuth 2.0 Token Exchange with a DPoP proof, are
explicitly typed so that they cannot be confused with access or
ID Tokens, are issued only for keys that sign at the user's
direction, never for keys held by an agent acting on its own,
and optionally attest that the key requires user
verification and support selective disclosure.

--- middle

# Introduction

Many systems need to answer the question "does this public key
belong to alice@example.com?", asked by a party that is not the
relying party Alice signed in to. End-to-end encrypted (E2EE)
communication, such as that built on Messaging Layer Security
(MLS) {{RFC9420}}, asks it of every group member. Systems in
which software agents act for a person ask it whenever a
statement must come from the person rather than from an agent:
an approval of a payment that an agent relays to another
organization, limits the person set for a task that agents carry
out overnight, or a person's identity in a session shared with
agents. {{uses}} describes these needs.

Many deployments already have an authoritative answer to "is
this user alice@example.com?": the organization's OpenID Connect
(OIDC) {{OIDC}} identity provider (IdP). What is missing is a
standard way for the IdP to assert the binding between that
identity and a key the user holds.

Existing mechanisms each meet part of this need. ID Tokens
are IdP-signed but bearer, short-lived, and name no user key.
DPoP-bound access tokens are key-bound but are not identity
assertions for third parties. Certificate authorities and verifiable credential ecosystems
can express the binding but require infrastructure beyond the
IdP. {{gaps}} describes these mechanisms in more detail.

This document fills the gap with a narrowly scoped token: the
Key-Bound Identity Assertion (KBIA). A KBIA is a JWT {{RFC7519}}
signed by the authorization server (AS) that states that the
subject `sub` at issuer `iss` proved possession of the public key
in its `cnf` claim {{RFC7800}}. A client obtains a KBIA from the
AS's token endpoint using OAuth 2.0 Token Exchange {{RFC8693}},
proving possession of the key with a DPoP proof {{RFC9449}}.

A KBIA is designed to be published. It is not a bearer
credential: possession of a KBIA confers nothing, and relying on
it requires a signature by the bound key. Its lifetime is
independent of session tokens, and the AS keeps its signing keys
verifiable for that lifetime.

Because a KBIA attributes every signature of the bound key to
the End-User, the key must sign only at the End-User's
direction. The AS issues KBIAs only to clients that act at the
End-User's direction, such as an application on the End-User's
device or a wallet whose key only the End-User's approval
unlocks, and never to a client that acts for the End-User on its
own, such as an AI agent ({{key-holder}}). An agent acting for the
End-User has its own identity and a delegation credential naming
it as actor; it can relay signatures made with the bound key but
cannot make them. A KBIA can also state that the key signs only
after the End-User is verified ({{uv}}), so that a signature shows
the End-User was present.

## Goals

The goal is an assertion, issued by an existing OAuth 2.0
authorization server, that binds a user's identity to a
user-held key, is safe to publish, is verifiable by any peer for
its lifetime, supports revocation and selective disclosure, and
reuses existing OAuth mechanisms
(Token Exchange, DPoP, `cnf`, AS metadata, SD-JWT, and Token
Status List) rather than defining new protocols.

## Non-Goals

- Defining how applications use the bound key (for example, as
  an MLS credential or to sign an approval); that is left to
  application profiles.
- Identifying software agents. An agent that acts for an
  End-User has its own workload identity {{WIMSE-CREDS}} and
  holds its authority in delegation credentials that name it as
  actor {{RFC8693}}.
- Detecting a compromised IdP that issues KBIAs for keys the
  user does not hold; see {{idp-compromise}}.
- Replacing general-purpose verifiable credential ecosystems.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms "authorization server", "client",
"resource server", and "access token" from OAuth 2.0, and "ID
Token" and "End-User" from {{OIDC}}.

Key-Bound Identity Assertion (KBIA):
: A JWT, defined in {{kbia-format}}, in which an authorization
  server asserts that the End-User identified by `iss` and `sub`
  proved possession of the key in `cnf`.

Bound Key:
: The public key carried in a KBIA's `cnf` claim. Its private
  half is held by the End-User's client or clients.

Verifier:
: Any party that validates a KBIA, typically a peer of the
  End-User in an application protocol. A verifier need not be a
  client of the AS.

# Where a Key-Bound Identity Is Needed {#uses}

In each of the following, a party that is not the relying party
of the End-User's sign-in must attribute a signature to the
End-User. None can use an ID Token, which would be a replayable
bearer credential in that party's hands.

Group membership in end-to-end encrypted communication:
: An MLS {{RFC9420}} group member checks that the credential in
  each LeafNode belongs to the person it claims to name, often
  long after that person joined.

A person's approval relayed by agents:
: An agent acting for the End-User asks a service in another
  organization to perform an action, such as a payment, that the
  service will perform only with the End-User's own approval.
  Agents carry the request to the End-User and the approval back,
  so the approval must be signed with a key the End-User holds,
  and the service, which never interacted with the End-User,
  must know that the key is the End-User's. {{PACE-AUTHZ}} is an
  example of a protocol with this need.

Authority set in advance:
: The End-User sets limits for a task, such as a price ceiling,
  that agents carry out while the End-User is unavailable. The
  limits signed with the End-User's key travel with the chain of
  delegations and are verified later, offline, by parties that
  never interacted with the End-User or its identity provider.
  {{PRINCIPAL-GRANTS}} is an example.

People in sessions with agents:
: A conference or multi-party dialog among people and agents
  spans organizations, and each participant needs to know which
  participants are which people. {{PACE}} is an example.

Signed records:
: An auditor or a party to a dispute checks, after the fact,
  that the End-User signed an approval or a message.

The approvals and limits above are meaningful only if the
End-User, not an agent, made the signature. Agents still act on
their own for the End-User, under delegation credentials that
name them; what they cannot do is sign as the End-User. A
key-bound identity for these uses must therefore sign only at the
End-User's direction ({{key-holder}}), and approvals need evidence
that the End-User was present when the key signed ({{uv}}).

# Existing Mechanisms {#gaps}

This section describes existing mechanisms that bind, or could
be used to bind, a user's identity to a user-held key, and
where each falls short of an IdP-issued binding that can be
safely published and verified by any peer.

## OpenID Connect ID Tokens

An ID Token {{OIDC}} asserts that the End-User authenticated to
a specific client (its `aud`). It names no user-held key. It is
a bearer artifact: a party that obtains it can present it to
any relying party that accepts ID Tokens for the same audience.
Third parties can verify its signature but are not its intended
audience. It typically expires within an hour, and its
signature can no longer be verified once the IdP removes the
signing key from its JWKS. It discloses every claim it contains
and cannot be revoked.

## Mechanisms for Approvals

Agent systems obtain a person's approval in several ways, each of
which binds the approval to the person only for some verifiers.
A WebAuthn {{WEBAUTHN}} assertion is signed by the person's
authenticator, but its key is registered with one relying party,
so other parties cannot tell whose key it is. A token issued
after a back-channel approval through CIBA {{CIBA}} is signed by
the identity provider, not the person, and convinces only
parties that trust that provider to have asked. A verifiable
presentation {{OID4VP}} binds the approval to a credential's key,
but requires a wallet and a credential issuer ({{sd-jwt-vc}}).

## DPoP-Bound Access Tokens

DPoP {{RFC9449}} binds an access token to a client-held key
through `cnf.jkt`, with proof of possession at issuance.
However, an access token expresses authorization for a resource
server, not an identity assertion for third parties. It may be
opaque, is audience-restricted to resource servers, and is
short-lived. Sender-constraining prevents reuse without the
key, but access tokens are not designed for publication.

## Enterprise X.509 PKI

An enterprise certificate authority, possibly using ACME for
S/MIME {{RFC8823}}, can issue certificates {{RFC5280}} binding
a user key to an email address or other identity after
checking a certificate signing request. Certificates are safe
to publish, long-lived, and revocable. However, the binding is
asserted by the CA, which in most deployments is separate from
the IdP, and certificates disclose their full subject.
Operating a PKI for end users is a significant burden that many
organizations with a modern IdP have moved away from.

## SD-JWT Verifiable Credentials {#sd-jwt-vc}

An SD-JWT VC {{SD-JWT-VC}} with holder key binding, issued
through OpenID for Verifiable Credential Issuance {{OID4VCI}},
can express the required statement, with selective disclosure
and status lists. Deploying it requires the IdP to act as a
credential issuer and the client to act as a wallet, with the
associated issuance protocol, credential type metadata, and
trust frameworks. Enterprise IdPs and communication clients
generally do not otherwise need this machinery.

A KBIA can be viewed as a narrowly profiled holder-bound
credential. Should the working group prefer it, the KBIA could
instead be defined as an SD-JWT VC type issued through the
token endpoint, retaining the issuance flow and verification
rules defined here.

## Approaches Without an IdP

Several systems avoid IdP-asserted bindings altogether:

- Out-of-band verification, such as the safety numbers used by
  end-to-end encrypted messengers, compares key fingerprints
  between users. It provides strong assurance independent of
  any provider but does not scale to first contact between
  users who have not met.
- User-managed key hierarchies, such as Matrix cross-signing
  {{MATRIX}}, let users sign their device keys with a master
  key, but the master key is bound to the identity only by the
  homeserver or by out-of-band verification.
- Key transparency {{KEYTRANS}} publishes the mapping from
  identities to keys in an auditable log, so that users can
  detect keys published for them that they did not create.
  It addresses detection of misbehaving providers rather than
  the issuance of the binding, and complements an IdP-asserted
  binding rather than replacing it.

WebAuthn {{WEBAUTHN}} credentials are user-held keys with
possession verified by the relying party, but they are scoped
to a single relying party, and attestation describes the
authenticator rather than the user's identity. They do not
provide a binding verifiable by third parties.

# Overview

~~~
  Client (holds user key K)          Authorization Server
    |                                         |
    | (1) OIDC / OAuth login (unchanged)      |
    |---------------------------------------->|
    |     access token or ID Token            |
    |<----------------------------------------|
    |                                         |
    | (2) POST /token                         |
    |     grant_type=token-exchange           |
    |     subject_token=<access/ID token>     |
    |     requested_token_type=kbia           |
    |     DPoP: <proof signed by K>           |
    |---------------------------------------->|
    |                                         |  verify subject
    |                                         |  token, DPoP proof,
    |                                         |  key policy
    |     KBIA {iss, sub, cnf.jwk=K, exp}     |
    |<----------------------------------------|
    |
    | (3) Publish KBIA to peers
    v
  Verifier
    - validates KBIA signature (AS JWKS), typ, exp, status
    - trusts K for (iss, sub)
    - relies on statements signed by K
~~~
{: #fig-overview title="KBIA Issuance and Use"}

1. The client authenticates the End-User using any existing
   OAuth or OIDC flow and obtains an access token or ID Token.
2. The client requests a KBIA from the token endpoint using
   Token Exchange, attaching a DPoP proof signed by the key to
   be bound.
3. The AS verifies the request and issues a KBIA naming the
   key. The client publishes the KBIA to peers, who validate it
   and then rely on signatures made with the bound key.

# KBIA Format {#kbia-format}

## JOSE Header

A KBIA is a JWS {{RFC7515}} in compact serialization. Its JOSE
header:

- MUST contain `typ` with the value `kbia+jwt`, as recommended
  by explicit typing in {{RFC8725}}.
- MUST contain `alg` identifying an asymmetric signature
  algorithm. `none` and MAC-based algorithms MUST NOT be used.
- MUST contain `kid` identifying the AS signing key.

## Claims

The KBIA payload is a JSON object with the following claims:

`iss`:
: REQUIRED. The issuer identifier of the AS {{RFC8414}}.

`sub`:
: REQUIRED. The subject identifier of the End-User. The AS MUST
  use the same `sub` value it would place in an ID Token for the
  requesting client, so that applications can correlate the
  KBIA with the End-User's OIDC identity.

`cnf`:
: REQUIRED. A confirmation claim {{RFC7800}} containing a `jwk`
  member with the bound public key as a JWK {{RFC7517}}. The JWK
  MUST NOT contain private key material. A `jwk` member is used
  rather than `jkt` so that verifiers can verify signatures by
  the bound key without obtaining it separately.

`iat`:
: REQUIRED. The time of issuance.

`exp`:
: REQUIRED. The expiration time. See {{lifetime}}.

`nbf`:
: OPTIONAL. The time before which the KBIA is not valid.

`jti`:
: REQUIRED. A unique identifier for the KBIA.

`auth_time`:
: REQUIRED. The time at which the End-User last authenticated
  to the AS, as defined in {{OIDC}}.

`acr`, `amr`:
: OPTIONAL. As defined in {{OIDC}}, describing the
  authentication that preceded issuance.

`status`:
: OPTIONAL. A reference to a Token Status List entry
  {{STATUS-LIST}} that the AS uses to revoke the KBIA.

`key_uv`:
: OPTIONAL. `true` if the AS verified at issuance that the bound
  key signs only after the End-User is verified ({{uv}}). Absent
  otherwise.

The KBIA MUST NOT contain an `aud` claim. A KBIA is intended
for any verifier, and its absence ensures that relying parties
that check `aud` will not accept a KBIA in place of an ID Token
or access token.

The AS MAY include End-User claims defined in {{OIDC}}
Section 5.1, such as `email` and `email_verified`, that the
client requested ({{issuance}}). The AS SHOULD include only
claims the client explicitly requested.

Example payload:

~~~json
{
  "iss": "https://idp.example.com",
  "sub": "248289761001",
  "cnf": {
    "jwk": {
      "kty": "OKP",
      "crv": "Ed25519",
      "x": "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo"
    }
  },
  "iat": 1762041600,
  "exp": 1769817600,
  "jti": "kbia-6c1e3f2a",
  "auth_time": 1762041580,
  "email": "alice@example.com",
  "email_verified": true
}
~~~

## Selectively Disclosable Form {#sd-form}

An AS MAY issue a KBIA as an SD-JWT {{SD-JWT}}. In that case
the `typ` header value is `kbia+sd-jwt`. The claims `iss`,
`sub`, `cnf`, `iat`, `exp`, `nbf`, `jti`, `auth_time`,
`status`, and `key_uv` MUST NOT be selectively disclosable. End-User claims
such as `email` SHOULD be selectively disclosable, so that the
holder can choose which claims each verifier learns.

When a KBIA is presented in SD-JWT form, a Key Binding JWT
{{SD-JWT}} is not required, because the
application protocol establishes possession of the bound key
({{verification}}).

# Issuance {#issuance}

## Request

A client requests a KBIA using Token Exchange {{RFC8693}} at the
AS's token endpoint, with the following parameters:

`grant_type`:
: REQUIRED. `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`:
: REQUIRED. An access token or ID Token previously issued by
  this AS to the requesting client for the End-User.

`subject_token_type`:
: REQUIRED. `urn:ietf:params:oauth:token-type:access_token` or
  `urn:ietf:params:oauth:token-type:id_token`.

`requested_token_type`:
: REQUIRED. `urn:ietf:params:oauth:token-type:kbia`.

`scope`:
: OPTIONAL. Space-delimited OIDC scope values (for example,
  `email`) indicating End-User claims to include in the KBIA.

`kbia_format`:
: OPTIONAL. `jwt` (default) or `sd-jwt`.

The request MUST include a DPoP proof {{RFC9449}} in the `DPoP`
header, signed by the private half of the key to be bound. The
client MUST authenticate to the token endpoint if it is a
confidential client.

~~~
POST /token HTTP/1.1
Host: idp.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVkRFNBIiwiandrIjp7...

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ...
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid_token
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Akbia
&scope=email
~~~

## Authorization Server Processing

The AS MUST:

1. Validate the subject token, and verify that it was issued by
   this AS to the requesting client and has not expired.
2. Validate the DPoP proof per {{RFC9449}} Section 4.3. The AS
   MUST require a server-provided DPoP `nonce`
   ({{RFC9449}} Section 8) so that a proof cannot be
   pre-computed or replayed.
3. Verify that the key in the DPoP proof's `jwk` header uses an
   algorithm listed in `kbia_key_alg_values_supported`
   ({{metadata}}).
4. Verify that the End-User's authentication is recent enough
   under AS policy, using the `auth_time` associated with the
   subject token. The AS MAY instead respond with the error
   `interaction_required`, prompting the client to
   re-authenticate the End-User.
5. Verify that the client acts only at the End-User's direction
   ({{key-holder}}).
6. Apply any policy on which End-Users and clients may obtain
   KBIAs.

If all checks succeed, the AS issues a KBIA whose `cnf.jwk` is
the public key from the DPoP proof's `jwk` header, and whose
`sub` is the subject of the subject token.

The AS MUST NOT issue a KBIA with a `cnf` key that differs from
the DPoP proof key. The AS MUST NOT accept a KBIA as a
`subject_token` or as any other credential at any endpoint.

## Response

The response follows {{RFC8693}} Section 2.2:

~~~json
{
  "access_token": "eyJ0eXAiOiJrYmlhK2p3dCIsImFsZyI6IlJTMjU2...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:kbia",
  "token_type": "N_A",
  "expires_in": 7776000
}
~~~

The `access_token` member carries the KBIA, as required by the
Token Exchange response syntax; despite the member name, the
KBIA is not an access token. The `token_type` value `N_A`
indicates that it MUST NOT be used as an access token. No
refresh token is issued.

## Errors

In addition to the errors in {{RFC8693}} and {{RFC9449}}, the
AS uses:

`invalid_dpop_proof`:
: The DPoP proof is missing, invalid, or uses an unsupported
  key algorithm.

`interaction_required`:
: The End-User must re-authenticate before a KBIA can be issued.

# Key Holder {#key-holder}

A KBIA states that the End-User controls the bound key, and
verifiers attribute every signature of that key to the End-User.
What matters is who directs a signature, not where the key is
stored: every client is software acting for the End-User, but a
messaging application or a wallet signs only when the End-User
asks it to, while an AI agent or another autonomous service
decides for itself what to sign. If such software held the key,
whatever it signed would be taken as the End-User's own
statement, even if the End-User operates it.

An AS MUST issue KBIAs only to clients that act at the End-User's
direction, and MUST NOT issue a KBIA to a client that acts for
the End-User on its own. Clients that act at the End-User's
direction include an application on the End-User's phone or
computer, and a wallet or remote signing service whose key only
the End-User's approval unlocks, for example with a passkey. The
AS determines this from the client's registration, or from a
client attestation {{ABCA}} that identifies the client
application. Software that hosts both, such as an assistant
running in the same application or on the same device, MUST keep
the bound key out of the agent's reach, for example in a platform
authenticator that signs only after user verification ({{uv}}).

This does not limit what agents can do for the End-User. An agent
acts under its own identity, such as a workload credential
{{WIMSE-CREDS}}, with the End-User's authority conveyed in a
delegation credential that names the End-User as subject and the
agent as actor {{RFC8693}}, so that verifiers see both who acted
and for whom. KBIAs cover the statements that must be the
End-User's own, such as approving an action or setting the limits
of a task. An agent can relay a KBIA and signatures made with its
bound key, but cannot make them.

# User Verification {#uv}

A signature by the bound key shows that the key was used, not
that the End-User was present. `auth_time` describes the
authentication before issuance, which may be months earlier. An
application in which a signature stands for the End-User's
decision, such as an approval ({{uses}}), needs a key that signs
only after the End-User is verified, as platform authenticators,
hardware-backed keys with user-authentication requirements, and
wallets unlocked by a passkey do.

The AS includes `key_uv` with the value `true` only when it has
verified, from a key attestation for the bound key, that the key
has this property. It MUST NOT include `key_uv` otherwise.

Editor's note: How the client conveys a key attestation in the
token request is to be defined, for example alongside the client
attestation of {{ABCA}}.

Freshness of a signature comes from the application: the bound
key signs a value the verifier chose, or one bound to the
request being approved, and the verifier checks the time of the
signature.

# Lifetime and Signing Keys {#lifetime}

KBIAs are intended to be verifiable long after issuance, for
example by a peer that first encounters a user weeks after the
user's key was bound. The AS determines the KBIA lifetime by
policy; lifetimes of days to months are expected. The AS
advertises its maximum lifetime in `kbia_max_lifetime`
({{metadata}}).

The AS MUST keep the public key used to sign a KBIA available
at its `jwks_uri` {{RFC8414}}, or at `kbia_jwks_uri` if
advertised, until every KBIA signed with that key has expired.
This allows the AS to rotate keys used for other tokens without
invalidating outstanding KBIAs. An AS that wishes to keep its
main JWKS small SHOULD sign KBIAs with dedicated keys published
at `kbia_jwks_uri`.

A client SHOULD obtain a new KBIA before the current one
expires, and MUST obtain a new KBIA when it changes its bound
key.

# Revocation

An AS that supports revocation includes a `status` claim
referencing a Token Status List {{STATUS-LIST}}. The AS SHOULD
revoke a KBIA when the End-User's account is disabled, when the
End-User or an administrator reports the bound key as
compromised, or when the End-User's `sub` is reassigned.

Verifiers that require timely revocation SHOULD check the
status list. Applications that operate offline may rely on
cached status, accepting the resulting delay in learning of
revocations.

# Verification {#verification}

A verifier validating a KBIA MUST:

1. Check that the `typ` header is `kbia+jwt` (or `kbia+sd-jwt`
   for the selectively disclosable form) and reject any other
   value.
2. Check that `iss` is an issuer the verifier trusts for the
   application, and obtain its signing keys from the issuer's
   metadata ({{RFC8414}} or {{OIDC-DISCOVERY}}), using
   `kbia_jwks_uri` if present and `jwks_uri` otherwise.
3. Verify the signature using the key identified by `kid`.
4. Check that the current time is before `exp` and, if present,
   at or after `nbf`, allowing for reasonable clock skew.
5. If a `status` claim is present and local policy requires it,
   check that the KBIA has not been revoked.
6. Check that `cnf.jwk` is present, contains no private key
   members, and uses an algorithm acceptable to the
   application.
7. For the SD-JWT form, verify all presented Disclosures per
   {{SD-JWT}}.

A valid KBIA tells the verifier that the AS authenticated the
End-User `sub`, observed proof of possession of the bound key,
and issued it to a client that acts at the End-User's direction
({{key-holder}}). It does not, by itself, tell the verifier that the party
presenting the KBIA holds the bound key. A verifier MUST NOT
treat possession or presentation of a KBIA as authentication
of the presenter. Applications establish that a peer holds the
bound key by verifying signatures made with it, for example a
signature over a verifier-chosen challenge, over application
messages, or over a credential such as an MLS LeafNode.

A verifier that treats a signature by the bound key as the
End-User's decision, such as an approval, SHOULD require
`key_uv`, and checks the signature's freshness as the
application profile specifies ({{uv}}).

Verifiers MAY identify the bound key by its JWK thumbprint
{{RFC7638}}, for example for display and out-of-band comparison.

# Authorization Server Metadata {#metadata}

This document defines the following AS metadata parameters
{{RFC8414}}:

`kbia_signing_alg_values_supported`:
: REQUIRED if the AS supports KBIAs. JSON array of JWS `alg`
  values the AS uses to sign KBIAs.

`kbia_key_alg_values_supported`:
: REQUIRED if the AS supports KBIAs. JSON array of JWS `alg`
  values the AS accepts for bound keys.

`kbia_formats_supported`:
: OPTIONAL. JSON array containing `jwt` and/or `sd-jwt`.
  Defaults to `["jwt"]`.

`kbia_max_lifetime`:
: OPTIONAL. The maximum KBIA lifetime in seconds.

`kbia_jwks_uri`:
: OPTIONAL. URL of a JWK Set containing the keys used to sign
  KBIAs, including retired keys still needed under
  {{lifetime}}.

A client determines support by the presence of
`kbia_signing_alg_values_supported` and
`urn:ietf:params:oauth:grant-type:token-exchange` in
`grant_types_supported`.

# Application Profiles

This document defines the assertion; applications define how
the bound key is used. An application profile SHOULD specify:

- Which issuers verifiers accept, and how issuer trust is
  configured.
- Which End-User claims, if any, are used to name the user,
  and whether `email_verified` is required.
- How the KBIA is distributed to verifiers.
- How possession of the bound key is demonstrated to
  verifiers.
- How changes of the bound key are handled, including whether
  the old key signs a statement endorsing the new key.
- Whether `key_uv` is required, and how a signature is bound to
  a fresh value.

For example, an E2EE messaging application might publish the
KBIA alongside a chain of statements signed by the bound key
that authorize each of the user's device keys, and use the
device keys in MLS credentials.

An agent system might carry a person's approval of an action as
a signature by the bound key over a hash of the request, together
with the KBIA, so that the service performing the action and any
later auditor can verify that the person approved it, while the
agents that relay it can neither alter nor forge it.
{{PACE-AUTHZ}} and {{PRINCIPAL-GRANTS}} define such uses.

# Security Considerations

## Confusion with Other Tokens

KBIAs are designed for wide distribution, so they must never be
accepted as authentication or authorization tokens. This
document mitigates confusion by explicit typing ({{RFC8725}}),
by omitting `aud`, by the `N_A` token type, and by forbidding
the AS from accepting KBIAs as input. Resource servers and
relying parties MUST reject JWTs whose `typ` is `kbia+jwt` or
`kbia+sd-jwt`.

## Keys Held by Agents

If an AS issued a KBIA for a key held by an agent, every
statement the agent signed would be attributed to the End-User,
and approvals meant to constrain the agent would be made by the
agent itself. {{key-holder}} forbids this. Verifiers MUST NOT
accept as the End-User's key a key that appears only in a
credential whose subject or actor is an agent.

## Proof of Possession at Issuance

The DPoP proof with a server-provided nonce shows the AS that
the requesting client holds the private key at the time of the
request. Without it, a client could obtain a KBIA for a key
belonging to someone else, causing verifiers to attribute that
key's signatures to the wrong user.

## Bound Key Compromise

An attacker holding the private bound key can impersonate the
End-User to any verifier that accepts the KBIA, until it
expires or is revoked. Clients SHOULD store bound keys in
non-exportable, hardware-backed storage where available. The
AS SHOULD offer a means for End-Users and administrators to
revoke KBIAs.

## Compromise of the Subject Token

An attacker that steals a subject token and holds its own key
could request a KBIA binding its key to the End-User. The AS
SHOULD prefer sender-constrained subject tokens (such as
DPoP-bound access tokens), SHOULD require recent
authentication ({{issuance}}), and MAY require that the DPoP
key used to obtain the subject token match the key to be
bound.

## IdP Compromise {#idp-compromise}

A compromised or malicious AS can issue KBIAs binding a user's
identity to an attacker's key, and verifiers relying solely on
KBIAs cannot detect this. Applications needing protection
against this threat SHOULD combine KBIAs with out-of-band key
verification or with a key transparency system {{KEYTRANS}},
which allows End-Users to detect keys issued for them that
they did not request.

## Assertion Lifetime

Long KBIA lifetimes improve offline verifiability but extend
the window in which a compromised bound key or a reassigned
`sub` is accepted. Deployments SHOULD balance lifetime against
the availability of revocation checking.

# Privacy Considerations

A KBIA links the End-User's `sub` and any included claims to a
long-lived key, and is shown to many verifiers. Verifiers can
use the stable `sub` and key to correlate the End-User across
contexts. ASs SHOULD include only End-User claims the client
requested and SHOULD support the selectively disclosable form
({{sd-form}}). Pairwise subject identifiers ({{OIDC}} Section
8.1) are generally unsuitable for KBIAs, since verifiers in
different contexts need to recognize the same End-User; an AS
that uses pairwise identifiers MUST use the identifier for the
requesting client.

In agent systems, a KBIA travels with requests to parties the
End-User never contacted, each of which learns `sub` and the
bound key. Where the End-User's identity is not needed, for
example where a chain of delegations names the End-User only by
a pseudonym, applications SHOULD NOT attach a KBIA.

Status list checks may reveal to the AS which KBIAs a verifier
is checking; see {{STATUS-LIST}} for mitigations.

# IANA Considerations

## OAuth URI Registration

IANA is requested to register the following value in the
"OAuth URI" registry established by {{RFC6755}}:

URN:
: urn:ietf:params:oauth:token-type:kbia

Common Name:
: Token type URI for a Key-Bound Identity Assertion

Change Controller:
: IETF

Specification Document:
: This document

## OAuth Authorization Server Metadata

IANA is requested to register the following parameters in the
"OAuth Authorization Server Metadata" registry established by
{{RFC8414}}, each with Change Controller "IETF" and
Specification Document "{{metadata}} of this document":

- `kbia_signing_alg_values_supported`
- `kbia_key_alg_values_supported`
- `kbia_formats_supported`
- `kbia_max_lifetime`
- `kbia_jwks_uri`

## JSON Web Token Claims

IANA is requested to register the following claim in the "JSON
Web Token Claims" registry established by {{RFC7519}}:

Claim Name:
: key_uv

Claim Description:
: Bound key requires user verification

Change Controller:
: IETF

Specification Document(s):
: {{uv}} of this document

## OAuth Parameters

IANA is requested to register the following parameter in the
"OAuth Parameters" registry:

Name:
: kbia_format

Parameter Usage Location:
: token request

Change Controller:
: IETF

Specification Document:
: {{issuance}} of this document

## Media Types

IANA is requested to register the media types
`application/kbia+jwt` and `application/kbia+sd-jwt` in the
"Media Types" registry, for use in the `typ` header as
described in {{kbia-format}}. Full registration templates will
be provided in a future revision.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
