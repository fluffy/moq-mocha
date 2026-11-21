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
  RFC6749:
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
  OPR: I-D.aravind-oauth-operator-of-record
  CONDITION-KEYS: I-D.winmagic-oauth-condition-bound-keys
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


--- abstract

This document defines the Key-Bound Identity Assertion (KBIA), a JWT
that an OAuth 2.0 authorization server issues, through Token Exchange
with a DPoP proof, to bind a user's identity to a public key the user
holds. Unlike an ID Token, a KBIA is not a bearer credential and can
be published, so any party can verify that a signature came from that
user, as end-to-end encrypted messaging and agent systems require.
The bound key can endorse further keys, each short-lived and limited
to one use, such as a session, a dialog, or one verifier.

--- middle

# Introduction

This document defines the Key-Bound Identity Assertion (KBIA), a
JWT {{RFC7519}} signed by the authorization server (AS) stating
that the subject `sub` at issuer `iss` proved possession of the
public key in its `cnf` claim {{RFC7800}}. A client obtains a
KBIA through OAuth 2.0 Token Exchange {{RFC8693}}, proving
possession of the key with a DPoP proof {{RFC9449}}.

A KBIA is designed to be published. Possession of it confers
nothing, and relying on it requires a signature by the bound key.
Because every such signature is attributed to the End-User, the
AS issues KBIAs only to clients that act at the End-User's
direction, never to an agent acting on its own, which can relay
signatures made with the key but not make them ({{key-holder}}).
A KBIA can also state that the key signs only after the End-User
is verified ({{uv}}).

The bound key can endorse further
keys, each short-lived and limited to one use, such as one
session, one dialog, or one audience ({{endorsements}}).

## Goals

The goal is an assertion, issued by an existing OAuth 2.0
authorization server, that:

- binds a user's identity to a key the user holds;
- is safe to publish and verifiable by any peer for its
  lifetime;
- lets the user sign either with the bound key or with a
  short-lived key that the bound key endorsed for one use
  ({{endorsements}});
- supports revocation and selective disclosure; and
- reuses existing OAuth mechanisms (Token Exchange, DPoP, `cnf`,
  AS metadata, SD-JWT, and Token Status List) rather than
  defining new protocols.

## Non-Goals

- Defining how applications use the bound key (for example, as
  an MLS credential or to sign an approval); that is left to
  application profiles.
- Identifying software agents. An agent that acts for an
  End-User has its own workload identity {{WIMSE-CREDS}} and
  holds its authority in delegation credentials that name it as
  actor {{RFC8693}}.
- Detecting a compromised IdP that issues KBIAs for keys the
  user does not hold.
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
of the End-User's sign-in needs to attribute a signature to the
End-User. Using ID Token is a non-starter which would be a replayable
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
  must know that the key is the End-User's.

Authority set in advance:
: The End-User sets limits for a task, such as a price ceiling,
  that agents carry out while the End-User is unavailable. The
  limits signed with the End-User's key travel with the chain of
  delegations and are verified later, offline, by parties that
  never interacted with the End-User or its identity provider.

People in sessions with agents:
: A conference or multi-party dialog among people and agents
  spans organizations, and each participant needs to know which
  participants are which people. {{PACE}} is an example.

The approvals and limits above are meaningful only if the
End-User, not an agent, made the signature. In agentiuc flows,
the agents still act on their own for the End-User, under 
delegation credentials that name them; what they cannot do is sign 
as the End-User. A key-bound identity for these uses hence therefore 
sign only at the End-User's direction ({{key-holder}}), and approvals 
need evidence that the End-User was present when the key signed ({{uv}}).

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
  member with the bound public key as a JWK {{RFC7517}}. 
  A `jwk` member is used rather than `jkt` so that verifiers 
  can verify signatures by the bound key without obtaining 
  it separately.

`iat`:
: REQUIRED. The time of issuance.

