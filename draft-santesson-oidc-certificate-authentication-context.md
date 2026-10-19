---
title: "OIDC Certificate Authentication Context"
abbrev: "OIDC Cert Auth Context"
ipr: trust200902
category: std
updates: 7773

docname: draft-santesson-oidc-certificate-authentication-context-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "LAMPS Working Group"
keyword:
 - certificate
 - oidc
 - authentication
 - X.509
venue:
  group: "LAMPS"
  type: "Working Group"
  mail: "spasm@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/spasm/"
  github: "swedenconnect/ietf-rfc7773-oidc-auth-context"
  latest: "https://swedenconnect.github.io/ietf-rfc7773-oidc-auth-context/draft-santesson-oidc-certificate-authentication-context.html"

author:
 -
    ins: S. Santesson
    name: Stefan Santesson
    org: IDsec Solutions AB
    abbrev: IDsec Solutions
    street: Forskningsbyn Ideon
    city: Lund
    code: "223 70"
    country: SE
    email: sts@aaa-sec.com
 -
    ins: M. Lindstrom
    name: Martin Lindstrom
    org: IDsec Solutions AB
    abbrev: IDsec Solutions
    street: Forskningsbyn Ideon
    city: Lund
    code: "223 70"
    country: SE
    email: martin@litsec.se

normative:
  RFC3339:
  RFC3553:
  RFC5280:
  RFC6901:
  RFC7493:
  RFC7773:
  RFC8259:
  OIDC.Core:
    title: "OpenID Connect Core 1.0 incorporating errata set 2"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore
    date: 2023-12
    target: https://openid.net/specs/openid-connect-core-1_0.html

informative:
  RFC7519:
  RFC8176:
  JSON-SCHEMA:
    title: "JSON Schema: A Media Type for Describing JSON Documents"
    author:
      - ins: A. Wright
      - ins: H. Andrews
      - ins: B. Hutton
      - ins: G. Dennis
    date: 2022-06
    target: https://json-schema.org/draft/2020-12/json-schema-core

...

--- abstract

RFC 7773 defines the Authentication Context certificate extension, which
carries information about how the subject of an X.509 certificate was
authenticated when the certificate was issued. RFC 7773 defines one
context information type, for subjects authenticated by means of a SAML
assertion. This document defines a second context information type for
subjects authenticated by means of an OpenID Connect ID Token. The
information is expressed as a JSON object and provides the OpenID Connect
counterpart to the SAML Authentication Context Information defined in
RFC 7773.

This document also updates RFC 7773 by clarifying that an application
processing the extension ignores context information types it does not
support regardless of the criticality of the extension, and by recommending
that the extension not be marked critical.

--- middle

# Introduction

The Authentication Context certificate extension {{RFC7773}} allows a
Certification Authority (CA) to record, in an issued certificate,
information about the authentication event that preceded issuance. This is
useful when the CA relies on an external identity provider to authenticate
the subject, for example when certificates are issued on the fly to support
electronic signatures, and relying parties need to understand the origin and
assurance of the identity data placed in the certificate.

The extension is a sequence of `AuthenticationContext` structures, each
identified by a `contextType` URI that also determines the syntax of the
accompanying `contextInfo` string. {{RFC7773}} defines one context type,
SAML Authentication Context Information (SACI), for subjects authenticated
by a SAML assertion, and explicitly provides for other documents to define
additional context types.

OpenID Connect {{OIDC.Core}} has become a widely deployed alternative to
SAML for federated authentication. An OpenID Connect authentication results
in an ID Token, a JSON Web Token {{RFC7519}} containing claims about the
authentication event and the authenticated End-User. This document defines
OpenID Connect Authentication Context Information (OACI), a context type
whose `contextInfo` is a JSON object {{RFC8259}} carrying the information
from an ID Token that is relevant to the issuance of a certificate.

OACI is deliberately structured as the OpenID Connect analogue of SACI. It
records the issuer of the ID Token, the time of authentication, the
authentication context class(es) asserted, a reference to the ID Token, the
identity under which the CA acted as Relying Party, and a mapping from
ID Token claims to the subject name attributes of the certificate that were
derived from them.

