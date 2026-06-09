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
<reference anchor="OIDC2" target="https://doi.org/10.1109/OJCOMS.2024.3376193">
  <front>
    <title>OIDC²: Open Identity Certification With OpenID Connect</title>
    <author fullname="Jonas Primbs" initials="J." surname="Primbs"/>
    <author fullname="Michael Menth" initials="M." surname="Menth"/>
    <date year="2024" month="March" day="11"/>
  </front>
</reference>

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

This specification defines how to bind a public key to an OpenID Connect ID Token using mechanisms defined in [@!RFC9449], OAuth 2.0 Demonstrating Proof of Possession (DPoP).

{mainmatter}

# Introduction

OpenID Connect (OIDC) enables a Relying Party (RP) to obtain End User authentication and identity claims from an OpenID Provider (OP) in the form of an ID Token.
When authenticating with OIDC, an RP initiates the protocol by sending an authentication request to the OP that contains a nonce.
In response, the OP authenticates the End User's identity and sends the RP an ID Token, signed by the OP, containing claims about the user and the requested nonce.

In applications composed of multiple RP components, e.g., a frontend and a backend component, the RP authenticating component (e.g., the frontend) requests and obtains the ID Token from the OP and may present it as a bearer token to one or more RP consuming components (e.g., multiple backends) for authentication as the End User.
However, bearer tokens are vulnerable to theft and replay attacks, allowing attackers to impersonate End Users in such RP component-to-component communication use cases.