`exp`:
: REQUIRED. The expiration time, set by AS policy.

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
: OPTIONAL. `true` means the bound key cannot sign until the
  End-User is verified, for example by PIN or biometric, and that
  the AS checked this when it issued the KBIA ({{uv}}). The AS
  omits the claim when it has not checked this.

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

A client requests a KBIA with a Token Exchange {{RFC8693}}
request to the AS's token endpoint, using these parameters:

`grant_type`:
: REQUIRED. `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`:
: REQUIRED. An access token or ID Token that this AS issued to
  the requesting client for the End-User.

`subject_token_type`:
: REQUIRED. `urn:ietf:params:oauth:token-type:access_token` or
  `urn:ietf:params:oauth:token-type:id_token`.

`requested_token_type`:
: REQUIRED. `urn:ietf:params:oauth:token-type:kbia`.

`scope`:
: OPTIONAL. OIDC scope values, separated by spaces, naming the
  End-User claims to include in the KBIA, for example `email`.

`kbia_format`:
: OPTIONAL. `jwt` for a KBIA as in {{kbia-format}}, the default,
  or `sd-jwt` for the selectively disclosable form
  ({{sd-form}}).

The request MUST carry a DPoP proof {{RFC9449}} in the `DPoP`
header, signed with the key to be bound. A confidential client
MUST also authenticate to the token endpoint.

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

The AS checks the request as follows, and rejects it with the
error named in each step if the check fails ({{errors}}).

1. The subject token is valid, was issued by this AS to the
   requesting client for the End-User, and has not expired. If
   not, the AS returns `invalid_request` ({{RFC8693}} Section
   2.2.2).
2. The DPoP proof is valid as {{RFC9449}} Section 4.3 specifies.
   If not, the AS returns `invalid_dpop_proof`.
3. The DPoP proof contains a nonce that the AS issued recently
   ({{RFC9449}} Section 8), which shows the proof was made just
   now rather than in advance. If the nonce is missing or old,
   the AS returns `use_dpop_nonce` and puts a fresh nonce in the
   `DPoP-Nonce` response header; the client then sends the
   request again with a new proof containing that nonce.
4. The key in the DPoP proof uses an algorithm listed in
   `kbia_key_alg_values_supported` ({{metadata}}). If not, the AS
   returns `invalid_dpop_proof`.
5. The End-User signed in recently enough under AS policy, judged
   by the `auth_time` of the subject token. If not, the AS
   returns `interaction_required`, and the client has the
   End-User sign in again before retrying.
6. The client acts only at the End-User's direction
   ({{key-holder}}), and AS policy allows this End-User and
   client to obtain KBIAs. If not, the AS returns
   `unauthorized_client`.

If every check passes, the AS issues a KBIA in the requested
`kbia_format`, with:

- `sub` set to the subject of the subject token;
- `cnf.jwk` set to the public key from the DPoP proof's `jwk`
  header, and no other key;
- the End-User claims for the requested `scope` that the client
  is authorized to receive; and
- `key_uv` only as {{uv}} specifies.

The AS MUST NOT accept a KBIA as a `subject_token`, or as any
other credential, at any endpoint.

## Response

Token Exchange always returns the issued token in
`access_token`, so that member carries the KBIA, which is not an
access token. `token_type` is `N_A`, meaning the KBIA MUST NOT be
used as an access token ({{RFC8693}} Section 2.2.1).
`expires_in` is the KBIA's remaining lifetime in seconds. The AS
issues no refresh token; the client requests a new KBIA before
this one expires.


The response follows {{RFC8693}} Section 2.2:

~~~json
{
  "access_token": "eyJ0eXAiOiJrYmlhK2p3dCIsImFsZyI6IlJTMjU2...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:kbia",
  "token_type": "N_A",
  "expires_in": 7776000
}
~~~


## Errors {#errors}