OACI is defined using the extensibility provided by Section 2 of
{{RFC7773}}. With a second context type in existence, the processing rule
in {{RFC7773}} that ties the handling of unsupported context types to the
criticality of the extension becomes an obstacle to interoperability: an
application that implements only SACI would be required to reject a
certificate carrying OACI in a critical extension, although the two convey
the same kind of information. This document therefore also updates
{{RFC7773}} as described in {{updates-7773}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms OpenID Provider (OP), Relying Party (RP),
End-User, ID Token and Claim as defined in {{OIDC.Core}}. The CA that issues
the certificate is the RP of the OpenID Connect authentication.

The terms `contextType`, `contextInfo` and `AuthenticationContext` refer to
the ASN.1 components defined in Section 2 of {{RFC7773}}.

# OpenID Connect Authentication Context Information {#oaci}

## Context Type Identifier {#context-type}

An `AuthenticationContext` carrying OpenID Connect Authentication Context
Information in accordance with this specification MUST have a `contextType` 
with the following URI value:

~~~
urn:ietf:params:x509-auth-context:oidc
~~~

The `contextInfo` component of such an `AuthenticationContext` MUST be
present and MUST contain a JSON object as specified in {{json-structure}}.

## Encoding {#encoding}

The `contextInfo` value is a `UTF8String` {{RFC7773}}. Its content MUST be
a single JSON text {{RFC8259}} whose top-level value is an object. The JSON
text MUST conform to the I-JSON message format {{RFC7493}}. In particular,
it MUST be encoded in UTF-8, MUST NOT begin with a byte order mark, and
MUST NOT contain duplicate member names.

Producers SHOULD serialize the object without insignificant whitespace in
order to limit certificate size. Consumers MUST accept any serialization
that is valid I-JSON.

Consumers MUST ignore members of the top-level object, and of the objects in
the `claim_mappings` array, that they do not recognize. This allows future
documents and deployment profiles to add members without breaking existing
implementations.

## JSON Structure {#json-structure}

The top-level object contains the members defined below. The member names
are chosen for readability by a reader of the certificate; where a member
corresponds to an ID Token claim, the correspondence is stated. Unless
otherwise specified, a member is OPTIONAL. This document places no
requirement on the presence of any member whose source value is itself
optional in {{OIDC.Core}}. Deployment profiles of this document MAY require
the presence of specific members, and relying parties decide which members
they require in order to accept a certificate.

iss:
: REQUIRED. String. The Issuer Identifier of the OpenID Provider that
  authenticated the End-User, as given by the `iss` claim of the ID Token.
  This is a URL using the https scheme with no query or fragment component,
  as specified in Section 2 of {{OIDC.Core}}. This member corresponds to
  the `IdentityProvider` attribute of SACI.

auth_time:
: String. The time at which the End-User authentication occurred, taken
  from the `auth_time` claim of the ID Token. The value is expressed as a
  date-time string according to {{RFC3339}} in UTC, using the "Z" time zone
  designator and without fractional seconds, e.g.
  `2026-10-19T13:04:05Z`. If the ID Token contains no `auth_time` claim,
  this member is absent. This member corresponds to the
  `AuthenticationInstant` attribute of SACI.

acr:
: Array of strings. The Authentication Context Class Reference values
  satisfied by the authentication. The array MUST contain at least one
  element when present. In most cases the array contains exactly one
  element, the value of the `acr` claim of the ID Token. A CA MAY list more
  than one value when it has determined that the performed authentication
  satisfies more than one authentication context class; how such a
  determination is made is out of scope for this document. The array
  carries no ordering semantics. This member corresponds to the
  `AuthnContextClassRef` attribute of SACI.

amr:
: Array of strings. The Authentication Methods References of the
  authentication, as given by the `amr` claim of the ID Token. Values are
  as registered in the "Authentication Method Reference Values" registry
  established by {{RFC8176}}, or values defined by agreement between the OP
  and the RP. The array MUST contain at least one element when present.
  This member has no counterpart in SACI. See {{sec-amr}} for the
  intended use of this member.

jti:
: String. A unique identifier of the ID Token, as given by the `jti` claim
  of the ID Token. This member corresponds to the `AssertionRef` attribute
  of SACI. Note that `jti` is OPTIONAL in an ID Token.

client_id:
: String. The OAuth 2.0 client identifier under which the CA acted as
  Relying Party towards the OpenID Provider. This value appears as an
  audience of the ID Token (the `aud` claim) and, where present, as the
  `azp` claim. This member corresponds to the `ServiceID` attribute of
  SACI.

claim_mappings:
: Array of objects. Each object describes how one claim about the End-User,
  obtained from the ID Token or from the UserInfo endpoint of the OP, was
  used to populate a subject name attribute in the certificate. The
  structure of each object is defined in {{claim-mapping}}. This member
  corresponds to the `IdAttributes` element of SACI. The array MUST contain
  at least one element when present.

## Claim Mapping {#claim-mapping}

A claim mapping object records that the value of a certificate subject name
attribute was derived from an End-User claim. In contrast to SACI, the claim
value itself is not repeated in the mapping; the value is present in the
certificate attribute to which the mapping refers.

Each claim mapping object contains the following members:

type:
: REQUIRED. String. One of the values `rdn`, `san` or `sda`, with the
  meaning defined for the `Type` attribute in Section 3.1.2 of
  {{RFC7773}}:

  * `rdn` - an attribute in a Relative Distinguished Name of the
    certificate `subject` field.

  * `san` - a name in the Subject Alternative Name extension.

  * `sda` - an attribute in the Subject Directory Attributes extension.

ref:
: REQUIRED. String. Identifies the certificate attribute or name form,
  with the same content rules as the `Ref` attribute in Section 3.1.2 of
  {{RFC7773}}:

  * For type `rdn` and `sda`, the object identifier of the attribute type,
    in dotted-decimal notation (a sequence of decimal integers separated by
    periods, with no other characters).

  * For type `san`, the decimal tag number of the `GeneralName` choice,
    e.g. `1` for `rfc822Name` and `2` for `dNSName`. For the `otherName`
    choice, the object identifier of the `type-id` in dotted-decimal
    notation.

claim:
: REQUIRED. String. A JSON Pointer {{RFC6901}} identifying the claim within
  the set of claims about the End-User, where the root of the pointer is
  the ID Token claims set (or the UserInfo response object, see `source`).
  For a standard top-level claim this is simply `/` followed by the claim
  name, e.g. `/given_name`. The pointer form allows reference to members of
  structured claims, e.g. `/address/street_address`, and to claims in
  aggregated or distributed claim sets.

source:
: String. Indicates where the claim was obtained. The value `id_token`
  indicates the ID Token; the value `userinfo` indicates the UserInfo
  endpoint response. When absent, the default value is `id_token`.

A single certificate attribute MAY be derived from more than one claim, in
which case several claim mapping objects share the same `type` and `ref`
values. A single claim MAY be mapped to more than one certificate
attribute.

## Schema {#schema}

The following schema, expressed in YAML using the vocabulary of
{{JSON-SCHEMA}}, is a normative statement of the structure defined in
{{json-structure}} and {{claim-mapping}}. In case of conflict between the
prose and the schema, the prose takes precedence.

~~~ yaml
$schema: "https://json-schema.org/draft/2020-12/schema"
$id: "urn:ietf:params:x509-auth-context:oidc"
title: OpenID Connect Authentication Context Information
type: object
required: [iss]
properties:
  iss:
    type: string
    format: uri
  auth_time:
    type: string
    format: date-time
    pattern: "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$"
  acr:
    type: array
    minItems: 1
    items:
      type: string
  amr:
    type: array
    minItems: 1
    items:
      type: string
  jti:
    type: string
  client_id:
    type: string
  claim_mappings:
    type: array
    minItems: 1
    items:
      $ref: "#/$defs/claimMapping"
$defs:
  claimMapping:
    type: object
    required: [type, ref, claim]
    properties:
      type:
        type: string
        enum: [rdn, san, sda]
      ref:
        type: string
        pattern: "^[0-9]+(\\.[0-9]+)*$"
      claim:
        type: string
        format: json-pointer
      source:
        type: string
        enum: [id_token, userinfo]
        default: id_token
~~~

Unrecognized members are permitted everywhere; the schema intentionally
does not set `additionalProperties` to `false`.

## Relationship to the ID Token {#relationship}

The OACI object is produced by the CA from an ID Token (and optionally a
UserInfo response) that the CA has validated according to Section 3.1.3.7
of {{OIDC.Core}}. The OACI object is not itself a token: it carries no
signature and no `exp` or `iat` members, because its integrity and
lifetime are those of the certificate in which it is embedded. A relying
party that wishes to inspect the original ID Token may use the `iss`,
`client_id` and `jti` members to locate it, if the CA retains tokens
according to its practices.

# Updates to RFC 7773 {#updates-7773}

## Processing of Unsupported Context Types {#processing}

Section 2 of {{RFC7773}} states that an application that finds an
authentication context information type it does not understand MUST ignore
it if the extension is non-critical and MUST reject the certificate if the
extension is marked critical.

This rule conflates two separate matters: whether an application recognizes
the Authentication Context extension, which is what the criticality flag of
{{RFC5280}} governs, and how an application that does recognize the
extension treats individual `AuthenticationContext` entries. The
information carried by the extension is descriptive. It allows a relying
party that cares about the origin of the subject's identity to evaluate it
against its own policy, but it requires no action from a relying party that
does not. Requiring every relying party to support every context type
present in a certificate therefore serves no security purpose, and it
prevents a CA from including several context types in one certificate, for
example to serve both legacy applications and new applications that support
different versions of the same information.

Accordingly, the rule is replaced by the following.

An application that recognizes the Authentication Context extension MUST be
able to parse the `AuthenticationContexts` structure. It processes each
`AuthenticationContext` entry whose `contextType` it supports according to
the specification of that context type, and MUST ignore each
`AuthenticationContext` entry whose `contextType` it does not support. This
applies irrespective of whether the extension is marked critical.

The criticality flag of the extension retains only its meaning under
Section 4.2 of {{RFC5280}}: an application that does not recognize the
Authentication Context extension at all MUST reject a certificate in which
the extension is marked critical.

## Criticality {#criticality}

Section 2 of {{RFC7773}} states that the extension MAY be marked critical.

Since the extension carries descriptive information and mandates no action
by a relying party, marking it critical excludes relying parties that do
not implement the extension without providing any corresponding benefit.
This document updates {{RFC7773}} as follows: the Authentication Context
extension SHOULD NOT be marked critical.

# Security Considerations

The security considerations of {{RFC7773}} apply.

## Trust in the Recorded Information

The information in an OACI object is asserted by the CA, not by the OpenID
Provider. The CA copies values from an ID Token that it validated at the
time of issuance; a relying party has no means of re-validating the ID
Token from the certificate alone. A relying party therefore trusts the OACI
content to exactly the extent that it trusts the CA, as expressed in the
CA's certificate policy and practices.

## Authentication Context Class References

The values in the `acr` array are only meaningful within the trust
framework that defines them. A relying party MUST NOT infer an assurance
level from an `acr` value defined by a framework it does not recognize.
When a CA lists more than one `acr` value, it asserts that the authentication
satisfied each of them; a CA MUST NOT list a value it has not verified to be
satisfied, since a relying party will typically accept a certificate if any
one listed value meets its policy.

## Authentication Methods References {#sec-amr}

The `amr` member is informative. As noted in Section 4 of {{RFC8176}}, `amr`
values are asserted by the OP without any standardized means of
verification and are not a substitute for an authentication context class.
A CA or relying party SHOULD NOT base an assurance decision solely on `amr`
values. The member is provided so that the concrete methods used can be
recorded for audit and troubleshooting, and because `amr` values, unlike
`acr` values, are drawn from a public registry and remain interpretable
outside the originating trust framework.

## Privacy

An OACI object reveals the identity of the OP that authenticated the
subject, the time of authentication and, through `claim_mappings`, which
identity attributes in the certificate were asserted by that OP. Together
with the certificate serial number and `jti`, this allows a certificate to
be correlated with an authentication event logged at the OP and the CA.
CAs SHOULD include only members needed by the intended relying parties, and
deployment profiles SHOULD consider whether `jti` and `auth_time` are
required.

This document intentionally does not define a member for the `sub` claim or
any other End-User identifier. The claim values that were placed in the
certificate are present in the certificate subject name; repeating them, or
adding further identifiers, would increase the personal data carried by the
certificate without adding information a relying party can verify.

## Ignoring Unsupported Context Types

Under {{processing}}, an application silently ignores context types it does
not support. A relying party whose policy requires authentication context
information of a particular type must therefore check for the presence of
that type explicitly and treat its absence according to policy; the absence
of a supported entry is not an error condition in itself.

# IANA Considerations {#iana}

## X.509 Authentication Context Types Registry {#iana-registry}

IANA is requested to create a new registry titled "X.509 Authentication
Context Types" under a new "X.509 Authentication Context" registry group.
The registry lists the URIs defined for use as `contextType` values in the
Authentication Context certificate extension {{RFC7773}}, together with the
specification that defines the syntax of the corresponding `contextInfo`.

The registration policy is Specification Required (see {{!RFC8126}}).
Each entry contains:

* Context Type URI: the URI used as the `contextType` value.
* Description: a short description of the context information type.
* Reference: the specification defining the `contextInfo` syntax.

Context type URIs are not required to be assigned under the URN
sub-namespace registered in {{iana-urn}}; any URI under the control of the
registrant is acceptable. Registrations under the URN sub-namespace are
expected to be made by IETF documents.

The initial content of the registry is:

| Context Type URI | Description | Reference |
|------------------|-------------|-----------|
| http://id.elegnamnden.se/auth-cont/1.0/saci | SAML Authentication Context Information | {{RFC7773}}, Section 3 |
| urn:ietf:params:x509-auth-context:oidc | OpenID Connect Authentication Context Information | This document, {{oaci}} |
{: title="Initial registry content"}

## IETF URN Sub-namespace Registration {#iana-urn}

IANA is requested to register the following entry in the "IETF URN
Sub-namespace for Registered Protocol Parameter Identifiers" registry
established by {{RFC3553}}:

Registry name:
: x509-auth-context

Specification:
: This document.

Repository:
: The "X.509 Authentication Context Types" registry ({{iana-registry}}).

Index value:
: The index value is the string following the prefix
  `urn:ietf:params:x509-auth-context:`. Values are registered as complete
  URIs in the "X.509 Authentication Context Types" registry; the entries
  of that registry beginning with the prefix constitute the values of this
  sub-namespace.

--- back

# Examples

## Example OACI Object

The following OACI object records an authentication performed by the
OpenID Provider `https://op.example.com` at a context class defined by an
example trust framework. The certificate subject name contains the
End-User's given name (OID 2.5.4.42), surname (OID 2.5.4.4) and serial
number (OID 2.5.4.5), and an email address in the Subject Alternative Name
extension, all derived from ID Token claims.

~~~ json
{
  "iss": "https://op.example.com",
  "auth_time": "2026-10-19T13:04:05Z",
  "acr": ["http://id.example.org/loa/3"],
  "amr": ["sc", "pin"],
  "jti": "0f3e2c6a-7b1d-4e4c-9a2e-5d1c2b3a4f60",
  "client_id": "ca.example.com",
  "claim_mappings": [
    { "type": "rdn", "ref": "2.5.4.42", "claim": "/given_name" },
    { "type": "rdn", "ref": "2.5.4.4",  "claim": "/family_name" },
    { "type": "rdn", "ref": "2.5.4.5",
      "claim": "/https:~1~1id.example.org~1claims~1personal_identity_number" },
    { "type": "san", "ref": "1",        "claim": "/email" }
  ]
}
~~~

Note that the third mapping refers to a claim whose name is itself a URI.
Since `/` is the reference token separator in a JSON Pointer, the `/`
characters inside the claim name are escaped as `~1` (see Section 3 of
{{RFC6901}}).

[[ Editor's note: claims with URI names are common in national profiles and
the escaping makes them hard to read. Consider instead a plain claim name in
`claim` with a separate OPTIONAL `path` member (a JSON Pointer relative to
the claim) for members of structured claims. ]]

## Example Extension

The `AuthenticationContext` carrying the above object has
`contextType` set to `urn:ietf:params:x509-auth-context:oidc` and
`contextInfo` set to the compact serialization of the object. A DER
rendering of the complete extension will be added in a future revision.

# Design Notes
{:numbered="false"}

[[ This section is intended to be removed before publication as an RFC. ]]

* `acr` is an array even when only one value is present, so that consumers
  implement one code path.

* Member names follow the ID Token claim names where that aids
  recognition (`iss`, `acr`, `amr`, `jti`, `auth_time`) but the
  structure is not a JWT claims set and is not intended to be processed as
  one; `auth_time` in particular uses RFC 3339 text rather than NumericDate
  for readability in certificate viewers.

* Claim values are not repeated in `claim_mappings`, unlike SACI, because
  they are already present in the certificate and repeating them has not
  proven useful in practice.

* The update to RFC 7773 in {{updates-7773}} is the reason this document
  carries "Updates: 7773". Without it, defining a new context type would
  not update RFC 7773, which already provides for additional types.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
