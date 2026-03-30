%%%
title = "OpenID Connect Key Binding 1.0 - draft 00"
abbrev = "openid-connect-key-binding"
ipr = "none"
workgroup = "OpenID Connect"
keyword = ["security", "openid", "lifecycle"]

[seriesInfo]
name = "Internet-Draft"
value = "openid-key-binding-1_0"
status = "standard"

[[author]]
initials="D."
surname="Hardt"
fullname="Dick Hardt"
organization="Hellō"
    [author.address]
    email = "dick.hardt@gmail.com"

[[author]]  
initials="E."
surname="Heilman"
fullname="Ethan Heilman"
organization="Cloudflare"
    [author.address]
    email = "ethan.r.heilman@gmail.com"

%%%
<reference anchor="OpenID.Core" target="https://openid.net/specs/openid-connect-core-1_0.html">
  <front>
    <title>OpenID Connect Core 1.0 (incorporating errata set 2)</title>
    <author fullname="Nat Sakimura" initials="N." surname="Sakimura"/>
    <author fullname="Michael B. Jones" initials="M." surname="Jones"/>
    <author fullname="John Bradley" initials="J." surname="Bradley"/>
    <date year="2023" month="December" day="15"/>
  </front>
</reference>

<reference anchor="OpenID.Discovery" target="https://openid.net/specs/openid-connect-discovery-1_0.html">
  <front>
    <title>OpenID Connect Discovery 1.0 (incorporating errata set 2)</title>
    <author fullname="Nat Sakimura" initials="N." surname="Sakimura"/>
    <author fullname="Michael B. Jones" initials="M." surname="Jones"/>
    <author fullname="John Bradley" initials="J." surname="Bradley"/>
    <author fullname="Edmund Jay" initials="E." surname="Jay"/>
    <date year="2023" month="December" day="15"/>
  </front>
</reference>

<reference anchor="IANA.JOSE.ALGS" target="https://www.iana.org/assignments/jose/jose.xhtml#web-signature-encryption-algorithms">
  <front>
    <title>IANA JSON Web Signature and Encryption Algorithms Registry</title>
    <author fullname="IANA"/>
    <date year="2025"/>
  </front>
</reference>

.# Abstract

OpenID Key Binding specifies how to bind a public key to an OpenID Connect ID Token using mechanisms defined in [@!RFC9449], OAuth 2.0 Demonstrating Proof of Possession (DPoP).

{mainmatter}

# Introduction

OpenID Connect is a protocol that enables a Relying Party (RP) to delegate authentication and obtain identity claims to an OpenID Connect Provider (OP).

When authenticating with OpenID Connect, an RP provides a nonce in its authentication request. The ID Token signed and returned by the OP contains the nonce and claims about the user. When verifying the ID Token, the RP confirms it contains the nonce, binding the session that made the request to the response.

It is common for an RP to be composed of multiple components such as a RP authenticating component that  obtains the ID Token from the OP and an RP consuming component which checks the ID Token presented to it by the authenticating component. When the RP authenticating component wants to prove to an RP consuming component that it has authenticated a user, it may present the ID Token as a bearer token. However, bearer tokens are vulnerable to theft and replay attacks - if an attacker obtains the ID Token, they can impersonate the authenticated user.

By binding a cryptographic key to the ID Token, the RP authenticating component can prove to RP consuming components not only that a user has been authenticated, but that the RP authenticating component itself was the original recipient of that authentication. This transforms the ID Token from a vulnerable bearer token into a proof-of-possession token that provides stronger security guarantees.

The RP may also prove possession of the bound key when presenting an ID Token back to the OP.

Use cases include: a mobile app that has received an ID Token exchanging the ID Token with a proof of possession with a first party authorization service for an access token; an instance of a peer to peer application such as video conferencing where one instance of the application sends the ID Token with a proof of possession to a second instance to prove which user is operating the first instance.