The AS returns errors in the form {{RFC8693}} Section 2.2.2
specifies and follows erros from {{RFC6749}}, {{RFC8693}}, and
{{RFC9449}}. `interaction_required` is defined by {{OIDC}} for
the authorization endpoint; this specification also uses it at the
token endpoint, to tell the client that the End-User must sign in
again before a KBIA can be issued.

# Key Holder {#key-holder}

A verifier treats every signature by the bound key as a
statement the End-User made, so the key must sign only what the
End-User chose to sign. An application or wallet that signs when
the End-User asks meets this. An AI agent that decides for
itself what to sign does not, because verifiers would take what
the agent decided as the End-User's decision.

An AS MUST issue KBIAs only to clients that act at the End-User's
direction, and MUST NOT issue one to a client that acts for the
End-User on its own. Clients that act at the End-User's
direction include an application on the End-User's phone or
computer, and a wallet or remote signing service that only the
End-User's approval unlocks, for example with a passkey. The AS
tells which kind a client is from its registration or from a
client attestation {{ABCA}}. When an agent runs in the same
application or on the same device, the bound key MUST be kept
out of the agent's reach, for example in a platform
authenticator that signs only after user verification ({{uv}}).

This does not limit what agents can do. An agent acts under its
own identity, such as a workload credential {{WIMSE-CREDS}},
with the End-User's authority in a delegation credential that
names the End-User as subject and the agent as actor
{{RFC8693}}, so verifiers see who acted and for whom. KBIAs are
for the statements that must be the End-User's own, such as
approving an action or setting a task's limits. An agent can
relay a KBIA and signatures made with its key, but cannot make
those signatures itself. {{OPR}} takes a different approach, in
which a claim records after the fact whether a human or an agent
operated a presentation. Issuing KBIAs only to clients that act
at the End-User's direction instead prevents an agent from
making the End-User's signature at all.

# User Verification {#uv}

A valid signature proves only that the bound key signed, not
that the End-User was there when it did. The End-User's sign-in
(`auth_time`) happened before the KBIA was issued, possibly
months earlier. When a signature stands for the End-User's
decision, such as an approval ({{uses}}), the verifier needs to
know that the End-User was verified when the key signed.

Only the platform that holds the key can ensure this. Platform
authenticators, hardware-backed keys, and passkey-unlocked
wallets can create a key that signs only after the End-User's
PIN or biometric, so that neither the client nor an agent can
sign without the End-User. Keys that exist only while user,
platform, and policy conditions hold {{CONDITION-KEYS}} are
another way. At issuance, the AS checks a key attestation from
the platform and includes `key_uv` with the value `true` only
when the attestation shows that the key has this policy. It MUST
NOT include `key_uv` otherwise.

Editor's note: How the client conveys a key attestation in the
token request is to be defined, for example alongside the client
attestation of {{ABCA}}.

`key_uv` describes how the key was created, not each signature,
and whether later changes, such as a new biometric enrollment,
invalidate the key depends on the platform. To show that a 
signature is fresh, the key signs a value the verifier chose or 
one bound to the request being approved.

TODO: Passkeys {{WEBAUTHN}} show user verification on every
signature but cannot sign a DPoP proof or arbitrary content.
Define how a KBIA can bind one.

# Revocation {#revocation}

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
of the presenter.

Resource servers and relying parties MUST reject a JWT whose
`typ` is `kbia+jwt` or `kbia+sd-jwt` as an access token, ID
Token, or other authentication or authorization credential. A
verifier MUST NOT accept as the End-User's key a key that appears
only in a credential whose subject or actor is an agent
({{key-holder}}). Applications establish that a peer holds the
bound key by verifying signatures made with it, for example a
signature over a verifier-chosen challenge, over application
messages, or over a credential such as an MLS LeafNode.

A verifier that treats a signature by the bound key as the
End-User's decision, such as an approval, SHOULD require
`key_uv`, and checks the signature's freshness as the
application profile specifies ({{uv}}).

Verifiers MAY identify the bound key by its JWK thumbprint
{{RFC7638}}, for example for display and out-of-band comparison.

