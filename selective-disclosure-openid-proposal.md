# Verifiable credential presentation with selective disclosure using OpenID Connect Self-Issued

- Authors:
  - Danny Strockis [dastrock@microsoft.com](mailto:dastrock@microsoft.com)
  - Daniel Buchner [daniel.buchner@microsoft.com](mailto:daniel.buchner@microsoft.com)
- Last updated: 2019-01-29

## Status
- Status: **PROPOSAL**
- Status Date: 2020-01-29
- Status Note: Draft proposal for initial feedback

## Abstract
The [W3C Verifiable Credentials (hereafter VC) specification](https://www.w3.org/TR/vc-data-model) does not currently outline how credential data should be requested by a Verifier. This document outlines the approach currently being taken at Microsoft, which includes both message formats and a presentation protocol. This document is not written as an official W3C proposal - it may need to be refactored before contributing to standards bodies.

## Contents

- [Introduction](#Introduction)
    - Background on OpenID Connect Self-Issued
    - An aside on Credential Manifests
    - An aside on zero-knowledge proofs
- [Credential Structure](#Credential-Structure)
- [Presentation Request](#Presentation-Request)
- [Presentation Response](#Presentation-Response)
- [Future Work](#Future-Work)

## Introduction

In Fall 2019, several members of the DIF community successfully worked together to develop a working draft for DID authentication using the OpenID Connect standard. This work was led by Oliver Terbu at uPort, with contributions from participants at Consensys, Validated ID, Mattr, SecureKey, Digital Bazaar, Microsoft, Transmute, Evernym, and more. At IIW in October, Evernym and Microsoft demonstrated interoperable implementations of the standard that were used in a live customer pilot.

This proposal seeks to build upon that existing momentum, enabling verifiable credentials to be presented in a DID authentication flow.

Chapter 5 of the OpenID Connect specification provides affordances for selectively requesting individual claims from an identity providier using the `claims` parameter. It also introduces the concept of Aggregated Claims, which the spec defines as:

> Aggregated Claims: Claims that are asserted by a Claims Provider other than the OpenID Provider but are returned by OpenID Provider.

We believe these affordances can be used for verifiable credential presentation with support for selective disclosure without much modification of the OpenID Connect standard.

In our humble opinion, there are several advantages to leveraging OpenID Connect for verifiable credential presentation, such as:

- Many protocol design decisions, such as the names of parameters, signature formats, and message transport, are pre-determined for us. This significantly reduces the number of decisions we need to debate as a community.
- Because of OpenID Connect's widespread adoption, there's a wealth of tooling available to developers. The OpenID foundation has a certification suite that could be extended to include verifiable credentials. Open source implementations for OpenID Connect flows and JWTs exist in every programming language. Existing applications that currently support OpenID Connect could be altered to start accepting VCs. Using OpenID Connect allows us to build upon existing work rather than starting from scratch.
- Using OpenID Connect positions decentralized identity as a natural evolution of existing patterns and practices, rather than a pardigm shift. In our experience, this goes a long way towards making early adopters amenable to trying out verifiable credentials. Some organizations we work with have extensive review requirements and policies for introducing new security technology. By using OpenID Connect, JWTs, and familiar cryptography, we can mitigate some of the obstacles to adoption.

In this proposal, we've attempted to follow the OpenID Connect specification rather strictly. However, this proposal is only a starting point. If we'd like to make modifications to OpenID Connect, we can propose changes to the OpenID Foundation. Several authors of the OpenID Connect specification have expressed interest in working with us to evolve OpenID Connect to support decentralized identity and verifiable credentials.

The remainder of this document will use the example of a digital driver's license to demonstrate how verifiable presentation would work. This document seeks to accommodate the following selective disclosure variations:

1. Present the entire driver's license
2. Present only the birthdate attribute of the driver's license.
3. Present just the fact that the subject is over 21 years of age.
4. Present serveral attributes of the driver's license: height, weight, and gender.

For full disclosure, our team at Microsoft has not yet implemented this proposal. We are hoping that by sharing it early, we can collaborate with DIF to produce an implementation that many can benefit from.

### Background on OpenID Connect Self-Issued

If you're familiar with the following prior art, you'll have a much easier time understanding the proposal that follows.

- [OpenID Connect Core Specification](https://openid.net/specs/openid-connect-core-1_0.html)
    - Chapters 5, 6, and 7 are particularly releveant.
- [DIF's Self-Issued OpenID Connect Provider DID Profile](https://identity.foundation/did-siop/)
- [OpenID Connect for Identity Assurance](https://openid.net/2019/11/14/openid-connect-for-identity-assurance/)
    - Section 5 introduces the `purpose` parameter, which is used in this proposal.
- [JSON Web Tokens](https://tools.ietf.org/html/rfc7519)
- [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)
    - Section 6.3 gives examples of a verifiable credential in JWT format.

### An aside on credential manifests

This proposal may slightly overlap with DIF's developing concept of a Credential Manifest, which is currently defined as: 

> The Credential Manifest is a common data format for describing the inputs a Subject must provide to an Issuer for subsequent evaluation and issuance of the credential indicated in the Credential Manifest.

Throughout this document, we'll be assuming the verifiable driver's license has already been issued to the subject. There may be opportunities to incorporate ideas from this proposal into the credential manifest specification, or vice versa. But how the issuance process occurred is not in scope for this proposal. 

### An aside on zero-knowledge proofs

This proposal does not use zero-knowledge techniques as a means of providing selective disclosure. Many of our customer use cases require that highly correlatable PII be shared in a verifiable credential exchange, making it impossible to acheive the level of anonymity and privacy provided by anonymous credentials. We believe that there are some tactical advantages to using more traditional digital signatures for these use cases.

With that said, we recognize that there are many important use cases where the anonymity and privacy provided by zero-knowledge proofs is highly desireable. We're currently working with Microsoft Research to develop an implementation of verifiable credentials based on zero-knowlege techniques. Outside of this document, we're interested in working with the community to standardize presentation of zero-knowledge verifiable credentials.

In this document, selective disclosure is achieved by "pre-generation of atomic credentials" (credit for that phrase goes to Mike Lodder). As the next section describes, we're proposing to decouple a verifiable claim from its verifiable credential, using hashes to link the two and enable selective disclosure while reducing the number of signatures produced.

## Credential Structure

A verifiable credential has the following structure:

![Basic VC Structure](img/vc-structure.png)

In this proposal, each claim is separated out from the verifiable credential's metadata and proofs, into a dedicated object we're calling a **verifiable claim** (we're taking other suggestions). The value of each attribute in a verifiable credential is a hash of a verifiable claim.

Note that in this example, each claim only contains a single key-value pair of data. This is not a restriction; a claim may contain a rich set of data if the issuer so chooses.

```
// A verifiable credential in JWT format

// JWT header
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "did:example:issuer#keys-1"
}
.
// JWT payload
{
  "@context": ["https://www.w3.org/2018/credentials/v1", "https://stateagency.gov"],
  "@type": ["VerifiableCredential", "DriversLicense"],
  "iss": "did:example:issuer",
  "sub": "did:example:subject",
  "jti": "http://issuer.com/credentials/3732",
  "nbf": 1541493724,
  "iat": 1541493724,
  "exp": 1573029723,
  "credentialSubject": {
    "birthDate": {"claimHash": "jknagbiab4on2tn3...", "alg": "SHA256"},
    "address": {"claimHash": "nnfad18931n134...", "alg": "SHA256"},
    "motorClass": {"claimHash": "jklan2e232jfna..."}, "alg": "SHA256",
    "name": {"claimHash": "xvncxvzvfdafqv..."}, "alg": "SHA256",
    "gender": {"claimHash": "qeqreqtqgqrbbe...", "alg": "SHA256"},
    "heightWeight": {"claimHash": "puiupjlknnkoj...", "alg": "SHA256"},
    "dlNumber": {"claimHash": "j4892424jnj1243...", "alg": "SHA256"},
    "isOver21": {"claimHash": "jldkne1ass4141...", "alg": "SHA256"},
  },
  ...
  "credentialStatus": {
    "id": "https://issuer.com/status/24",
    "type": "CredentialStatusList2017"
  }
}
.
// JWT signature
iOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImRpZDpleGFtcGxlOmFiZmUxM2Y3MTIxMjA0
MzFjMjc2ZTEyZWNhYiNrZXlzLTEifQ.eyJzdWIiOiJkaWQ6ZXhhbXBsZTplYmZlYjFmNzEyZWJjNmYxY
zI3NmUxMmVjMjEiLCJqdGkiOiJodHRwOi8vZXhhbXBsZS5lZHUvY3JlZGVudGlhbHMvMzczMiIsImlzc
```

| Parameter | Description |
| --------- | ----------- |
| `alg` | The algorithm used to sign the presentation response, per JWT specification. |
| `typ` | Must be `JWT`, per JWT specification. |
| `kid` | The key identifier, per W3C Verifiable Credentials specification section 6.3. |
| `@context` | The context(s) of the credential, which indicate the type of the credential. |
| `@type` | The type(s) of the credential, which indicate the type of the credential. |
| `iss` | The DID of the issuer, per W3C Verifiable Credentials specification section 6.3. |
| `sub` | The DID of the subject, per W3C Verifiable Credentials specification section 6.3. |
| `jti` | A unique identifier for this credential, per W3C Verifiable Credentials specification section 6.3. |
| `nbf` | The start time for the validity of the credential, per JWT specification. |
| `iat` | The issuance time of the credential, per JWT specification. |
| `exp` | The expiration time of the credential, per JWT specification. |
| `credentialSubject` | Contains the contents of the credential, per W3C Verifiable Credentials specification. |
| `credentialSubject.{attribute}` | Each attribute included in the credential, per W3C Verifiable Credentials specification section 6.3.  |
| `credentialSubject.{attribute}.claimHash` | The hashed value of each verifiable claim, whose structure is described below.  |
| `credentialSubject.{attribute}.alg` | The algorithm used to hash the verifiable claim.  |
| `credentialStatus` | The revocation mechanism, per W3C Verifiable Credentials specification. |
| signature | The digital signature of the issuer, as a JSON web signature. |

The verifiable credential represented above is accompanied by a verifiable claim for each attribute in the credential. A verifiable credential takes the following format. 

```
{
  "@context": "https://www.w3.org/2018/credentials/v1",
  "@type": "VerifiableClaim",
  "nonce": "1b99aee8-64d4-471a-8d74-79691abc3bb0",
  "claim": {
    "birthDate": "09/24/91",
  }
}
```

| Parameter | Description |
| --------- | ----------- |
| `@context` | The context of the object, which is always `https://www.w3.org/2018/credentials/v1`. This may need to be modified to properly support JSON-LD. |
| `@type` | The type of the object, which is always `VerifiableClaim`. This may need to be modified to properly support JSON-LD. |
| `nonce` | A randomization factor which is used to redact/disguise the hashed value of the claim. |
| `claim` | The claim's unredacted value, which may include more than a single key-value pair. |

With this credential structure, a holder can selectively present individual verifiable claims along with a verifiable credential, revealing only a subset of the credential's claims to the verifier. Upon presentation, the verifier can hash the verifiable claim and compare the result to the corresponding `claimHash` value in the verifiable credential.

When compared to individually signed attributes, we believe this approach has two advantages:

- It reduces the number of signatures that must be produced by the issuer and verified by the verifier. This may slightly decrease the size of a credential, and the time it takes to perform verification. 
- It binds the various attributes of a credential to each other, preventing a holder from deceiving the verifier by presenting two similar claims and pretending they originated from the same credential. A toy example: Alice has a double major in engineering and literature from the same university. She receives a verifiable credential for each degree she achieved. But, her engineering GPA was a 4.0, while her literature GPA was a 2.3. When she applies for a writing job, she is able to deceive the verifier by submitting her individually signed 4.0 GPA attribute with her literature degree.

As mentioned above, this approach does not help prevent correlation between issuers and verifiers, or between multiple verifiers who receive the same credential. The same signatures, hashes, and other correlatable data are included with each presentation.

The remainder of this document outlines how verifiable credentials with this structure can be presented using the OpenID Connect Self-Issued protocol, facilitating selective disclosure between a holder and verifier.


## Presentation Request

A verifiable credential presentation begins by the verifier sending the following request to the holder.

```
openid://auth?response_type=id_token&client_id=https%3A%2F%2Fverifier.com/presentation/response&scope=openid%20did_authn&request_uri=https%3A%2F%2Fverifier.com/presentation/requests/098d4630-2da7-4b9e-b585-1b493bd1cdd0
```

| Parameter | Description |
| --------- | ----------- |
| `response_type` | Must be `id_token`, per OpenID Connect specification. |
| `client_id` | The URI to which the presentation response will be returned, per OpenID Connect specification. |
| `scope` | Must include `openid`, per OpenID Connect specification, and `did_authn`, per DID SIOP specification. |
| `request_uri` | A reference to the presentation request, which is served by the issuer. This is a mechanism used to keep the above request short, so that it can be encoded into QR images, deeplinks, and other mediums. If length is not a concern, the `request_uri` property can be omitted in favor of the `request` property.  |
| `request` | (not included in example above) The presentation request (described below) encoded as a JWT. |

The `request_uri` is then resolved, returning a presentation request in JWT format (decoded below for readability):

```
// JWT header
{
   "alg": "ES256K",
   "typ": "JWT",
   "kid": "did:example:verifier#veri-key1"
}
.
// JWT payload
{
    "iss": "did:example:verifier",
    "response_type": "id_token",
    "client_id": "https://verifier.com/presentation/response",
    "scope": "openid did_authn",
    "state": "af0ifjsldkj",
    "nonce": "n-0S6_WzA2Mj",
    "response_mode" : "form_post",
    "registration" : {
        "jwks_uri" : "https://uniresolver.io/1.0/identifiers/did:example:verifier;transform-keys=jwks",
        "id_token_signed_response_alg" : [ "ES256K", "EdDSA", "RS256" ]
    },
    "claims": {
        "id_token": {
            "https://stateagency.gov/DrivingLicenseCredential": {
                "essential": "true",
                "purpose": "To verify your age.",
                "claims": ["isOver21", "heightWeight", "firstName"]
            }
        }
    }
}
.
// JWT signature
KLJo5GAyBND3LDTn9H7FQokEsUEi8jKwXhGvoN3JtRa51xrNDgXDb0cq1UTYB-rK4Ft9YVmR1NI_ZOF8oGc_7wAp
8PHbF2HaWodQIoOBxxT-4WNqAxft7ET6lkH-4S6Ux3rSGAmczMohEEf8eCeN-jC8WekdPl6zKZQj0YPB
1rx6X0-xlFBs7cl6Wt8rfBP_tZ9YgVWrQmUWypSioc0MUyiphmyEbLZagTyPlUyflGlEdq
```

| Parameter | Description |
| --------- | ----------- |
| `alg` | The algorithm used to sign the presentation request, per OpenID Connect specification. |
| `typ` | Must be `JWT`, per OpenID Connect specification. |
| `kid` | The key identifier, per DID SIOP specification. |
| `iss` | The DID of the verifier, per DID SIOP specification. |
| `response_type` | Must be `id_token`, per OpenID Connect specification. |
| `client_id` | The URI to which the presentation response will be returned, per OpenID Connect specification. |
| `scope` | Must include `openid`, per OpenID Connect specification, and `did_authn`, per DID SIOP specification. |
| `state` | Optional data that is returned to the verifier in the presentation response, per OpenID Connect specification. |
| `nonce` | A randomized challenge for preventing response replay, per OpenID specification. |
| `response_mode` | Can be `query`, `fragment`, or `form_post`, per OpenID Connect specification. |
| `registration` | Includes any details about the client (the verifier in this case) necessary to register the client with the user, per OpenID Connect Dynamic Client Registration. |
| `registration.jwks_uri` | A URI that can resolve the public keys needed to validate the request, per DID SIOP specification. |
| `registration.id_token_signed_response_alg` | The accepted algorithms for the presentation response, per DID SIOP specification. |
| `claims` | Includes the claims requested by the verifier, which may include verifiable credentials. |
| `claims.id_token` | Indicates that the requested claims should be returned in the resulting presentation response, per OpenID Connect specification. |
| `claims.id_token.{schema-uri}` | The type of the requested verifiable credential as a URI.  |
| `claims.id_token.{schema-uri}.essential` | Indicates if the requested credential is required or not, per the OpenID Connect specification. |
| `claims.id_token.{schema-uri}.purpose` | Displayable string that describes the reason the verifier is requesting the credential, per OpenID Connect Identity Assurance draft. |
| `claims.id_token.{schema-uri}.claims` | An array of attributes that are being requested from the selected credential. If this property is omitted, it is assumed that all attributes are requested. The available values of the array are inferred from the credential's schema. |
| `claims.id_token.{schema-uri}.{other}` | The OpenID Connect specification allows for additional fields and structures to be included here for additional criteria. |
| signature | The digital signature of the verifier, as a JSON web signature. |

## Presentation Response

The holder then returns a presentation response to the verifier.


```
// JWT header
{
    "alg": "ES256K",
    "typ": "JWT",
    "kid": "did:example:subject#key-1"
}
.
// JWT payload
{
   "iss": "https://self-issued.me",
   "state": "af0ifjsldkj",
   "nonce": "n-0S6_WzA2Mj",
   "exp": 1311281970,
   "iat": 1311280970,
   "sub_jwk" : {
      "crv":"secp256k1",
      "kid":"did:example:subject#verikey-1",
      "kty":"EC",
      "x":"7KEKZa5xJPh7WVqHJyUpb2MgEe3nA8Rk7eUlXsmBl-M",
      "y":"3zIgl_ml4RhapyEm5J7lvU-4f5jiBvZr4KgxUjEhl9o"
   },
   "sub": "9-aYUQ7mgL2SWQ_LNTeVN2rtw7xFP-3Y2EO9WV22cF0",
   "did": "did:example:subject",
   "_claim_names": {
      "https://stateagency.gov/DrivingLicenseCredential": "src1"
   },
   "_claim_sources": {
      "src1": {
        "JWT": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
        "birthDate": "eyJvdfgadgagagdbba...",
        "heightWeight": "eyJhbjkfldan2f32q...",
        "name": "eyJh901f3j41lf41l14..."
      }
   }
}
.
// JWT signature
hhbXBsZS5jb20va2V5cy9mb28uandrIiwibmJmIjoxNTQxNDkzNzI0LCJpYXQiO
jE1NDE0OTM3MjQsImV4cCI6MTU3MzAyOTcyMywibm9uY2UiOiI2NjAhNjM0NUZTZXIiLCJ2YyI6eyJAY
29udGV4dCI6WyJodHRwczovL3d3dy53My5vcmcvMjAxOC9jcmVkZW50aWFscy92MSIsImh0dHBzOi8vd
```

| Parameter | Description |
| --------- | ----------- |
| `alg` | The algorithm used to sign the presentation response, per OpenID Connect specification. |
| `typ` | Must be `JWT`, per OpenID Connect specification. |
| `kid` | The key identifier, per DID SIOP specification. |
| `iss` | Must be `https://self-issued.me`, per OpenID Connect specification. |
| `state` | Optional data that is returned to the verifier in the presentation response, per OpenID Connect specification. |
| `nonce` | A randomized challenge for preventing response replay, per OpenID specification. |
| `exp` | The expiration time of the presentation response, per OpenID Connect specification. |
| `iat` | The issuance time of the presentation response, per OpenID Connect specifiication. |
| `sub_jwk` | The public key that can be used to verify the response, in JWK format per OpenID Connect specification. |
| `sub` | The thumbprint of the public key in `sub_jwk`, per OpenID Connect specification. |
| `did` | The DID of the subject, per DID SIOP specification. |
| `_claims_names` | The requested claim types, with a reference to the claim sources where the claim values can be found, per OpenID Connect specification. |
| `_claim_sources` | The values satisfying the requested claims & criteria, per OpenID Connect specification. |
| `_claims_sources.{source}.JWT` | A verifiable credential satsifying the request, in JWT proof format. The key `JWT` is used here per the OpenID Connect specification. |
| `_claim_sources.{source}.{claim}` | The verifiable claims satisfying the request, as base64 encoded JSON objects. These values can be hashed and compared to the `claimHash` value in the verifiable credential to ensure they are bound to the verifiable credential that has been presented. |
| signature | The digital signature of the holder, as a JSON web signature. |


## Future Work

The following are ideas for next steps and improvement upon this proposal.

- Formalize the details of this proposal, and turn it into a proper specification. Several details remain unspecified, like the accepted hash algorithms for verifiable claims.
- Work with the OpenID Foundation to make modifications to the existing specifications in support of verifiable credential features and scenarios. 
- The proposal above is very limited in terms of the "criteria" that verifiers can specify in their presentation requests. Additional fields in the presentation request can be added to offer richer request functionality, akin to that proposed in the credential manifest proposals.
- Some have suggested replacing the simple hashing scheme used in this proposal with a Merkle tree or other cryptographic means of testing the membership of a verifiable claim to a verifiable credential.
- Incorporation of zero-knowledge proofs into the request & response protocol, if possible.
