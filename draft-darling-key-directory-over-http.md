---
title: "Key Directory over HTTP"
category: info

docname: draft-darling-key-directory-over-http-latest
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
  github: "thibmeu/draft-darling-key-directory-over-http"
  latest: "https://thibmeu.github.io/draft-darling-key-directory-over-http/draft-darling-key-directory-over-http.html"

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

This document defines recommendations for protocol implementers that wish to
expose public keys over HTTP. This allows new protocols to share public keys
via HTTP with considerations for caching, rotation, or providing a well-known
URL.


--- middle

# Introduction

Multiple Internet protocols rely on public key cryptography. They require keys
to be distributed by origins to clients. This is done via certificates,
via software releases, or via HTTP. This document focuses on this last mechanism.
It aims to set recommendation on how to design a key directory that should be
served over HTTP.

Distribution via HTTP allows for a more dynamic use of public keys, for rotation,
or caching on intermediate servers or clients.
This document specifies how clients and mirrors should consume cache directive
set by origins, how origins should expose their key directory, and rotate them.
The document does not cover a specific directory format, as these needs might
vary from one protocol to the next.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

Client:
: An entity using public key material

Origin:
: An entity exposing public key material via HTTP

Mirror:
: An intermediary entity between client and server. May cache data, and act as a
  privacy proxy

Key metadata:
: Public data associated to a public key

Key Directory:
: Set of public keys

Directory Metadata:
: Public data associated to a key directory. This is protocol specific.


# Architecture

A server is exposing a key directory for clients to fetch. Clients MAY fetch the
directory from a Mirror, either to protect its privacy, or because the Server
wants to leverage a content delivery network.

This document focuses on the below interaction, which is triggered when the
Client does not have valid key for the server. This can be because the Client is
new, its cache is expired, or the server refuses requests with the current key set.

~~~aasvg
+--------+                +--------+               +--------+
| Client |                | Mirror |               | Origin |
+---+----+                +---+----+               +----+---+
    |                         |                         |
    +--- GET Key Directory -->|                         |
    |                         +--- GET Key Directory -->|
    |                         |<---- Key Directory -----+
    |                         +---.                     |
    |                         |    | cache              |
    |                         |<--'                     |
    |<---- Key Directory -----+                         |
    +---.                     |                         |
    |    | cache              |                         +---.
    |<--'                     |                         |    | rotate
    |                         |                         |<--'
    |                         |                         |
~~~


## Cache behaviour

Cache control, intermediaries

Client should use cache directive per key, so that if a key becomes invalid,
they use the next one in their cache.

TODO: write about key preference, based on not-before, not-after, or order in
the directory.

## Key id

Protocol implementers pass a blob of data based on public key material/others
Key ID = Hash(blob)

Key ID have an associated truncated Key ID. The size of truncated Key ID depends
on the number of keys a directory expects to serve at one point in time

| Keys in directory | Truncated key ID bytes |
|:------------------|:-----------------------|
| 1-255             | 1                      |
| 256-65535         | 2                      |
| 65535-*           | Not supported          |

Truncated key ID = last bytes of key ID in network byte order.

## Rotation (kid, cache, others)

### Algorithm

We approach a public key generation by the function `Generate(params, RAND) -> (publickey, privatekey, metadata)`

At any point in time, keys in the directory MUST have a unique truncated key id.
When adding a key in the directory, that key MUST have a unique truncated key id.

Generation looks as follow

~~~
do
  (publickey, privatekey, metadata) <- Generate(params, RAND)
  keyid <- H(publickey|metadata)
  truncated_key_id <- last_bytes(keyid)
while (truncated_key_id is not unique)
~~~

### Scheduled (server, client behaviour)

Two options:

* passive = rely on cache header to set not-after on the client side. stop
  advertising the key at time t, and delete it at time t+maxage
  take intermediate into consideration
* active = keep serving the key but add not-after before expiration. It should
  be NOW()+maxage

In both case, the protocol MUST define an error to signal a key which is not
supported. It is RECOMMENDED to use truncated token ID as a short identifier.

### Immediate (server, client behaviour)

There are moment where keys have to be rotated immediatly. Existing keys may
have to be invalidated and/or new keys be provisioned. Immediate keys rotation
may happen in the event of a key compromise, loss, or other imperious reason.

Immediate key rotation will cause some client request to the server to fail
until they retrieve a new version of the directory. The key directory endpoint
is going to be placed under a higher load.

Client requests are expected to fail.

1. You MAY introduce a random backoff to spread the load of key distribution over
time
2. Clients on a scheduled rotation MAY be configured to distrust rotation outside
a fixed schedule. Protocols SHOULD define such policies.

## Well known URL

It is RECOMMENDED protocol register a {{!WELL-KNOWN=rfc8615}}URL and associated
content-type.

A key directory server MUST support both GET and HEAD request on that endpoint.

~~~
GET /.well-known/<your-protocol>

HTTP/1.1 200 OK
Cache-Control: max-age=300
Content-Type: <your-protocol>
~~~


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


# Deployment Considerations

Rotation schedule: fast?
Proxy improves client experience and shields key directory server


# Privacy Considerations

TODO Privacy

Clients fetching keys mean they reveal their IP, time, and other informations.
When the key directory is for an external service, Clients SHOULD consider
proxying their traffic through a mirror server. Mirrors SHOULD NOT collide with
the key server.


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