# Key Endorsements {#endorsements}

A KBIA binds one key. An End-User often needs more, such as one key
per device, keys that rotate without a new KBIA, keys that others
can encrypt to while the End-User is offline, and keys that are
valid only for one session, dialog, or counterpart, so that a
leaked key is of little use. A key endorsement provides these,
in the way that an MLS KeyPackage {{RFC9420}} provides keys that
others use under a member's credential.

A key endorsement is a JWT, signed with a KBIA's bound key, that
declares a further key, the endorsed key, to be the End-User's
for a stated use, context, and period. The AS is not involved.

Only the client that holds the bound key of a valid KBIA can
endorse a key. It MUST endorse only its own keys or keys held by
another client that acts at the End-User's direction
({{key-holder}}), such as the End-User's second device, and MUST
NOT endorse a key held by an agent. A verifier MUST accept
only endorsements signed with the key in the KBIA's `cnf`
({{verifying}}).

## Endorsement Format {#endorsement-format}

The JOSE header MUST contain `typ` with the value
`kbia-endorsement+jwt` and `alg` identifying an asymmetric
algorithm. The payload contains:

`cnf`:
: REQUIRED. The endorsed key, as the `jwk` member of a
  confirmation claim {{RFC7800}}, in the same form as the bound
  key in a KBIA's `cnf`.

`kbia_jkt`:
: REQUIRED. The JWK SHA-256 Thumbprint {{RFC7638}} of the bound
  key that signs the endorsement, which links the endorsement to
  its KBIA.

`key_use`:
: REQUIRED. A JSON array of values from the KBIA Key Uses
  registry ({{iana-key-uses}}) stating what the endorsed key may
  be used for. The values defined here are `mls`, for an MLS
  LeafNode signature key,`presentation`, to prove possession to
  a verifier and `encryption`, for a key that others encrypt to.
  No value permits approvals, which are always signed with the
  bound key ({{endorse-proves}}).

`iat`, `exp`:
: REQUIRED. `exp` MUST NOT be later than the KBIA's `exp`.

`jti`:
: REQUIRED. A unique identifier for the endorsement.

`key_ctx`:
: OPTIONAL. A JSON object that limits the endorsed key to one
  context, with members `type`, a string naming the kind of
  context, and `id`, its identifier. Contexts include a session,
  an MLS group, an agent dialog, and a transaction. The
  application profile defines the values of `type` and how a
  verifier learns the context in which a key is used. `id` MUST
  NOT identify the End-User.

`aud`:
: OPTIONAL. The verifiers that may accept the endorsed key.

`one_time`:
: OPTIONAL. `true` if the endorsed key may be used once, as for
  an encryption key that one sender uses, in the way MLS
  KeyPackages are used once.

For example, an endorsement of a key that the End-User's phone
uses only in one dialog, for a few hours:

~~~json
{
  "kbia_jkt": "NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs",
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
      "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0"
    }
  },
  "key_use": [ "mls", "presentation" ],
  "key_ctx": { "type": "dialog", "id": "d-17" },
  "iat": 1762041600,
  "exp": 1762056000,
  "jti": "end-91b2"
}
~~~

## Creating an Endorsement {#endorsing}

The client that holds the bound key:

1. Generates the key to be endorsed, or obtains it from another
   of the End-User's clients after authenticating that client.
2. Chooses the narrowest scope the use allows: `key_use`;
   `key_ctx`, if the key is for one session, group, or dialog;
   `aud`, if it is for particular verifiers; `one_time`, if it is
   for a single use; and an `exp` that is as early as the use
   allows and not later than the KBIA's `exp`.
3. Builds the endorsement as {{endorsement-format}} specifies,
   with `kbia_jkt` set to the thumbprint of its bound key and
   `cnf` set to the endorsed key.
