---
title: "Key Directory over HTTP"
category: info

docname: draft-darling-ohai-hpke-key-directory-over-http-latest
submissiontype: IETF
consensus: true
v: 3
area: ""
workgroup: "???"
keyword:
 - Internet Draft
 - Publickey
 - Key directory
 - Key rotation
 - Key cache
 - Key ID
venue:
  group: "???"
  type: ""
  mail: "httpapi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/httpapi/"
  github: "thibmeu/draft-darling-ohai-hpke-key-directory-over-http"
  latest: "https://thibmeu.github.io/draft-darling-ohai-hpke-key-directory-over-http/draft-darling-ohai-hpke-key-directory-over-http.html"

author:
 -
    fullname: Fisher Darling
    organization: Cloudflare Inc.
    # email: your.email@example.com
 -
    fullname: Thibault Meunier
    organization: Cloudflare Inc.
    email: ot-ietf@thibault.uk
 -
    fullname: Simon Newton
    organization: Cloudflare Inc.
    email: rfc@simonnewton.com

normative:

informative:


--- abstract

Service owners have a use to share public key as part of a standard directory.
This is meant to be convenient to share, allow for rotation, and be secure to
man-in-the-middle or split-view attack.
This document defines recommendations for protocol implementers.

Notes:
JWK does not offer a standard endpoint either with jose or cose. It has a draft
for HPKE [1]/[2]. There is an endpoint /.well-known/jwks.json that is tribal
knowledge but is not registered with IANA.
Certificates could also be a similar mechanism, but there is x509 as well.
This directory might be bootstraped with x509 backed trust, but this is an
overhead we don't touch. I'm not confident in defining a jwk subset (all attributes [3]).

[1] https://datatracker.ietf.org/doc/draft-ietf-jose-hpke-encrypt/
[2] https://datatracker.ietf.org/doc/draft-ietf-cose-hpke/
[3] https://www.iana.org/assignments/jose/jose.xhtml

--- middle

# Introduction

Multiple Internet protocols rely on public key cryptography. They require keys
to be distributed by servers to clients. This could be done via certificates,
via software releases, or via HTTP. This document focuses on this last mechanism.
It aims to set recommendation on how to design a key directory that should be
served over HTTP.

Distribution via HTTP allows for a more dynamic use of public keys, for rotation,
or caching on intermediate servers or clients.

The document covers:

1. cache behaviour
2. how to populate key id
3. possibly well known url and client behaviour
4. rotation considerations (kid, cache, others), to minimize key compromise

Format is a lot more work (likely different as well), so reusing COSE key set and jwk set could work for examples (maybe more, not sure)


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# System

At a high level, keys are base64url encoded.
Endpoint is in JSON and CBOR, with a specific Content-Type but NOT a different
endpoint.
Consistency can be done with [1]
Integration with key transparency [2] should be considered.
Rotation is done via HTTP headers. A key is valid until this header expires, even though it's removed before.
A key has a `not-before` in Unix timestamp, but DOES NOT have `not-after`.
Keys should .
Key IDs MAY have pet names, but the actual IDs should be derived from the key data. This
allows to provide some guarantees over key uniqueness/version. A key MUST NOT have a version.

## Future considerations

These considerations should be addressed in future drafts.

### Consistency

Consistency allows client to prevent themselves from split view attack. A
proposal that has been made for Privacy Pass is to use multiple mirrors 
{{!CONSISTENCY=I-D.ietf-privacypass-consistency-mirror}}. With a
sufficiently high quorum, clients get more confident that they are not singled
out.
It presents scalability issues as you need multiple mirrors, and have one more
requests from client per mirror in the quorum.

### Key Transparency

Key Directory over HTTP should integrate with transparency, once the protocol has
been defined in {{!KEYTRANS=I-D.ietf-keytrans-protocol}}.
There are specific consideration as to what goes in the log: the full
directory, keys individually, privacy considerations.


# Privacy Considerations

TODO Privacy

Clients fetching keys mean they reveal their IP, time, and other informations.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Test vectors

List how to test cache
List how to test rotation

# Use cases

See existing key directory on https://key-directory-over-http.research.cloudflare.com/

## DAP

HpkeConfigList [1]

~~~tls
HpkeConfig HpkeConfigList<0..2^16-1>;

struct {
  HpkeConfigId id;
  HpkeKemId kem_id;
  HpkeKdfId kdf_id;
  HpkeAeadId aead_id;
  HpkePublicKey public_key;
} HpkeConfig;

opaque HpkePublicKey<0..2^16-1>;
uint16 HpkeAeadId; /* Defined in [HPKE] */
uint16 HpkeKemId;  /* Defined in [HPKE] */
uint16 HpkeKdfId;  /* Defined in [HPKE] */
~~~

Partially informed comments:
* HpkeConfigId could be removed
* Need not-before to handle early capture

[1] https://datatracker.ietf.org/doc/html/draft-ietf-ppm-dap-13#section-4.5.1

## OHTTP

Key Configutation [1]

~~~tls
HPKE Symmetric Algorithms {
  HPKE KDF ID (16),
  HPKE AEAD ID (16),
}

Key Config {
  Key Identifier (8),
  HPKE KEM ID (16),
  HPKE Public Key (Npk * 8),
  HPKE Symmetric Algorithms Length (16) = 4..65532,
  HPKE Symmetric Algorithms (32) ...,
}
~~~

Partially informed comments:
* Key Identifier could be removed/be deterministic
* No mention of not-before
* No mention of HTTP Caching for rotation

[1] https://www.ietf.org/rfc/rfc9458.html#name-key-configuration

## Privacy Pass

Issuer directory [1]

~~~tls
 {
    "issuer-request-uri": "https://issuer.example.net/request",
    "token-keys": [
      {
        "token-type": 2,
        "token-key": "MI...AB",
        "not-before": 1686913811,
      },
      {
        "token-type": 2,
        "token-key": "MI...AQ",
      }
    ]
 }
~~~

Partially informed comments:
* Not as flexible as HPKE
* Has some protocol metadata (token-type, issuer-request-uri, rate-limit)

[1] https://www.rfc-editor.org/rfc/rfc9578#name-configuration

## Masque relay

..

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
