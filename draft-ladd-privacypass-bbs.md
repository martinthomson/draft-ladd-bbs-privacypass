---
title: "BBS for PrivacyPass"
abbrev: "BBS PPAS"
category: info

docname: draft-ladd-privacypass-bbs-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Privacy Pass"
keyword:
 - anonymous credentials
venue:
  group: "Privacy Pass"
  type: "Working Group"
  mail: "privacy-pass@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/privacy-pass/"
  github: "wbl/draft-ladd-bbs-privacypass"
  latest: "https://wbl.github.io/draft-ladd-bbs-privacypass/draft-ladd-privacypass-bbs.html"

author:
 -
    fullname: Watson Ladd
    organization: Akamai Technologies
    email: "watsonbladd@gmail.com"

normative:

informative:


--- abstract

Existing token types in privacy pass conflate attribution with rate limiting. This document describes a token type where the issuer attests to a set of properties of the client, which the client can then selectively prove to the origin. Repeated showings of the same credential are unlinkable, unlike other token types in privacy pass.

--- middle

# Introduction

In 2004 Boneh-Boyen-Shacham introduced the eponymous BBS signature. The BBS signature scheme as documented in {{!BBS=I-D.draft-irtf-cfrg-bbs-signatures}} lets a signer sign a sequence of strings called attributes, and provides a way for a holder of a signature to prove possession and the value of some of the attributes. This document explains how to use this technology with privacy pass.

# Conventions and Definitions

{::boilerplate bcp14-tagged}
# Protocol Overview

This protocol defines the use of Privacy Pass with selectively disclosable attributes, using {{BBS}} signatures and zero-knowledge proofs. Those attributes must be agreed upon by Client, Attester, and Issuer during issuance. Example of such attributes include public metadata values ({{!PPEXT=I-D.draft-hendrickson-privacypass-public-metadata}}).

To run this protocol the Issuer must have a public key and an issuance URL, as well as a common understanding of the meaning of each attribute string (sequence of octets). E.g an age might be encoded as a single octet, or in ASCII numeric base 10 represnetation, or a fixed field, and could be the first attribute in the list.

After the successful completion of the Issuance protocol, the Client is able to use the received `TokenResponse` to generate multiple unlinkable tokens.

## Issuance over generic transports

### Attribute Values

The Client begins by forming a sequence of strings corresponding to the attributes they wish signed. They engage in the Attestation protocol with the Attestor, and then send the serialized array of attributes to the Issuer. The way with which the list of attributes is communicated to the Issuer is considered protocol specific. An option is to serialize the attributes as a base64 encoded JSON array. The Issuer MUST parse the received attributes list before signing, and verify that each attribute is permitted by their polishes.

The Attestor interaction and validation of the attributes is not specified here. In a split instantiation as per {{?PPARCH=I-D.draft-ietf-privacypass-architecture}}, Attestors and Issuers MUST ensure that the claims by the Client are not changeable between attestation and signing.

### Header Value