This specification profiles OpenID Connect 1.0 [@!OpenID.Core], RFC8628 - OAuth 2.0 Device Authorization Grant [@!RFC8628], and RFC9449 - OAuth 2.0 Demonstrating Proof of Possession (DPoP) [@!RFC9449] to enable cryptographically bound ID Tokens that resist theft and replay attacks while maintaining compatibility with existing OpenID Connect infrastructure.


## Requirements Notation and Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [@!RFC2119].

In the .txt version of this specification,
values are quoted to indicate that they are to be taken literally.
When using these values in protocol messages,
the quotes MUST NOT be used as part of the value.
In the HTML version of this specification,
values to be taken literally are indicated by
the use of *this fixed-width font*.

## Terminology

This specification uses the following terms:

- **OP**: The OpenID Provider as defined in [@!OpenID.Core].

- **RP**: The Relying Party as defined in [@!OpenID.Core].

The parameters **dpop_jkt** and **DPoP** as defined in [@!RFC9449]

## Protocol Profile Overview

This specification profiles how to bind a public key to an ID Token.

1. The RP requests a DPoP-bound Access Token with at least the `openid` and `bound_key` scope
2. The RP sends a Token Exchange request adding the DPoP-bound Access Token in the request body and the DPoP proof in the `DPoP` header
3. The OP issues an ID Token adding the `cnf` claim containing the public key from the DPoP-bound Access Token

```
+------+                               +------+
|      |              ...              |      |
|  RP  |   (1) Request DPoP-bound      |  OP  |
|      |       Access Token with scope |      |
|      |       openid & bound_key      |      |
|      |              ...              |      |
|      |                               |      |
|      |-- Token Exchange Request ---->|      |
|      |   (2) Exchange Access Token   |      |
|      |   with DPoP proof             |      |
|      |                               |      |
|      |<-- Token Exchange Response ---|      |
|      |   (3) cnf claim containing    |      |
|      |   the public key in ID Token  |      |
+------+                               +------+
```

## OpenID Connect Metadata

The OP's OpenID Connect Metadata Document [@!OpenID.Discovery] SHOULD include:

- the `bound_key` scope in the `supported_scopes`
- the `dpop_signing_alg_values_supported` property containing a list of supported algorithms as defined in [@?IANA.JOSE.ALGS]

## DPoP-bound ID Token Request

The RP authenticating component exchanges a DPoP-bound Access Token for a DPoP-bound ID Token at the OP's Token Endpoint.

### DPoP-bound Access Token Requirements

To obtain a DPoP-bound ID Token, the RP authenticating component needs a DPoP-bound Access Token with the following scopes granted:

`openid`
  **REQUIRED**.
  Indicates that the RP is authorized to request an ID Tokens at all

`bound_key`
  **REQUIRED**.
  Indicates that the RP is authorized to request a DPoP-bound ID Token

Further scopes related to ID Token claims, e.g., `profile`, `email`, `address`, ...
  **OPTIONAL**.
  Indicate that the RP is authorized to access corresponding identity claims.
  The RP can request a subset of these claims for the DPoP-bound ID Token.