4. Signs the endorsement with the bound key.
5. Gives the endorsement and the KBIA to the endorsed key's
   holder, or publishes them, for example in a directory where
   senders find `encryption` keys, each `one_time` endorsement
   being handed out once.

The endorsed key's holder sends the KBIA and the endorsement with
each use of the key, as the application profile specifies,
for example in an MLS credential or with a
signature over a verifier's challenge. `aud` and `key_ctx` limit
which verifiers and context accept the key.

## Verifying {#verifying}

A verifier that receives an endorsed key with its endorsement
and the KBIA:

1. validates the KBIA ({{verification}});
2. checks the endorsement's `typ`, verifies its signature with
   the KBIA's `cnf` key, and checks that `kbia_jkt` is the
   thumbprint of that key;
3. checks `iat` and `exp`, and that `exp` is not later than the
   KBIA's `exp`;
4. checks that `key_use` contains the use being made of the key;
5. if `key_ctx` is present, checks that its `type` and `id` match
   the context in which the key is being used, and rejects the key
   in any other; and if `aud` is present, checks that it includes the
   verifier; and
6. if `one_time` is present, rejects a second use, which for an
   `encryption` key means that a sender MUST NOT encrypt to it
   twice.

The verifier then relies on the endorsed key as it would on the
bound key, within that use and scope, and only for signatures
made with it over something tied to the current use, such as an
MLS LeafNode or the verifier's challenge. Revoking the KBIA revokes
every endorsement under it; a single endorsement ends at its
`exp`.

## What an Endorsement Proves {#endorse-proves}

A verifier of an endorsement learns two things: the AS bound the
bound key to the End-User, and the holder of the bound key
declared the endorsed key to be the End-User's for the stated
use, context, and period. It does not learn who holds the
endorsed key. Only the bound key's client is vetted, by the AS
at issuance ({{key-holder}}). For an endorsed key, the verifier
relies on the endorsing client to have endorsed only a key held
at the End-User's direction.

An agent that obtains an endorsed key can act as the End-User
within the endorsement's scope without verifiers noticing. That
scope, set by `key_use`, `key_ctx`, `aud`, `exp`, and `one_time`,
limits the harm, and revoking the KBIA ends every endorsement
under it. For this reason, approvals and other statements of the
End-User's decision are signed only with the bound key, whose
holder and user verification the AS checked at issuance ({{uv}}).

TODO: Let an endorsement carry a key attestation for the endorsed
key, so that a verifier can check where the key is held without
trusting the endorsing client.

## Endorsement Is Not Delegation

An endorsement and a delegation credential both sign over another
key with a scope and an expiry, but they say different things. An
endorsement says the key is the End-User's, so what it signs is
the End-User's own statement. A delegation credential says the
key belongs to another party, such as an agent, that acts for the
End-User, so verifiers see both who acted and for whom
{{RFC8693}}.

An agent therefore receives authority, even for one session,
through a delegation credential that names its own key, never
through an endorsement of a key for it to hold. A verifier MUST
NOT accept an endorsement (`typ` `kbia-endorsement+jwt`) as a
delegation credential, or a delegation credential as an
endorsement.

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
  KBIAs. A key stays in the set until every KBIA it signed has
  expired.

A client determines support by the presence of
`kbia_signing_alg_values_supported` and
`urn:ietf:params:oauth:grant-type:token-exchange` in
`grant_types_supported`.

# Relationship to SD-JWT VC {#sd-jwt-vc}

An SD-JWT VC {{SD-JWT-VC}} with holder key binding, issued
through OpenID for Verifiable Credential Issuance {{OID4VCI}},
can make the same statement as a KBIA, with selective disclosure
and status lists, and is the main alternative to this document.
Using it requires the IdP to act as a credential issuer and the
client to act as a wallet, with an issuance protocol, credential
type metadata, and a trust framework, which enterprise IdPs and
communication clients generally do not otherwise need.