Each attribute will be selectively disclosable by the Client during the redemption protocol. Protocol and application critical information (for example, a token type) need to be included in the header input value of the BBS signature generation operation as defined in [Section 3.4.1](https://www.ietf.org/archive/id/draft-irtf-cfrg-bbs-signatures-03.html#name-signature-generation-sign) of {{BBS}}. The header value will not be selectively disclosable, meaning that the Client will be required to reveal it during redemption. For HTTP based applications it is RECOMMENDED to include the token's type to the header. It is also RECOMMENDED the header value to be protocol or application defined. See [](#privacy-considerations) for more information.

### Token Response

If parsing and validating the received attributes was successful, the Issuer can sign them together with the selected header using the signature generation function of {{BBS}}. The Issuer then returns the signature and optionally the selected header (if it is not protocol defined) to the Client. The means with which those values are communicated are considered protocol specific. An option is to encode them using base64. The Client MUST verify the returned signature against their selected set of attributes and header value, using the signature verification operation as defined in [Section 3.4.2](https://www.ietf.org/archive/id/draft-irtf-cfrg-bbs-signatures-03.html#name-signature-verification-veri) of {{BBS}}.

## Redemption

After the received signature is verified, the Client can generate multiple unlinkable tokens, attesting over a (possibly different at each time) subset of the signed attributes. To do that, the Client uses the proof generation operation as defined in [Section 3.4.3](https://www.ietf.org/archive/id/draft-irtf-cfrg-bbs-signatures-03.html#name-proof-generation-proofgen) of {{BBS}}, with the presentation header input value set to a specified channel binding or origin identifier. For HTTP applications it is RECOMMENDED that the origin be used. The presentation header MUST additionally be bound to the received Origin challenge (for example by using the digest of the challenge value). The set of attributes the Client wishes to show is protocol specific and is communicated by means outside this draft.

Upon receiving a Token, the Origin must construct the required header and presentation header values, which may depend on protocol specific information (like the token's type) or session specific information (like a Client chosen nonce). The form of those values is protocol specific. The Origin MUST validate that both the header and presentation header, as well as the received attributes, conform to their policies (for example, that the presentation header contains a specific challenge digest etc.). Then, they can use the proof verification operation as defined in [Section 3.4.4](https://www.ietf.org/archive/id/draft-irtf-cfrg-bbs-signatures-03.html#name-proof-verification-proofver) of {{BBS}}, to verify the received Token and attributes.

# HTTP Protocol with Public Metadata

This section defines a HTTP based instatiation of the issuance and redemption protocol with public metadata ({{PPEXT}}). The BBS operations used are instantiated with the parameters defined by the `BBS_BLS12381G1_XMD:SHA-256_SSWU_RO_H2G_HM2S_` ciphersuite ([Section 6.2.2](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bbs-signatures-03#name-bls12-381-sha-256) of {{BBS}}). The signed attributes will consist of the public metadata values, giving to the Client the ability to selectively disclose distinguee subsets of metadata during different `Token` redemption attempts.

## Token Issuance

The Client and Issuer agree on the sequence of attributes, as well as the Issuer's public key. These attributes are put in a list of Extensions {{PPEXT}} with type TBD.

### Client-to-Issuer Request

The Client first creates an issuance request message using the Issuer key identifier as follows:

~~~
struct {
  uint16_t token_type = 0xTBD;  /* Type BBS with "BLS12-381-SHA-256" */
  uint8_t truncated_token_key_id;
} TokenRequest;
~~~

The structure fields are defines bellow:

- "token_type" is a 2-octet integer, which matches the type in the challenge.
- "truncated_token_key_id" is the least significant byte of the token_key_id in network byte order.

Using the constructed `TokenRequest`, the Client builds an `ExtendedTokenRequest`, defined in {{PPEXT}}, as follows:

~~~
struct {
  TokenRequest request;
  Extensions extensions;
} ExtendedTokenRequest;
~~~

The contents of the `ExtendedTokenRequest.extensions` value, MUST match the Client's configured extensions value. The Client then generates an HTTP POST request to send to the Issuer Request URI, with the `ExtendedTokenRequest` as the content.

### Issuer-to-Client Response

Upon receiving the Client's request, the Issuer needs to validate the following:

- The `ExtendedTokenRequest.TokenRequest` contains a supported `token_type`.
- The `ExtendedTokenRequest.TokenRequest` contains a `truncated_token_key_id` that corresponds to the truncated key ID of an Issuer Public Key.
- The `ExtendedTokenRequest.extensions` value is permitted by the Issuer's policy.

If any of these conditions is not met, the Issuer MUST return an HTTP 400 error to the Client. Otherwise, if the Issuer is willing to produce a token for the Client, they will complete the issuance flow by signing the agreed upon extension values. To do so, they first must parse the content value `ExtendedTokenRequest.extensions` and if successful, yield the extensions array. Then, they must sign those extensions, using the following steps:

~~~
header = 0xTBD
signature = Sign(skI, pkI,  header, extensions)
~~~

The Sign function is defined in [Section 3.4.1](https://www.ietf.org/archive/id/draft-irtf-cfrg-bbs-signatures-03.html#name-signature-generation-sign) of {{BBS}} and is instantiated with the parameters defined by the "BLS12-381-SHA-256" ciphersuite ([Section 6.2.2](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bbs-signatures-03#name-bls12-381-sha-256) of {{BBS}}). The result is encoded and transmitted to the client in the following `TokenResponse` structure:

~~~
struct {
  uint8_t signature[octet_point_length + octet_scalar_length];
} TokenResponse;
~~~

The `octet_point_length` and `octet_scalar_length` constants are defined in [Section 6.2](https://www.ietf.org/archive/id/draft-irtf-cfrg-bbs-signatures-03.html#name-bls12-381-sha-256) of {{BBS}}.

The Issuer generates an HTTP response with status code 200 whose content consists of `TokenResponse`, with the content type set as "application/private-token-response".

### Finalization

Upon receipt, the Client handles the response and, if successful, deserializes the content values `TokenResponse.signature` yielding the signature value. the Client MUST validate the produced signature as follows:

~~~
result = Verify(pkI, signature, 0xTBD, extensions)
~~~

The Verify function is defined in [Section 3.4.2](https://www.ietf.org/archive/id/draft-irtf-cfrg-bbs-signatures-03.html#name-signature-verification-veri) of {{BBS}} and is instantiated with the parameters defined by the "BLS12-381-SHA-256" ciphersuite ([Section 6.2.2](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bbs-signatures-03#name-bls12-381-sha-256) of {{BBS}}). If the result is not VALID, the Client MUST abort.

If signature verification is successful, the Client can generate multiple tokens with unlinkable authenticator values, using the procedure defined in [](#token-generation).

### Issuer Configuration

//TODO

## Token Redemption

This section describes an HTTP based protocol that allows Clients to generate and redeem tokens, using a `TokenResponse` received by the Issuer during the Issuance protocol defined in [](#token-issuance). The `TokenResponse` MUST first be verified using the procedure defined in [](#finalization).

### Token Generation

The Client can generate multiple Tokens with unlinkable authenticator values, using the ProofGen function as defined in [Section 3.4.3](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bbs-signatures-03#name-proof-generation-proofgen) of {{BBS}}, instantiated with the parameters defined by the "BLS12-381-SHA-256" ciphersuite ([Section 6.2.2](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bbs-signatures-03#name-bls12-381-sha-256) of {{BBS}}).

The Client can also select a subset of the extensions to disclose to the Origin. The means with which a Client will negotiate with the Origin the required extensions is outside the scope of this document. Let `disclosed_extensions_indexes` to be an ordered array of 2 byte integers from 1 to 65535, corresponding to the indexes that the extensions the Client wishes to disclose have, in the `Extensions.extensions` array. The client produces an authenticator value as follows:

~~~
nonce = random(32)
challenge_digest = SHA256(challenge)
header = 0xTBD
presentation_header = concat(nonce, challenge_digest)

authenticator = ProofGen(pkI,
                         signature,
                         header,
                         presentation_header,
                         extensions,
                         disclosed_extensions_indexes)
~~~

If the above calculation succeeds, the Client constructs a `Token` as follows:

~~~
struct {
  uint16_t token_type = 0xTBD;
  uint8_t nonce[32];
  uint8_t challenge_digest[32];
  uint8_t token_key_id[Nid];
  uint16_t disclosed_extensions_indexes<0..2^16-1>
  uint8_t authenticator<0..Nu>;
} Token;
~~~

The maximum length of the authenticator `Nu` is defined as `Nu = 2 * octet_point_length + (2^16 + 2) * octet_scalar_length`. Note, the addition of the `disclosed_extensions_indexes` field to the `Token` structure defined in {{?PPAUTH=I-D.draft-ietf-privacypass-auth-scheme}}.

### Token Verification

Upon receiving the `Token`, the Origin MUST parse it and validate that all the fields have the correct length. Additionally, the Origin MUST parse the `Token.disclosed_extensions_indexes` content value and verify that it comprised from an ordered array of integers from 1 to 65535. Lastly, the Origin MUST also parse the received (disclosed) `Extensions.extensions` value and create the array `disclosed_extensions`. Verifying a `Token` requires knowledge of the Issuer's public key (pkI) corresponding to the `Token.token_key_id` value. The Origin can verify the `Token` and corresponding disclosed extensions as follows:

~~~
header = 0xTBD
presentation_header = concat(Token.nonce, Token.challenge_digest)

res = ProofVerify(pkI,
                  Token.authenticator,
                  header,
                  presentation_header,
                  disclosed_extensions,
                  Token.disclosed_extensions_indexes)
~~~

The ProofVerify function is defined in Section 3.4.4 of {{BBS}}, instantiated with the parameters defined by the "BLS12-381-SHA-256" ciphersuite ([Section 6.2.2](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-bbs-signatures-03#name-bls12-381-sha-256) of {{BBS}}).

# Privacy Pass integration

In {{PPARCH}} parameters are provided that any instantiation must amend. TODO: put values in

# Security Considerations

 As the redeemed tokens are not single use, instantiations MUST specify a channel binding to use or origin identifier so that an Origin cannot harvest tokens for use at another origin.

# Privacy Considerations

The position of a revealed attribute, as well as the number of unrevealed attributes, is revealed to the origin. Applications MUST ensure all clients recieve the same set of attributes in the same positions to preserve privacy. The Issuer is visible on redemption, this creates partioning attacks. TODO: resolve through hiding them.

# IANA Considerations

We would like IANA to add to the Privacy Pass Token Type Registry the following registration:

Value: IANA picks
Name: BBS Token
Token structure: As in Token Generation Section
Token Key Encoding: TODO
Publicly Verifiable: Y
Public Metadata: Y
Private Metadata: N
Nk: Indefinite
NiD: 32? TODO: check this
Reference: This document

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
