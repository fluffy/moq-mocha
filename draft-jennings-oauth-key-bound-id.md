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
  OPENPUBKEY:
    title: "OpenPubkey: Augmenting OpenID Connect with User held Signing Keys"
    author:
      -
        name: Ethan Heilman
      -
        name: Lucie Mugnier
      -
        name: Athanasios Filippidis
      -
        name: Sharon Goldberg
      -
        name: Sebastien Lipman
      -
        name: Yuval Marcus
      -
        name: Mike Milano
      -
        name: Sidhartha Premkumar
      -
        name: Chad Unrein
    date: 2023
    target: https://eprint.iacr.org/2023/296


--- abstract

OpenID Connect ID Tokens assert that a user authenticated to a
particular client, but they do not bind that assertion to a key
held by the user, and they are bearer tokens that are unsafe to
disclose to third parties. Applications such as end-to-end
encrypted messaging need an assertion from the user's identity
provider that a long-lived public key belongs to an
authenticated user, which can be published to and verified by
arbitrary peers long after issuance. This document defines the
Key-Bound Identity Assertion (KBIA), a JWT issued by an OAuth 2.0
authorization server that binds a user's subject identifier to a
public key whose possession the user has proven. KBIAs are
obtained using OAuth 2.0 Token Exchange with a DPoP proof, are
explicitly typed so that they cannot be confused with access or
ID Tokens, and optionally support selective disclosure.

--- middle

# Introduction

End-to-end encrypted (E2EE) communication systems, such as those
built on Messaging Layer Security (MLS) {{RFC9420}}, need to
answer the question "does this public key belong to
alice@example.com?". Many deployments already have an
authoritative answer to "is this user alice@example.com?": the
organization's OpenID Connect (OIDC) {{OIDC}} identity provider
(IdP). What is missing is a standard way for the IdP to assert
the binding between that identity and a key the user holds.

Existing mechanisms each meet part of this need. ID Tokens
are IdP-signed but bearer, short-lived, and name no user key.
DPoP-bound access tokens are key-bound but are not identity
assertions for third parties. Committing to a key in the OIDC
`nonce`, as OpenPubkey {{OPENPUBKEY}} does, works with
unmodified IdPs but leaves the IdP unaware of the binding.
Certificate authorities and verifiable credential ecosystems
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
  an MLS credential); that is left to application profiles.
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

## DPoP-Bound Access Tokens

DPoP {{RFC9449}} binds an access token to a client-held key
through `cnf.jkt`, with proof of possession at issuance.
However, an access token expresses authorization for a resource
server, not an identity assertion for third parties. It may be
opaque, is audience-restricted to resource servers, and is
short-lived. Sender-constraining prevents reuse without the
key, but access tokens are not designed for publication.

## Nonce Commitment (OpenPubkey) {#existing-openpubkey}

OpenPubkey {{OPENPUBKEY}} sets the OIDC `nonce` to a commitment
to the user's public key, so that the IdP-signed ID Token
indirectly names the key. This works with unmodified IdPs and
is verifiable by third parties.

The IdP is unaware that it is binding a key and performs no
possession check; possession is shown only by the client's
later signatures. The result is an ID Token and inherits its
properties: it is a bearer artifact, discloses all claims,
cannot be revoked, and becomes unverifiable when the IdP
rotates its signing key. ID Tokens obtained by refresh may not
carry the original nonce, so each new binding requires a fresh
authentication.

OpenPubkey optionally replaces the IdP's RSA signature with a
Guillou-Quisquater (GQ) proof of knowledge of that signature.
The resulting token verifies against the IdP's public key but
cannot be presented as an ID Token, which removes the bearer
risk for RSA-signing IdPs. To remain verifiable after IdP key
rotation, OpenPubkey relies on archived JWKS or an additional
cosigner, which adds infrastructure.

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

## SD-JWT Verifiable Credentials

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
`sub`, `cnf`, `iat`, `exp`, `nbf`, `jti`, `auth_time`, and
`status` MUST NOT be selectively disclosable. End-User claims
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
5. Apply any policy on which End-Users and clients may obtain
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
End-User `sub` and observed proof of possession of the bound
key. It does not, by itself, tell the verifier that the party
presenting the KBIA holds the bound key. A verifier MUST NOT
treat possession or presentation of a KBIA as authentication
of the presenter. Applications establish that a peer holds the
bound key by verifying signatures made with it, for example a
signature over a verifier-chosen challenge, over application
messages, or over a credential such as an MLS LeafNode.

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

For example, an E2EE messaging application might publish the
KBIA alongside a chain of statements signed by the bound key
that authorize each of the user's device keys, and use the
device keys in MLS credentials.

# Security Considerations

## Confusion with Other Tokens

KBIAs are designed for wide distribution, so they must never be
accepted as authentication or authorization tokens. This
document mitigates confusion by explicit typing ({{RFC8725}}),
by omitting `aud`, by the `N_A` token type, and by forbidding
the AS from accepting KBIAs as input. Resource servers and
relying parties MUST reject JWTs whose `typ` is `kbia+jwt` or
`kbia+sd-jwt`.

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

# Nonce-Based Key Commitment {#nonce-commitment}
{:numbered="false"}

Where an IdP does not support KBIAs, applications can
approximate one by committing to the user's key in the OIDC
`nonce` parameter, as done by OpenPubkey {{OPENPUBKEY}}: the
client sets `nonce` to a hash of the public key and a random
salt, and publishes the resulting ID Token together with the
salt. A verifier recomputes the hash and checks it against the
token's `nonce`.

This approach works with unmodified IdPs. Its limitations,
and the optional GQ signature variant, are described in
{{existing-openpubkey}}. Applications using it SHOULD at least request the
ID Token with a client ID dedicated to key binding and ensure
that no relying party accepts ID Tokens issued to that client
ID for authentication, since the published token is otherwise a
replayable bearer token.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