A KBIA is a holder-bound credential with a narrow profile. The
End-User's own identity provider is the issuer, it carries one
identity and one key, and it is issued at the token endpoint
with no wallet or presentation protocol. It reuses SD-JWT for
selective disclosure ({{sd-form}}) and Token Status List for
revocation. The same assertion could be expressed as an SD-JWT
VC type issued at the token endpoint, with the issuance flow and
verification rules of this document unchanged.

# Security Considerations

A KBIA is only as trustworthy as its issuer. An AS that is
compromised or misbehaves can bind any key to any of its users,
and verifiers cannot detect this from the KBIA. Key transparency
{{KEYTRANS}} can let users detect keys bound to them that they
did not create.

Whoever holds the bound key can sign as the End-User until the
KBIA expires or is revoked ({{revocation}}). Clients SHOULD keep
the bound key in hardware that does not allow it to be exported.
Verifiers that need timely revocation check the status list
rather than relying on cached status.

An attacker that steals a subject token could otherwise bind its
own key to the End-User. The AS limits this by requiring that the
subject token was issued to the requesting client, that the
End-User authenticated recently, and that the DPoP proof carries
a fresh AS-provided nonce ({{issuance}}).

A KBIA signing key stays published for as long as the KBIAs it
signed are valid, which is longer than for most token signing
keys. The AS SHOULD use dedicated keys for KBIAs, so that keeping
them published does not prolong trust in its other tokens, and
SHOULD destroy each private key once it stops signing. If a
signing key is compromised, the AS removes it from its key set,
which invalidates every KBIA it signed.

A KBIA is public, so presenting one proves nothing about the
presenter ({{verification}}). Only a signature by the bound key
does, and a signature can be replayed unless it covers a value
the verifier chose or one bound to the request ({{uv}}).
Verifiers MUST also reject a KBIA used as an access token or ID
Token, which its `typ` identifies.

A KBIA attributes every signature of the bound key to the
End-User, so a bound or endorsed key held by an agent lets the
agent speak as the End-User. The AS checks who holds the bound
key ({{key-holder}}), but no party checks who holds an endorsed
key ({{endorse-proves}}). Approvals are therefore signed only
with the bound key.

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

IANA is also requested to register the following claims in the
same registry, each with Change Controller "IETF" and
Specification Document(s) "{{endorsements}} of this document":

- `kbia_jkt`: Thumbprint of the KBIA bound key
- `key_use`: Permitted uses of an endorsed key
- `key_ctx`: Context of an endorsed key
- `one_time`: Single-use key

## KBIA Key Uses {#iana-key-uses}

IANA is requested to create the "KBIA Key Uses" registry, with
the Specification Required policy {{!RFC8126}}. Each entry has a
value, a description, and a reference. The initial entries,
each referencing {{endorsements}} of this document, are `mls`,
`presentation`, and `encryption`.

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

## OAuth Extensions Error Registration

IANA is requested to register the following error in the "OAuth
Extensions Error" registry:

Error Name:
: interaction_required

Error Usage Location:
: token endpoint response

Related Protocol Extension:
: Key-Bound Identity Assertions

Change Controller:
: IETF

Specification Document:
: {{errors}} of this document

## Media Types

IANA is requested to register the media types
`application/kbia+jwt`, `application/kbia+sd-jwt`, and
`application/kbia-endorsement+jwt` in the
"Media Types" registry, for use in the `typ` header as
described in {{kbia-format}} and {{endorsements}}. Full registration templates will
be provided in a future revision.

--- back

# Comparison with Existing Mechanisms {#gaps}

This section describes existing mechanisms that bind, or could
be used to bind, a user's identity to a user-held key, and
where each falls short of an IdP-issued binding that can be
safely published and verified by any peer. {{sd-jwt-vc}}
discusses SD-JWT VC, the closest alternative.

## OpenID Connect ID Tokens

An ID Token {{OIDC}} asserts that the End-User authenticated to
a specific client (its `aud`). It names no user-held key. It is
a bearer artifact, so a party that obtains it can present it to
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

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
