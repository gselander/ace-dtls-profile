---
coding: utf-8

title: Datagram Transport Layer Security (DTLS) Profile for Authentication and Authorization for Constrained Environments (ACE)
abbrev: CoAP-DTLS
docname: draft-gerdes-ace-dtls-authorize-latest
date: 2016-09-12
category: std

ipr: trust200902
area: Security
workgroup: ACE Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Gerdes
    name: Stefanie Gerdes
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63906
    email: gerdes@tzi.org
 -
    ins: O. Bergmann
    name: Olaf Bergmann
    organization: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63904
    email: bergmann@tzi.org
 -
    ins: C. Bormann
    name: Carsten Bormann
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63921
    email: cabo@tzi.org

normative:
  RFC2119:
  RFC3986:
  RFC4279:
  RFC6838:
  RFC5746:
  RFC6347:
  RFC7049:
  RFC7252:
  RFC5226:

informative:
  RFC5988:
  RFC6655:
  RFC6690:
  RFC7641:
  RFC7959:
  I-D.ietf-selander-ace-object-security:

entity:
        SELF: "[RFC-XXXX]"

--- abstract

This specification defines a profile for delegating client
authentication and authorization in a constrained environment for
establishing a Datagram Transport Layer Security (DTLS) channel between resource-constrained nodes.
The protocol relies on DTLS to
transfer authorization information and session keys between entities in a constrained network. A
resource-constrained node can use this protocol to delegate
authentication of communication peers and management of authorization
information to a trusted host with less severe limitations regarding
processing power and memory.

--- middle


# Introduction

(See Abstract for now.)

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 {{RFC2119}}.

Readers are expected to be familiar with the terms and concepts
described in {{I-D.ietf-ace-oauth-authz}}.

# Protocol

The CoAP-DTLS profile for ACE comprises three parts:

1. transfer of authentication and, if necessary, authorization
   information between C and RS;
1. transfer of access requests and the
   respective ticket transfer between C and AS; and
1. transfer of ticket requests and the
   respective ticket grants between AS and C.

## Overview

In {{protocol-overview}}, a protocol flow for this profile is depicted (messages
in square brackets are optional):

~~~~~~~~~~~~~~~~~~~~~~~

   C                            RS                   AS
   | [-- Resource Request --->] |                     |
   |                            |                     |
   | [<----- AS Information --] |                     |
   |                            |                     |
   | --- Token Request  ----------------------------> |
   |                            |                     |
   | <---------------------------- Access Token ----- |
   |                             + Client Information |
   |                            |                     |
   | <== DTLS channel setup ==> |                     |
   |     + Access Token         |                     |
   |                            |                     |
   | == Authorized Request ===> |                     |
   |                            |                     |
   | <=== Protected Resource == |                     |