Other use cases include, but are not limited to, messaging or conferencing applications.
In these use cases, the RP authenticating component (e.g., an email client sending a mail) and the RP consuming component (e.g., an email client receiving the mail) are not necessarily the same software.
However, in the context of email communication, they MAY act as instances of the same Relying Party (e.g., "email client").
To prevent the End User of the RP consuming component from replaying the ID Token to impersonate the End User of the RP authenticating component (e.g., an attacker who received a mail with the sender's ID Token and replays it to another End User), the ID Token must be constrained to its sender.

This specification defines a mechanism to bind a cryptographic key to the ID Token.
The RP authenticating component can use this key to prove to RP consuming components that not only a user has been authenticated, but that the RP authenticating component itself was the original recipient of that authentication.
This provides stronger security guarantees, preventing token theft and replay attacks, by transforming the ID Token from a bearer token into a proof-of-possession token.

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

## OpenID Connect Metadata

The OP's OpenID Connect Metadata Document [@!OpenID.Discovery] SHOULD include:

- the `bound_key` scope in the `scopes_supported`
- the `dpop_signing_alg_values_supported` property containing a list of supported algorithms as defined in [@?IANA.JOSE.ALGS]

## Protocol Profile Overview

This specification works by adding parameters and headers to the Authentication Request and Token Request and then validating these fields such that the ID Token returned in the Token Response contains a `cnf` claim for a public key.
The RP signals to the OP it is requesting a key-bound ID Token by including the scope `bound_key` in the Authentication Request.

This specification extends OpenID Connect with the addition of a parameter, `dpop_jkt`, to the Authentication Request, and the addition of a `DPoP` header to the Token Request and Refresh Request.
If the OP chooses to issue a key-bound ID Token it validates the `dpop_jkt` parameter and `DPoP` header and returns an ID Token in the Token Response which includes a `cnf` claim for the public key.
This specification does not add new messages, requests or responses.
It preserves the current OpenID Connect flows and interactions.

For the Authorization Code Flow the following changes are made:

1. adding the `bound_key` scope and `dpop_jkt` parameter to the OpenID Connect Authentication Request
2. receiving the authorization `code` as usual in the Authentication Response
3. adding the `DPoP` header that includes the SHA-256 hash of the `code` as the claim `c_s256` in the Token Request to the OP `token_endpoint`
4. adding the `cnf` claim containing the public key to the returned ID Token

```
+------+                              +------+
|      |-- Authentication Request --->|      |
|  RP  |   (1) bound_key & dpop_jkt   |  OP  | 
|      |                              |      | 
|      |<-- Authentication Response --|      |
|      |   (2) authorization code     |      | 
|      |                              |      | 
|      |-- Token Request ------------>|      |
|      |   (3) DPoP header w/ c_s256  |      |
|      |                              |      |
|      |<-- Token Response -----------|      |
|      |   (4) cnf claim containing   |      |
|      |   the public key in ID Token |      | 
+------+                              +------+
```

The Device Authorization Flow follows the pattern of the Authorization Code Flow but sets the claim `c_s256` to the SHA-256 of the `device_code` in place of the authorization `code`, making the following changes:

1. adding the `bound_key` scope and `dpop_jkt` parameter to the OpenID Connect Authentication Request
2. receiving the `device_code` as usual in the Device Authentication Response
3. adding the `DPoP` header that includes the SHA-256 hash of the `device_code`, `c_s256`, as a claim in the Token Request to the OP `token_endpoint`
4. adding the `cnf` claim containing the public key to the returned ID Token

```
+----------+                                +------+
|          |-- Authentication Request ----->|      |
|    RP    |   (1) bound_key & dpop_jkt     |  OP  |
| (device  |                                |      |
| client)  |<-- Authentication Response ----|      |
|          |   (2) device_code, user_code   |      |
|          |       & Verification URI       |      |
|          |                                |      |
|          |   [polling]                    |      |
|          |-- Token Request -------------->|      |
|          |   (3) DPoP header w/ c_s256    |      |
|          |   c_s256 = SHA256(device_code) |      |
|          |                                |      |
|          |<-- Token Response -------------|      |
|          |   (4) cnf claim containing     |      |
|          |   the public key in ID Token   |      |
+----------+                                +------+
```

# Authorization Code Flow

## Authentication Request

If the RP authenticating component is running on a device that supports a web browser, it makes an authorization request per [@!OpenID.Core] 3.1. In addition to the `scope` parameter containing `openid`, and the `response_type` having the value `code`, the `scope` parameter MUST also include `bound_key`, and the request MUST include the `dpop_jkt` parameter having the value of the JWK Thumbprint [@!RFC7638] of the proof-of-possession public key using the SHA-256 hash function, as defined in [@!RFC9449] section 10.

Following is a non-normative example of an authentication request using the authorization code flow:

```text
GET /authorize?
response_type=code
&dpop_jkt=dnfb1T9jil_gOhti60baHs_WD_a4D8JN9VDJXbmBmGw
&scope=openid%20profile%20email%20bound_key
&client_id=s6BhdRkqt3
&state=af0ifjsldkj
&nonce=2a50f9ea812f9bb4c8f7
&redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb HTTP/1.1
Host: server.example.com
```

If the OP does not support the `bound_key` scope, it SHOULD ignore it per [@!OpenID.Core] 3.1.2.1.

## Authentication Response

If the key provided was not previously bound to the client, the OP SHOULD inform the user and obtain consent that a key binding will be done.

On successful authentication of, and consent from the user, the OP returns an authorization `code`.

Following is a non-normative example of a response:

```text
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
    code=SplxlOBeZQQYbYS6WxSbIA
    &state=af0ifjsldkj
```

## Token Request

To obtain the ID Token, the RP authenticating component:

1. generates `c_s256` by computing SHA256 hash of the authorization `code` encoded as `BASE64URL(SHA256(ASCII(code)))`
2. generates a `DPoP` header, including the `c_s256` claim in the `DPoP` header JWT. This binds the authorization `code` to the token request.

Non-normative example of a confidential client setting `Authorization: Basic` per [@!OpenID.Core] 3.1.3.1:

```text
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
DPoP: eyJhbGciOiJFUzI1NiIsImp3ayI6eyJjcnYiOiJQLTI1NiIsImt0eSI6\
 IkVDIiwieCI6InVrcHYzZlU2dHFRS2FVd2NkQkFRb0szSUh2SklXX185eU5kMW\
 9SN3F2WmMiLCJ5IjoibkJCeFhyeDBOeml3Z19ldmZVTVVVZ25HS0tVZjJBVHBX\
 RzlFb2puVW9VNCJ9LCJ0eXAiOiJkcG9wK2p3dCJ9.eyJjX3MyNTYiOiJvMXVCc\
 DllU2UzRHNtU2NOMGpZcmlGZ0tLRmRLLUJMeXdDOVdScFY1R0c4IiwiaHRtIjo\
 iUE9TVCIsImh0dSI6Imh0dHBzOi8vc2VydmVyLmV4YW1wbGUuY29tL3Rva2VuI\
 iwiaWF0IjoxNzYxOTM3NDQ5LCJqdGkiOiJJUVM1dFlQLWJwQlB0SnNvclQ0ejd\
 nIn0.ay7H-sV7o_NE19Qfdq7oFNZ_oH-8LRw7_dgiTRQAUusLjEhgzNYR1ZU1T\
 6IZGopiTEk55LPu_g0gKKku96d4kA

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
```

`Authorization: Basic` HTTP header is only included if a confidential client is used.

If a DPoP header is included in the token request to the OP, and the `dpop_jkt` parameter was not included in the authentication request, the OP MUST NOT include the `cnf` claim in the ID Token.

> This prevents an existing deployment using DPoP for access token from having key-bound ID Tokens issued accidentally.

The OP MUST:

- perform all verification steps as described in [@!RFC9449] section 5.
- calculate the `c_s256` from the authorization `code` just as the RP component did.
- confirm the `c_s256` in the DPoP JWT matches its calculated `c_s256`

# Device Authorization Flow

## Authentication Request

If the RP authenticating component is running on a device that does not support a web browser, it makes an authorization request per [@!RFC8628] 3.1. In the request, the `scope` parameter MUST contain both `openid` and `bound_key`. The request MUST include the `dpop_jkt` parameter having the value of the JWK Thumbprint [@!RFC7638] of the proof-of-possession public key using the SHA-256 hash function, as defined in [@!RFC9449] section 10.

Following is a non-normative example of an authentication request using the device authorization flow:

```text
POST /device_authorization HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
dpop_jkt=dnfb1T9jil_gOhti60baHs_WD_a4D8JN9VDJXbmBmGw
&scope=openid%20profile%20email%20bound_key
&client_id=s6BhdRkqt3
&nonce=KDOmGsiiMaiq-ZhBE-RmPgCsrH-bs-wqbqD2FsRWf7g
```

If the OP does not support the `bound_key` scope, it SHOULD ignore it per [@!OpenID.Core] 3.1.2.1.

## Authentication Response

As per [@!RFC8628], the OP in response to the Authentication Request, generates and returns to the RP authenticating component the required parameters `device_code`, `user_code`, `verification_uri` and `expires_in` and may return the optional parameters `verification_uri_complete` and `interval`.

Following is a non-normative example of an authentication response using the device authorization flow:

```json
{
"device_code":"GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
"user_code":"059-461-148",
"verification_uri":"https://client.example.org/device",
"verification_uri_complete":"https://client.example.org/?user_code=059-461-148",
"expires_in": 1800
}
```

## Token Request

As per [@!RFC8628] the RP authenticating component makes token requests to OP at regular intervals.
Prior to the OP authenticating and obtaining consent from the user, the OP returns an error.
Once the OP has authenticated and obtained consent from the user, the OP responds by returning the ID Token.

In addition to the parameters required by [@!RFC8628] the token request to the OP must contain a DPoP header.
The RP authenticating component computes this DPoP header as follows:

1. generates `c_s256` by computing SHA-256 hash of the authorization `device_code` encoded as `BASE64URL(SHA256(ASCII(device_code)))`
2. generates a `DPoP` header, including the `c_s256` claim in the `DPoP` header JWT. This binds the authorization `device_code` to the token request.

Non-normative example of a token request:

```text
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
DPoP: eyJhbGciOiJFUzI1NiIsImp3ayI6eyJjcnYiOiJQLTI1NiIsImt0eSI6\
 IkVDIiwieCI6InVrcHYzZlU2dHFRS2FVd2NkQkFRb0szSUh2SklXX185eU5kMW\
 9SN3F2WmMiLCJ5IjoibkJCeFhyeDBOeml3Z19ldmZVTVVVZ25HS0tVZjJBVHBX\
 RzlFb2puVW9VNCJ9LCJ0eXAiOiJkcG9wK2p3dCJ9.eyJjX3MyNTYiOiJ6LTZLS\
 k1GNjcxUFFLWFN1SUhBVlFmbkVWUjJ4MUFVc2ZIbHZDNTB2YTM4IiwiaHRtIjo\
 iUE9TVCIsImh0dSI6Imh0dHBzOi8vc2VydmVyLmV4YW1wbGUuY29tL3Rva2VuI\
 iwiaWF0IjoxNzYxOTM3NDQ5LCJqdGkiOiJJUVM1dFlQLWJwQlB0SnNvclQ0ejd\
 nIn0.9t65IuqqvabsJp4v9CpY_pj7ad97KCdR9LXXF-pFvUokP_h2OZ2KqlM10\
 O-l-vebFVHk0qbm1pcw3MWH_VhO7A

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adevice_code
&device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
&client_id=app_fzr7iWr50CWQkGDrLCZBYQc4_2Ak
```

If a DPoP header is included in the token request to the OP, and the `dpop_jkt` parameter was not included in the authentication request, the OP MUST NOT include the `cnf` claim in the ID Token.

> This prevents an existing deployment using DPoP for access token from having key-bound ID Tokens issued accidentally.

The OP MUST:

- perform all verification steps as described in [@!RFC9449] section 5.
- calculate the `c_s256` from the authorization `device_code` just as the RP component did.
- confirm the `c_s256` in the DPoP JWT matches its calculated `c_s256`

# Token Response

If the token request was successful, the OP MUST return an ID Token containing the `cnf` claim as defined in [@!RFC7800] set to the jwk of the user's public key and with  `typ` set to `dpop+id_token` in the ID Token's protected header.

Non-normative example of the ID Token payload:

```json
{
    "iss": "https://server.example.com",
    "sub": "24400320",
    "aud": "s6BhdRkqt3",
    "nonce": "n-0S6_WzA2Mj",
    "exp": 1311281970,
    "iat": 1311280970,
    "cnf":
        {
            "jwk": {
                "crv": "P-256",
                "kty": "EC",
                "x": "ukpv3fU6tqQKaUwcdBAQoK3IHvJIW__9yNd1oR7qvZc",
                "y": "nBBxXrx0Nziwg_evfUMUUgnGKKUf2ATpWG9EojnUoU4"
            }
        }
}
```

The OP MAY return a Refresh Token.
If a Refresh Token is returned, it MUST be bound to the public key of the DPoP proof used in the Token Request i.e. the same public key bound to the ID Token.

# Refresh Request

If a Refresh Token was returned in the Token Response, the RP may use the Refresh Token to make Refresh Requests to the OP's Token Endpoint and receive a refreshed ID Token ([@!OpenID.Core] 12).
This Refresh Token MUST be bound to the same public key as the ID Token and the OP MUST validate a DPoP proof ([@!RFC9449] 5) for this public key on each refresh request.

To refresh the ID Token, the RP authenticating component:

1. generates a `DPoP` header
2. makes a POST request to the OP's Token Endpoint with the `DPoP` header and the Refresh Token as a parameter.

Non-normative example:

```text
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJhbGciOiJFUzI1NiIsImp3ayI6eyJjcnYiOiJQLTI1NiIsImt0eSI6\
 IkVDIiwieCI6InVrcHYzZlU2dHFRS2FVd2NkQkFRb0szSUh2SklXX185eU5kMW\
 9SN3F2WmMiLCJ5IjoibkJCeFhyeDBOeml3Z19ldmZVTVVVZ25HS0tVZjJBVHBX\
 RzlFb2puVW9VNCJ9LCJ0eXAiOiJkcG9wK2p3dCJ9.eyJodG0iOiJQT1NUIiwia\
 HR1IjoiaHR0cHM6Ly9zZXJ2ZXIuZXhhbXBsZS5jb20vdG9rZW4iLCJpYXQiOjE\
 3NjE5Mzc4MjMsImp0aSI6ImJHOXpaV1psYm1ObFkyaHZiM05sY20ifQ.NVmGXw\
 opPNYiN7CpITgR0Fl1PYFFgIAbxPxs8N1llDPoQmR60il35b-Zez71eMkdM9gd\
 oqJkee3oKrimdrsCfA

grant_type=refresh_token&refresh_token=8xLOxBtZp8
```

The OP MUST validate the Refresh Token and MUST validate the `DPoP` header presented.
The OP MUST reject the `DPoP` header if it is not signed with the public key that was bound to the presented Refresh Token in the initial Token Request.
Unlike the Token Request, no `c_s256` claim is required in the `DPoP`header for the Refresh Request.

If an ID Token is returned as a result of a Refresh Request, an additional requirement applies:

- its `cnf` claim MUST be the same as in the ID Token issued when the original authentication occurred.

If a new Refresh Token is returned as a result of a Refresh Request, the newly issued Refresh Token MUST continue to be bound to the same public key as the original Refresh Token.

# ID Token Proof of Possession

The mechanism for how an RP authenticating component proves to an RP consuming component that it possesses the private keys associated with the `cnf` claim in the ID Token is out of scope of this document.

# Privacy Considerations

An RP authenticating component SHOULD only share an ID Token with a consuming component when such sharing is consistent with the original purpose for which the identity data was collected and the scope of consent obtained from the user.

If the RP consuming parties are expected to be controlled by third-party End Users (see the peer-to-peer applications in [Appendix A.2](#2.-peer-to-peer-authentication)), the consent screen at the OP MUST make it clear to the End-User that any identity claims relating to the granted scopes will be shared with and accessed by these third-party users.

## Unique ID Tokens

Per session, the RP authenticating component MAY require multiple individual ID Tokens.

* *Example 1*: Alice's RP authenticates in direct Matrix chats with Bob and Charly.
  This requires two individual OIDC key pairs and results in two individual key-bound ID Tokens.
* *Example 2*: Alice wants to be authenticated to Bob with her name and her email address.
  To Charly, Alice only wants to share her name.
  This requires Alice's RP to request key-bound ID Tokens with two distinct scopes, resulting in two different sets of identity claims.

# Security Considerations

## Public Key Substitution Attacks

A public key substitution attack is a type of Unknown Key Share (UKS) attack in which an adversary binds the adversary identity to another party's key.

To protect against such attacks, the `DPoP` header JWT sent in the Token Request MUST include the `c_s256` claim which contains the SHA-256 of the authorization `code`, or in the case of the Device Authorization Flow the SHA-256 of the `device_code`. This prevents replaying of the `DPoP` header JWTs between authentication sessions as each `DPoP` header JWT in a Token Request is now strictly bound to that session.

## Require Proof of Possession

An RP consuming component MUST NOT trust an ID Token with a `cnf` claim without a corresponding proof of possession from the RP authenticating component.

## ID Token Reverification

In addition to verifying the signature created by the RP authenticating component to prove possession of the private key associated with the `cnf` claim in the ID Token, an RP consuming component MUST independently verify the signature and validity of the ID Token, that the `aud` claim in the payload is the correct value, and that the `typ` claim in the protected header is `dpop+id_token`.

## Use as Access Token

The ID Token MUST NOT be used as an access token to access resources. The RP MAY exchange the ID Token with a proof of possession for an access token that can then be used to access resources.

## Unique Key Pair

To prevent token confusion attacks, the RP authenticating component SHOULD bind a unique key pair to its ID Tokens, and not use it for other purposes.

## Using cnf as a User Claim

The `cnf` claim in the ID Token MUST be verified together with a proof of possession and MUST NOT be treated as proof on its own. A proof of possession is REQUIRED to establish that a party controls the key identified by `cnf`. The `cnf` claim SHOULD only be used to bind a signed object with the other claims in the ID Token.

## Audience Claim

Key-bound ID Tokens may be shared by RP authenticating components with RP consuming components that identify themselves to different audiences.
*Example 1 (frontend/backend)*: RP authenticating component = `bitwarden-android`, RP consuming component = `vaultwarden-backend`.
*Example 2 (different platforms)*: RP authenticating component = `myapp-android`, RP consuming component = `myapp-ios`.

However, the audience claim `aud` MUST be strictly verified.
As it is recommended to use different client IDs for different platforms, two options remain:

1. Use a dedicated audience for end-to-end use cases, e.g., `bitwarden-protocol`, or
1. Accept a static list of expected audiences, e.g., `myapp-android`, and `myapp-ios`.

## Validity Period

Validity periods of key-bound ID Tokens depend on use cases.
In real-time communication systems (e.g., video conferencing), ID Tokens can be extremely short-lived (< 1 minute) because they are only needed to initially authenticate the P2P channel.
However, the time drift between the RP authenticating component and the RP consuming component has to be taken into account.
In near-real-time communication systems (e.g., instant messaging with matrix), short validity periods (< 10 minutes) are sufficient because identity verification processes can be repeated if the RP consuming component is offline, and ID Tokens must be valid when the RP consuming component receives the token.
In asynchronous communication systems (e.g., email), longer validity periods are preferable (> 1 day) because ID Tokens must be valid on the first message (email) lookup.

## Group Communication

When the RP authenticating component is authenticated as the End User in group communication (e.g., video conferences or group chats with more than 2 participants), using a single key-bound ID Token is preferred over multiple individual key-bound ID Tokens.
This improves performance because the RP authenticating component must create one instead of n (= number of participants) ephemeral key pairs, and the OP must issue one instead of n ID Tokens.

# IANA Considerations

The following entry should be added to the "Media Types" registry for the new JWT type:

Type name: application

Subtype name: dpop+id_token

{backmatter}

# Appendix A. Use Cases

## 1. Zero-Trust in distributed Relying Parties

It is common that applications consist of multiple components (typically a frontend and a backend).
In these applications, the user-facing component (the frontend) typically initiates the authentication flow by providing a nonce and authenticates the End User with the ID Token issued by the OP, which contains identity claims and the requested nonce.
RP consuming components (the backends) authenticate the RP authenticating component as the End User using the ID Token included as a bearer token in the request.

If communication between these RP components is compromised, attackers can steal the ID Token and use it to send malicious authenticated requests to the RP consuming components.
Binding the ID Token to the RP authenticating component's key pair mitigates these attacks because attackers cannot use a key-bound ID Token without providing valid proof of possession of the corresponding private key, which only the RP authenticating component knows.

## 2. Peer-to-Peer authentication

Assuming Bob wants to authenticate Alice using her OIDC identity in a peer-to-peer application, e.g., a WebRTC-based video conference.
With classic OIDC, Mallory could replay a nonce provided by Bob to Alice, who would initiate an OIDC authentication flow with her RP authenticating component.
Alice would then respond to Mallory with her ID Token issued by the OP, containing her identity claims and the nonce replayed by Mallory.
Mallory can then forward the ID Token along with a malicious message to Bob.
Bob would then authenticate Mallory with Alice's OIDC identity and believe that this message was written by Alice.

An ID Token bound to a key pair for which the OP has verified Alice's possession of the corresponding private key enables Alice to sign her message to Bob using that key.
This securely authenticates Alice to Bob and allows Bob to detect messages that were not sent by Alice.
Similar applications for secure email communication, Matrix-based instant messaging, and WebRTC-based video conferences have been discussed in the OIDC² proposal [@!OIDC2].

# Appendix B. Acknowledgements

The authors would like to thank early feedback provided by Filip Skokan, Frederik Krogsdal Jacobsen, George Fletcher, Jacob Ideskog, Jonas Primbs, Karl McGuinness, and Kosuke Koiwai.

# Appendix C. Notices

Copyright (c) 2026 The OpenID Foundation.

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