Obtaining such a DPoP-bound Access Token is not in scope of this spec.
Examples for obtaining them is a DPoP-bound Authorization Request (see [Section 5](https://datatracker.ietf.org/doc/html/rfc9449#section-5) of [@!RFC9449]) (Authorization Code Grant & Refresh Token Grant), or [Section 3](https://datatracker.ietf.org/doc/html/draft-parecki-oauth-dpop-device-flow-00#section-3) of [@!draft-parecki-oauth-dpop-device-flow-00] (Device Authorization Grant).

> Using the Token Exchange mechanism to obtain a DPoP-bound ID Token instead of extending the ID Token in any of the existing authorization flows simplifies this spec and preserves downwards compatibility for classic ID Token usage.

### DPoP Proof

To prove that the RP possesses the private key for the public key in the DPoP-bound Access Token's `cnf` claim, the RP MUST prepare a `DPoP` proof as specified in [Section 7](https://datatracker.ietf.org/doc/html/rfc9449#section-7) of [@!RFC9449].
Therefore, the RP MUST compute the `ath` (Access Token Hash) for the DPoP-bound Access Token in the `subject_token` parameter as specified in [Section 4.2](https://datatracker.ietf.org/doc/html/rfc9449#section-4.2) of [@!RFC9449].

Following is a non-normative example of the decoded DPoP proof header and payload:

```json
{
  "typ":"dpop+jwt",
  "alg":"ES256",
  "jwk": {
    "kty": "EC",
    "x": "1CFjuS-kiH-FuZbrAJA9TfRdPM-928rD9nb4LiEFIrs",
    "y": "qQjbtaUYLUlPP--IhWEbFYo5KU2XlCRaHitu1gpzv9Q",
    "crv": "P-256"
  }
}
.
{
  "jti": "e1j3V_bKic8-LAEB",
  "htm": "POST",
  "htu": "https://server.example.com/token",
  "iat": 1562262618,
  "ath": "fUHyO2r2Z3DZ53EsNrWBb0xWXoaNy59IiKCAqksmQEo"
}
```

> The `ath` parameter binds the DPoP Proof to the lifetime of the DPoP-bound Access Token. Note that this DPoP proof might be successfully replayed within the Access Token lifetime.

### Token Exchange Request

The RP requests a Token Exchange as specified in [Section 2.1](https://datatracker.ietf.org/doc/html/rfc8693#name-request) of [@!RFC8693] using the following parameters:

`grant_type`
  **REQUIRED**.
  The value `urn:ietf:params:oauth:grant-type:token-exchange` indicates that a token exchange is being performed.

`requested_token_type`
  **REQUIRED**.
  The value `urn:ietf:params:oauth:token_type:ic_token` indicates that a DPoP-bound Identity Certification Token is being requested.

`subject_token_type`
  **REQUIRED**.
  The value `urn:ietf:params:oauth:token_type:dpop` indicates that a DPoP-bound Access Token is being used in the `subject_token` parameter.

`subject_token`
  **REQUIRED**.
  A DPoP-bound Access Token containing the scope `openid`, `bound_key`, and optional OpenID Connect identity scope related scopes.

`audience`
  **OPTIONAL**.
  The logical name of the target service where the RP intends to use the requested DPoP-bound ID Token.

`scope`
  **OPTIONAL**.
  A list of space-delimited, case-sensitive strings, as defined in [Section 3.3](https://www.rfc-editor.org/rfc/rfc6749#section-3.3) of [@!RFC6749], that allow the RP to specify which identity claims are expected to be present in the requested DPoP-bound ID Token.
  Typically, only OpenID Connect related scopes, such as those defined in [Section 5.4](https://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims) in OpenID Core, are used here.
  This allows the RP to reduce the set of identity claims contained in the DPoP-bound ID Token compared to the identity claims contained in the ID Token.
  The scopes `openid` and `bound_key` MAY not be provided here.
  The OP MUST ignore all scopes that are not present in the provided DPoP-bound Access Token from the `subject_token` parameter.

*TBD: What about `resource`, `actor_token`, `actor_token_type`?*

The RP MUST add the DPoP Proof to the `DPoP` header of the Token Exchange request.

Non-normative example of a confidential client setting `Authorization: Basic` per [Section 3.1.3.1](https://openid.net/specs/openid-connect-core-1_0.html#TokenRequest) of [@!OpenID.Core] foro a Token Exchange request:

```text
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0\
  eSI6IkVDIiwieCI6IjFDRmp1Uy1raUgtRnVaYnJBSkE5VGZSZFBNLTkyOHJE\
  OW5iNExpRUZJcnMiLCJ5IjoicVFqYnRhVVlMVWxQUC0tSWhXRWJGWW81S1Uy\
  WGxDUmFIaXR1MWdwenY5USIsImNydiI6IlAtMjU2In19.eyJqdGkiOiJlMWo\
  zVl9iS2ljOC1MQUVCIiwiaHRtIjoiUE9TVCIsImh0dSI6Imh0dHBzOi8vc2V\
  ydmVyLmV4YW1wbGUuY29tL3Rva2VuIiwiaWF0IjoxNTYyMjYyNjE4LCJhdGg\
  iOiJmVUh5TzJyMlozRFo1M0VzTnJXQmIweFdYb2FOeTU5SWlLQ0Fxa3NtUUV\
  vIn078vI2oMOU4OLdfzgdE7_YfrIVDzNXP8NpBxnOwPA8ym7CbmqzgSQnpxs\
  EHT5C_tNEaZ8z6OSdXhrWArpwg17qA
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token_type:ic_token
&subject_token_type=urn:ietf:params:oauth:token_type:dpop
&subject_token=Kz~8mXK1EalYznwH-LC-1fBAo.4Ljp~zsPE_NeO.gxU
&audience=authenticating-party
&scope=profile%20email
```

### DPoP-bound ID Token Request Validation

The OP MUST perform the following validation steps:

1. The DPoP-bound Access Token from the `subject_token` parameter MUST be valid.
2. The DPoP Proof MUST be valid and its `ath` claim must be equal to the `subject_token`'s hash.
3. The `subject_token`'s public key contained in its `cnf` claim MUST be equal to the DPoP Proof's public key contained in the `jwk` header.
4. If the RP is a confidential client, the OP MUST successfully authenticate the RP.
5. All scopes within the `scope` parameter not present in the `subject_token`'s scope MUST be ignored.
6. Remaining scopes MUST contain the scope `openid` and `bound_key`.

## DPoP-bound ID Token Response

### DPoP-bound ID Token Structure

If the Token Exchange request was successful, the OP MUST return an ID Token containing the `cnf` claim as defined in [@!RFC7800] set to the jwk of the user's public key and with `typ` set to `id_token+cnf` in the ID Token's protected header.

Non-normative example of the ID Token payload:

```json
{
  "iss": "https://server.example.com",
  "sub": "24400320",
  "aud": "s6BhdRkqt3",
  "nonce": "n-0S6_WzA2Mj",
  "exp": 1311281970,
  "iat": 1311280970,
  "cnf": {
    "jwk": {
      "crv": "P-256",
      "kty": "EC",
      "x": "ukpv3fU6tqQKaUwcdBAQoK3IHvJIW__9yNd1oR7qvZc",
      "y": "nBBxXrx0Nziwg_evfUMUUgnGKKUf2ATpWG9EojnUoU4"
    }
  }
}
```

### Token Exchange Response

The OP responds with the following parameters.

`access_token`
  **REQUIRED**. The DPoP-bound ID Token.

`issued_token_type`
  **REQUIRED**.  The value `urn:ietf:params:oauth:token_type:ic_token` indicates that the issued token in the `access_token` parameter is a DPoP-bound Identity Certification Token.

`token_type`
  **REQUIRED**. The value `DPoP` indicates that the issued DPoP-bound ID Token must be used as a DPoP token.

`expires_in`
  **RECOMMENDED**. The validity lifetime, in seconds, of the DPoP-bound ID token issued by the OP.

`scope`
  **OPTIONAL**. The subset of space-delimited scopes from the token exchange request being applied to the DPoP-bound ID Token.

The OP MUST NOT issue a Refresh Token for the DPoP-bound ID Token.

The DPoP-bound ID Token's expiration time (`exp` claim) MUST NOT exceed the exchanged Access Token expiration time.

Following is a non-normative example of a DPoP-bound ID Token response using the token exchange:

```text
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "[TBD: DPoP-bound ID Token]",
  "issued_token_type": "urn:ietf:params:oauth:token_type:ic_token",
  "token_type": "DPoP",
  "expires_in": 3599,
  "scope": "profile email"
}
```

## Token Refresh

The RP refreshes expired DPoP-bound ID Tokens by sending a new Token Exchange request with a valid DPoP-bound Access Token.

If the DPoP-bound Access Token is expired, the RP refreshes it with a DPoP-bound Refresh Token from the Token response as described in [Section 5](https://datatracker.ietf.org/doc/html/rfc9449#section-5) of [@!RFC9449].

## ID Token Proof of Possession

The mechanism for how an RP authenticating component proves to an RP consuming component that it possesses the private keys associated with the `cnf` claim in the ID Token is out of scope of this document.

> If the WG wants to, we can also profile how to use KB to bind a proof of possession to an ID Token for presentation when a proof of possession is not present.

# Privacy Considerations

An RP authenticating component SHOULD only share an ID Token with a consuming component when such sharing is consistent with the original purpose for which the PII was collected and the scope of consent obtained from the user.

# Security Considerations

## Public Key Substitution Attacks

A public key substitution attack is a type of Unknown Key Share (UKS) attack in which an adversary binds the adversary identity to another party's key.

To protect against such attacks, the `DPoP` header JWT sent in the Token Request MUST include the `c_s256` claim which contains the SHA-256 of the authorization `code`. This prevents replaying of the `DPoP` header JWTs between authentication sessions as each `DPoP` header JWT in a Token Request is now strictly bound to the specific authentication `code` for that session.

## Require Proof of Possession

An RP consuming component MUST NOT trust an ID Token with a `cnf` claim without a corresponding proof of possession from the RP authenticating component.

## ID Token Reverification

In addition to verifying the signature created by the RP authenticating component to prove possession of the private key associated with the `cnf` claim in the ID Token, an RP consuming component MUST independently verify the signature and validity of the ID Token and that the `aud` claim in the payload is the correct value, and that the `typ` claim in the protected header is `id_token+cnf`.

## Use as Access Token

The ID Token MUST NOT be used as an access token to access resources. The RP MAY exchange the ID Token with a proof of possession for an access token that can then be used to access resources.

## Unique Key Pair

To prevent token confusion attacks, the RP authenticating component SHOULD bind a unique key pair to its ID Tokens, and not use it for other purposes.

## Using cnf as a User Claim

The `cnf` claim in the ID Token MUST be verified together with proof of possession and MUST NOT be treated as proof on its own. A proof of possession is REQUIRED to establish that a party controls the key identified by `cnf`. The `cnf` claim SHOULD only be used to bind a signed object with the other claims in the ID Token.

# IANA Considerations

The following entry should be added to the "Media Types" registry for the new JWT type:

Type name: application

Subtype name: dpop+id_token

{backmatter}

# Acknowledgements

The authors would like to thank early feedback provided by Filip Skokan, Frederik Krogsdal Jacobsen, George Fletcher, Jacob Ideskog, Karl McGuinness, and Kosuke Koiwai.


# Notices

Copyright (c) 2025 The OpenID Foundation.

The OpenID Foundation (OIDF) grants to any Contributor, developer,
implementer, or other interested party a non-exclusive, royalty free,
worldwide copyright license to reproduce, prepare derivative works from,
distribute, perform and display, this Implementers Draft, Final
Specification, or Final Specification Incorporating Errata Corrections
solely for the purposes of (i) developing specifications,
and (ii) implementing Implementers Drafts, Final Specifications,
and Final Specification Incorporating Errata Corrections based
on such documents, provided that attribution be made to the OIDF as the
source of the material, but that such attribution does not indicate an
endorsement by the OIDF.

The technology described in this specification was made available
from contributions from various sources, including members of the OpenID
Foundation and others. Although the OpenID Foundation has taken steps to
help ensure that the technology is available for distribution, it takes
no position regarding the validity or scope of any intellectual property
or other rights that might be claimed to pertain to the implementation
or use of the technology described in this specification or the extent
to which any license under such rights might or might not be available;
neither does it represent that it has made any independent effort to
identify any such rights. The OpenID Foundation and the contributors to
this specification make no (and hereby expressly disclaim any)
warranties (express, implied, or otherwise), including implied
warranties of merchantability, non-infringement, fitness for a
particular purpose, or title, related to this specification, and the
entire risk as to implementing this specification is assumed by the
implementer. The OpenID Intellectual Property Rights policy
(found at openid.net) requires
contributors to offer a patent promise not to assert certain patent
claims against other contributors and against implementers.
OpenID invites any interested party to bring to its attention any
copyrights, patents, patent applications, or other proprietary rights
that may cover technology that may be required to practice this
specification.

# Document History

   [[ To be removed from the final specification ]]

   -00

   initial draft