~~~~~~~~~~~~~~~~~~~~~~~
{: #protocol-overview title="Protocol overview"}

To determine the AS in charge of a resource hosted at the RS, C MAY
send an initial Unauthorized Resource Request message to RS.  RS then
denies the request and sends the address of its AS back to C.

Instead of the initial Unauthorized Resource Request message, C MAY
look up the desired resource in a
resource directory (cf. {{I-D.ietf-core-resource-directory}}) that
lists RS's resources as discussed in {{rd}}.

Once C knows AS's address, it can send an access token request to
the /token endpoint at the AS as specified in {{I-D.ietf-ace-oauth-authz}}.
If AS decides that the request is to be authorized it
generates an access token response for C containing a `profile` parameter
with the value `coap_dtls` to indicate that this profile MUST be used,
and a `cnf` parameter with additional data for the establishment of a
secure DTLS-channel between C and RS.

The following sections specify how CoAP is used to interchange
access-related data between RS and AS so that AS can
provide C and RS with sufficient information to establish a secure
channel, and simultaneously convey
authorization information specific for this communication relationship
to RS.

## Unauthorized Resource Request Message {#rreq}

The optional Unauthorized Resource Request message is a request for a resource
hosted by RS for which no proper authorization is granted. RS MUST
treat any CoAP request as Unauthorized Resource Request message when any of the
following holds:

* The request has been received on an unprotected channel.
* RS has no valid access token for the sender of the
  request regarding the requested action on that resource.
* RS has a valid access token for the sender of the
  request, but this does not allow the requested action on the requested
  resource.

Note: These conditions ensure that RS can handle requests autonomously
once access was granted and a secure channel has been established
between C and RS.

Unauthorized Resource Request messages MUST be denied with a client error
response. In this response, the Reource Server MUST provide
proper AS Information to enable the Client to request an
access token from RS's Authorization Server as described in {{as-info}}.

The response code MUST be 4.01 (Unauthorized) in case the sender of
the Unauthorized Resource Request message is not authenticated, or if
RS has no valid access token for C. If RS has an access token for C
but not for the resource that C has requested, RS
MUST reject the request with a 4.03 (Forbidden). If RS has
an access token for C but it does not cover the action C
requested on the resource, RS MUST reject the request with a 4.05
(Method Not Allowed).

Note:
: The use of the response codes 4.03 and 4.05 is intended to prevent
  infinite loops where a dumb Client optimistically tries to access
  a requested resource with any access token received from AS.
  As malicious clients could pretend to be C to determine C's
  privileges, these detailed response codes must be used only when a
  certain level of security is already available which can be achieved
  only when the Client is authenticated.

##  AS Information {#as-info}

The AS Information is sent by RS as a response to an
Unauthorized Resource Request message (see {{rreq}}) to point the sender of the
Unauthorized Resource Request message to RS's AS. The AS
information is a set of attributes containing an absolute URI (see
Section 4.3 of {{RFC3986}}) that specifies the AS in charge of RS.

An optional field `profile` indicates the preferred ACE profile for RS.

TBD: We might not want to add more parameters in the AS information because
: this would not only reveal too much information about RS's
: capabilities to unauthorized peers but also be of little value as C
: cannot really trust that information anyway.

The message MAY also contain a timestamp generated by RS. <!-- (RS wants either his own timestamp or a timestamp generated by AS(RS) back in the end to make sure the information it gets is fresh)-->

{{as-info-payload}} shows an example for an AS Information message
payload using CBOR diagnostic notation. (Refer to {{payload-format}} for a detailed
description of the available attributes and their semantics.)

~~~~~~~~~~
    4.01 Unauthorized
    Content-Format: application/ace+cbor
    {AS: "coaps://as.example.com/token", TS: 168537,
     profile: coap_dtls}
~~~~~~~~~~
{: #as-info-payload title="AS Information payload example"}

In this example, the attribute As points the receiver of this message
to the URI "coaps://as.example.com/token" to request access
permissions. The originator of the AS Information payload
(i.e., RS) uses a local clock that is loosely synchronized with a time
scale common between RS and AS (e.g., wall clock time). Therefore, it has included a time stamp on its own time
scale that is used as a nonce for replay attack prevention.

Note: There is an ongoing discussion how freshness of access tokens
: can be achieved in constrained environments. This specification for
: now assumes that RS and AS have a common understanding of time that
: allows RS to achieve its security objectives.

The examples in this document are written in CBOR diagnostic notation
to improve readability. {{as-info-cbor}} illustrates the binary
encoding of the message payload shown in {{as-info-payload}}.

~~~~~~~~~~
a2                                   # map(2)
    00                               # unsigned(0) (=AS)
    78 1c                            # text(28)
       636f6170733a2f2f61732e657861
       6d706c652e636f6d2f746f6b656e # "coaps://as.example.com/token"
    05                               # unsigned(5) (=TS)
    1a 00029259                      # unsigned(168537)
    14                               # unsigned(20) (=profile)
    81                               # array(1)
       01                            # unsigned(1) (=coap_dtls)
~~~~~~~~~~
{: #as-info-cbor title="AS Information example encoded in CBOR"}

## Client-to-AS Request {#access-request}

To retrieve an access token for the resource that C wants to access, C
requests an Access Token from AS. The Access Token request is
constructed as specified in {{I-D.ietf-ace-oauth-authz}}. The
parameter `profile` MUST be set to `coap_dtls`. If a symmetric
proof-of-possession key (c.f. {{I-D.ietf-ace-oauth-authz}}) is
requested C must ensure that the Access Token request is sent over a
secure channel that guarantees authentication, message integrity and
confidentiality. This could be, e.g., a DTLS channel (for "coaps") or
an OSCOAP {{I-D.ietf-selander-ace-object-security}} exchange (for
"coap").

An example Access Token request from C to RS is depicted in
{{authorization-message-example}}. (Refer to {{payload-format}} for a detailed
description of the available attributes and their semantics.)

~~~~~~~~~~
   POST coaps://as.example.com/token
   Content-Format: application/cbor
   {
     grant_type:    client_credentials,
     aud:           "tempSensor4711",
     client_id:     "myclient",
     client_secret: h'7365637265746b6579',
     token_type:    pop,
     alg:           HS256,
     profile:       coap_dtls
   }
~~~~~~~~~~
{: #authorization-message-example title="Access Token request example"}

The example shows an Access Token request for the resource identified
by the audience string "tempSensor4711" on the AS and the client
identifier "myclient". The symmetric key used for a
proof-of-possession token {{I-D.ietf-oauth-pop-architecture}} is
"secretkey".

TODO: Add example for encrypted shared secrets.

## AS-to-Client Response

When AS has received an Access Token request it has to evaluate
the access request information contained therein.
To grant access to the requested resource, AS constructs an
Access Token as specified in {{I-D.ietf-ace-oauth-authz}}. For use
with this profile, the attribute `profile` is set to `coap_dtls`.

Depending on the requested token type and algorithm in the Access
Token request, AS adds the following Client Information to the
response:

A newly generated session key. This specification describes a method
for AS to derive a session key from a shared secret with RS, and
attributes from the Access Token request such as the `aud` and
`client_secret` parameters. The generated key is transferred as
parameter `k` in a `COSE_Key` object. (See
[Section 7 of cose-msg](https://tools.ietf.org/html/draft-ietf-cose-msg-18#section-7).)

AS SHOULD set Max-Age according to the Access Token lifetime in its
response.

{{at-response}} shows an example AS response containing a new Access Token.
The information is transferred as a CBOR data structure as
specified in {{I-D.ietf-ace-oauth-authz}}. The Max-Age option tells the
receiving Client how long this token will be valid.

<!-- msg1 -->

~~~~~~~~~~
   2.01 Created
   Content-Format: application/cbor
   Location-Path: /token/asdjbaskd
   Max-Age: 86400
   {
      access_token: b64'SlAV32hkKG ...
      (remainder of CWT omitted for brevity;
      token_type:   pop,
      alg:          HS256,
      expires_in:   86400,
      profile:      coap_dtls,
      cnf: {
        COSE_Key: {
          kty: symmetric,
          k: h'73657373696f6e6b6579'
        }
      }
   }
~~~~~~~~~~
{: #at-response title="Example Access Token response"}

A response that declines any operation on the requested
resource is constructed according to [Section 5.2 of RFC 6749](https://tools.ietf.org/html/rfc6749#section-5.2), (cf. Section 6.3 of {{I-D.ietf-ace-oauth-authz}}).

~~~~~~~~~~
    4.00 Bad Request
    Content-Format: application/cbor
    {
      error: invalid_request
    }
~~~~~~~~~~
{: #token-reject title="Example Access Token response with reject"}

## DTLS Channel Setup Between C and RS {#dtls-channel}

When C receives an Access Token from AS, it checks if the payload
contains an `access_token` field and a `cnf` object. With this information C can
initiate establishment of a new DTLS
channel with RS. To use DTLS with pre-shared keys, C follows the PSK
key exchange algorithm specified in Section 2 of {{RFC4279}}, with the
following additional requirements:

1. C sets the psk_identity field of the ClientKeyExchange message
   to the contents of the `access_token` field received from the AS.
1. C uses the key conveyed in the `cnf` field of the AS response as PSK when constructing the premaster
   secret.

Note1: As RS cannot provide C with a meaningful PSK identity hint in
: response to C's ClientHello message, RS SHOULD NOT send a
: ServerKeyExchange message.

Note2: According to {{RFC7252}}, CoAP implementations MUST
: support the ciphersuite TLS\_PSK\_WITH\_AES\_128\_CCM\_8
: {{RFC6655}}. C is therefore expected to offer at least this
: ciphersuite to RS.

Note3: The session key is constructed by AS such that RS can derive the
PSK from the contents of the psk_identity contains in C's ClientHello
message (refer to
{{key-generation}} for details).

## Authorized Communication

Once a DTLS channel has been established as described in {{dtls-channel}}
C is authorized to access resources covered by the Access Token it has
presented in the `psk_identity`.

On the server side (i.e., RS), successful establishment of the DTLS
channel between C and RS ties the
authorization information contained in the `psk_identity` field to this
channel. Any request that RS receives on this channel is checked
against these authorization rules. Incoming CoAP requests that are not
authorized with respect to this Access Token MUST be rejected by RS with 4.01
response as described in {{rreq}}.

RS SHOULD treat an incoming CoAP request as authorized
if the following holds:

1. The message was received on a secure channel that has been
   established using the procedure defined in {{dtls-channel}}.
1. The authorization information tied to the secure channel is valid.
1. The request is destined for RS.
1. The resource URI specified in the request is covered by the
   authorization information.
1. The request method is an authorized action on the resource with
   respect to the authorization information.

Note that the authorization information is not restricted to a single
resource URI. For example, role-based authorization can be used to
authorize a collection of semantically connected resources
simultaneously. Implicit authorization also provides access rights
to authenticated clients for all actions on all resources that RS
offers. As a result, C can use the same DTLS channel not only
for subsequent requests for the same resource (e.g. for block-wise
transfer as defined in {{RFC7959}} or refreshing
observe-relationships {{RFC7641}}) but also for requests
to distinct resources.

Incoming CoAP requests received on a secure channel according to the
procedure defined in {{dtls-channel}} MUST be rejected

1. with response code 4.03 (Forbidden) when the resource URI specified
   in the request is not covered by the authorization information, and
1. with response code 4.05 (Method Not Allowed) when the resource URI
   specified in the request covered by the authorization information but
   not the requested action.

C cannot always know a priori if a Authorized Resource Request
will succeed. If C repeatedly gets AS Information messages (cf. {{as-info}}) as response
to its requests, it SHOULD request a new Access Token from AS to
continue communication with RS.

## Dynamic Update of Authorization Information {#update}

Once a security association exists between a Client and a Resource
Server, the Client can update the authorization information stored at
RS at any time. To do so, the Client requests a new Access Token
for the intended action on the respective resource and
from AS as described in
{{access-request}}.

Note:
: Requesting a new Access Token also can be a Client's reaction on a
  4.03 or 4.05 error that it has received in response to a
  request over a DTLS channel that was setup as specified in {{dtls-channel}}.

{{update-overview}} depicts the message flow where C requests a new
Access Token after a security association between C and RS has been
established using this protocol.

~~~~~~~~~~~~~~~~~~~~~~~

   C                            RS                   AS
   | <== DTLS channel + AT ===> |                     |
   |                            |                     |
   | --- Resource Request ----> |                     |
   |                            |                     |
   | <-- 4.0x + AS Information  |                     |
   |                            |                     |
   | --- Token Request  ----------------------------> |
   |                            |                     |
   | <---------------------------- New Access Token - |
   |                             + Client Information |
   |                            |                     |
   | <== renegotiate session => |                     |
   |     + New Access Token     |                     |
   |                            |                     |
   | == Authorized Request ===> |                     |
   |                            |                     |
   | <=== Protected Resource == |                     |

~~~~~~~~~~~~~~~~~~~~~~~
{: #update-overview title="Overview of Dynamic Update Operation"}

The major difference between dynamic update of authorization
information and the initial handshake is that the DTLS session
between C and RS may be renegotiated with the new Access Token
as described in {{ticket-handle}}.

### Handling of Ticket Transfer Messages {#ticket-handle}

If the security association with RS still exists and RS
has indicated support for session renegotiation according to
{{RFC5746}}, the new Access Token SHOULD be used to renegotiate the
existing DTLS session. In this case, the Access Token is used as
`psk_identity` as defined in {{dtls-channel}}. Otherwise, the Client
MUST perform a new DTLS handshake according to {{dtls-channel}} that
replaces the existing DTLS session.

After successful completion of the DTLS handshake RS updates the
existing authorization information for C according to the
new Access Token.

Note:
: No mutual authentication between C and RS is required for dynamic
  updates when a DTLS channel exists that has been established as
  defined in {{dtls-channel}}. RS only needs to verify the
  authenticity and integrity of the Access Token issued by AS which is
  achieved by having performed a successful DTLS handshake with the
  Access Token as psk_identity. This could even be done within the
  existing DTLS session by tunneling a CoDTLS
  {{I-D.schmertmann-dice-codtls}} handshake.

# DTLS PSK Generation Methods {#key-generation}

One goal of this profile is to provide for a DTLS PSK shared between C and RS. AS and RS MUST negotiate the method for the DTLS PSK generation.

## DTLS PSK Transfer {#key-transfer}

The DTLS PSK is generated by AS and transmitted to C and RS using a secure channel.

The DTLS PSK transfer method is defined as follows:

 * AS generates the DTLS PSK using an algorithm of its choice
 * AS MUST include a representation of the DTLS PSK in the Access Token and
   encrypt it together with all other information with a key
  K(AS,RS) it shares with RS. How AS and RS exchange
   K(AS,RS) is not in the scope of this document. AS and RS
   MAY use their preshared key as K(AS,RS).
 * AS MUST include a representation of the DTLS PSK in `cnf` field of its response to C.
 * AS must ensure that the DTLS PSK is transferred
   to C using encrypted channels.
 * RS MUST decrypt the session key using K(AS,RS)

## Distributed Key Derivation {#key-derivation}

AS generates a DTLS PSK for C which is transmitted using a secure channel. RS generates its own version of the DTLS PSK using the information provided in the `psk_identity` parameter of the ClientHello request.

The distributed key derivation method is defined as follows:

 * AS and RS both generate the DTLS PSK using the information
   included in the Access Token. They use a key derivation algorithm on the Access Token with a shared
   key K(AS,RS). The result serves as the DTLS PSK. How AS and RS
   exchange K(AS,RS) is not in the scope of this document. They MAY
   use their preshared key as K(AS,RS). (TODO: Negotiation of the
   key derivation algorithm between AS and RS.)
 * AS MUST include a representation of the DTLS PSK in the `cnf` field
   in its response to C which MUST be sent over a secure channel.
 * AS MUST NOT include a representation of the DTLS PSK in the Access Token.
 * (TBD) AS MUST NOT encrypt the Access Token.

# Examples

TODO

# Security Considerations

TODO

Initially, no secure channel exists to protect the communication
between C and RS. Thus, C cannot determine if the AS information
contained in an unprotected response from RS to an unauthorized
request (c.f. {#as-info}) is authentic. It is therefore advisable to
provide C with a (possibly hard-coded) list of trustworthy
authorization servers. AS information responses referring to a URI not
listed there would be ignored.


--- back

<!--  LocalWords:  Datagram CoAP CoRE DTLS introducer URI
 -->
<!--  LocalWords:  namespace Verifier JSON timestamp timestamps PSK
 -->
<!--  LocalWords:  decrypt UTC decrypted whitespace preshared HMAC
-->

<!-- Local Variables: -->
<!-- coding: utf-8 -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->