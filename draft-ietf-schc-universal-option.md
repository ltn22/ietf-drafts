---
v: 3

title: Options representation in SCHC YANG Data Models
abbrev: SCHC for CoAP
docname: draft-ietf-schc-universal-option-02
ipr: trust200902


v3xml2rfc:
  silence:
  - Found SVG with width or height specified

area: Internet
wg: SCHC Working Group
kw: Internet-Draft
cat: std
submissiontype: IETF

coding: utf-8

author:
      - name: Quentin Lampin
        org: Orange
        email: quentin.lampin@orange.com
      - name: Ana Minaburo
        org: Consultant
        street: Rue de Rennes
        city: Cesson-Sevigne
        code: 35510
        country: France
        email: anaminaburo@gmail.com
      - name: Marco Tiloca
        org: RISE AB
        street: Isafjordsgatan 22
        city: Kista
        code: SE-16440
        country: Sweden
        email: marco.tiloca@ri.se
      - name: Laurent Toutain
        org: IMT Atlantique
        street: CS 17607, 2 rue de la Chataigneraie
        city: Cesson-Sevigne Cedex
        code: 35576
        country: France
        email: Laurent.Toutain@imt-atlantique.fr
      - name: Samar Sirohi
        org: Institut MINES TELECOM; IMT Atlantique
        street: 2 rue de la Chataigneraie CS 17607
        city: Cesson-Sevigne Cedex
        code: 35576
        country: France
        email: Samar.Sirohi@imt-atlantique.fr

normative:
  RFC7252:
  RFC7950:
  RFC8724:
  RFC9363:

informative:
  RFC8613:
  I-D.ietf-core-oscore-key-update:
  I-D.ietf-schc-icmpv6-compression:
  I-D.toutain-schc-rfc9363bis:
  RFC7641:
  RFC7959:
  RFC7967:
  RFC8407:
  RFC8824:
  I-D.lampin-lpwan-schc-considerations:
  I-D.ietf-schc-8824-update:
  I-D.ietf-lpwan-architecture:
  GPC-SPE-207: 
    title: "Remote Application Management over CoAP – Amendment M v1.0"
    author:
       org: "GlobalPlatform"
    target: "https://globalplatform.org/specs-library/amendment-m-remote-application-mgmt-over-coap/"

entity:
  SELF: "[RFC-XXXX]"

--- abstract

Keeping option identifiers in SCHC Rules simplifies interoperability
and the evolution of SCHC compression when a protocol introduces
options that the current SCHC implementation does not know.
This document explains why the Field Identifiers (FIDs) defined for
CoAP options in RFC 9363 do not scale: a FID must be allocated,
standardized and implemented for each option a protocol defines,
before any Rule can describe that option.
It presents an evolution of the compression Rule defined in RFC 9363,
in which an entry refers to an option by the pair (option space,
option number), called a Universal Option, and in which entries are
indexed, ordered and can reference one another.
It also presents a data model dedicated to CoAP, which processes
options directly from the option number carried in the message,
instead of mapping each option to a FID of its own.


--- middle

# Introduction # {#intro}

Static Context Header Compression (SCHC) provides an essential mechanism for efficient communication in constrained networks by compressing protocol headers using predefined Rules. Originally developed for Low Power Wide Area Networks (LPWANs), SCHC has proven effective in scenarios with limited bandwidth, power constraints, and predictable traffic patterns.

The YANG Data Model defined in {{RFC9363}} was designed primarily for LPWAN technologies, focusing on highly constrained devices such as sensors and actuators that generate predictable traffic patterns, which allowed for static compression rules. This model also incorporates CoAP parameters defined in {{RFC8824}}, representing CoAP options as specific Field Identifiers (FIDs) within SCHC Rules.

While this approach works well in controlled LPWAN environments, it presents interoperability challenges when SCHC is applied to more dynamic networks or when protocols evolve to incorporate new options. The primary issue arises from the disconnection between protocol option identifiers and SCHC Field Identifiers (FIDs), making it difficult to efficiently handle newly defined or private options without disrupting existing implementations.

This document proposes a more flexible approach to representing protocol options in SCHC YANG Data Models. By preserving original protocol option identifiers within SCHC Rules, we aim to:

* Improve interoperability between SCHC implementations

* Enable more graceful handling of protocol evolution

* Simplify Rule management between SCHC endpoints

* Support compression of newly defined or private options without requiring updates to the SCHC implementation

The following sections examine the current challenges in detail, explore potential solutions including both semantic and syntactic compression approaches, and propose extensions to the YANG Data Model that preserve protocol option identifiers while maintaining efficient compression capabilities.

# Current Challenges


The fundamental challenge in SCHC compression for protocols with options stems from how options are represented in the SCHC Data Model. Unlike the relatively static headers of IPv6 or UDP, CoAP includes a flexible options mechanism that allows for protocol extension through new options.

## Option Representation Problem


In the current SCHC Data Model defined in {{RFC9363}}, CoAP options are represented as predefined Field Identifiers (FIDs) that are included in SCHC Rules. Each option is essentially abstracted away from its original protocol identifier and assigned a new SCHC-specific identifier. While this approach works adequately in static LPWAN environments where the set of options is known in advance and rarely changes, it creates significant challenges in more dynamic networks or when protocols evolve.


The core problem is that the mapping between protocol option identifiers (as defined in the protocol specification) and SCHC FIDs is not standardized or predictable. When a new CoAP option is defined after a SCHC implementation is deployed, there is no straightforward mechanism for the implementation to assign an appropriate FID to this option or to communicate this assignment to other SCHC implementations.


## Rule Management Challenges

This limitation becomes particularly problematic in scenarios involving rule management between two SCHC endpoints. The following scenario (cf. {{fig-rule-mngt}}) illustrates this issue:


~~~~ aasvg
              rule mngt
             +----------+
             |          |
             v          v
A <-------> S1 <~~~~~~> S2 <-----> B
~~~~
{: #fig-rule-mngt title="Rule Management between two SCHC end-points" artwork-align="center"}

In this scenario:

* Device A generates CoAP packets that may include various options, including newly defined ones.

* SCHC nodes S1 and S2 compress and decompress the traffic using Rules they share.

* When A uses a newly defined or private option, S1 can parse this option (as the CoAP header structure remains consistent) and could potentially derive a new Rule to optimize compression.

* However, S1 faces a critical problem: how to identify this new option in the Rule and communicate this identifier to S2 in a way that S2 can understand which option is involved and correctly reconstruct the header.

For example, suppose a Rule defines just a CoAP header, and S1 derives a more specific Rule including a URI-path. The entry shown in {{fig-entry-uri-path}} can be added to the derived Rule, and {{RFC9363}} defines appropriate identityref values (respectively fid:coap-option-uri-path, di:up, mo:equal and cda:not-sent)  that can be used to communicate this Rule description to S2.


~~~~ aasvg
+--------------+-----+---+----+-------+-------+---------+
|    FID       | FL  | FP| DI |  TV   |   MO  |   CDA   |
+==============+=====+===+====+=======+=======+=========+
| ...          | ... |...|... | ...   | ...   | ...     |
|CoAP.Uri-path | len | 1 | up | value | equal | not-sent|
+--------------+-----+---+----+-------+-------+---------+
~~~~
{: #fig-entry-uri-path title="New entry added by management" artwork-align="center"}

However, when A uses a recently defined option or a private option that was not known when the SCHC implementation was created, S1 cannot represent this option using existing FIDs. While S1 can still parse the CoAP header and identify the new option, it has no predefined FID to use in the Rule, as shown in {{fig-new-fid}}, and S2 is unable to find the original option number to reconstruct the header:

~~~~ aasvg
+---------------+-----+---+----+-------+-------+---------+
|     FID       | FL  | FP| DI |  TV   |   MO  |   CDA   |
+===============+=====+===+====+=======+=======+=========+
| ...           | ... |...|... | ...   | ...   | ...     |
|CoAP.new-option| len | 1 | up | value | equal | not-sent|
+---------------+-----+---+----+-------+-------+---------+
~~~~
{: #fig-new-fid title="New entry added by management" artwork-align="center"}


##Interoperability Consequences

This disconnect between protocol option identifiers and SCHC FIDs creates several interoperability issues:

* Limited Protocol Evolution Support: New protocol options cannot be efficiently compressed without updates to the SCHC implementation.

* Incompatible Rule Exchanges: Different SCHC implementations may assign different FIDs to the same new option, making Rule exchange between them problematic.

* Implementation Complexity: SCHC implementations must maintain complex mappings between protocol options and FIDs, and these mappings may differ between implementations.

* Deployment Barriers: Deploying SCHC in environments with evolving protocols becomes difficult and requires frequent updates to SCHC implementations.

The fundamental issue is that the protocol option space and the SCHC FID space are disconnected, with no standardized or dynamic mapping between them. This disconnect seriously limits SCHC's applicability in dynamic environments and its ability to adapt to protocol evolution.

In the following sections, we introduce two approaches to address this challenge for CoAP. The first approach, called 'semantic,' abstracts away from the byte-level representation and introduces a generic option framework that eliminates the need for mapping between option values and Field IDs. This semantic approach focuses on the logical meaning rather than the protocol-specific encoding.
The second approach, termed 'syntactic,' aims to closely align with CoAP's native representation of options by defining three distinct fields: option delta type, option length, and option value. Since many protocols encode their options as a type, a length and a value, this second approach can easily be derived for protocols other than CoAP.

# Semantic compression 

The first solution keeps the efficiency of semantic compression while addressing the interoperability challenges described above. It preserves protocol option identifiers directly within SCHC Rules, eliminating the need for mapping between protocol-specific option numbers and SCHC Field Identifiers (FIDs).

The core idea is to incorporate the original protocol identifiers into the compression Rules. Since multiple protocols may reuse the same option identifier for different purposes (for example, option 8 refers to Location-Path in CoAP but Timestamp in TCP), this approach associates each option value with its protocol namespace to avoid ambiguity.

The solution involves defining within SCHC an identity that references the protocol (creating a "protocol space"), followed by the specific option identifier used by that protocol. This preserves the protocol's native numbering scheme while allowing SCHC to differentiate between options from different protocols that might share the same option identifier.

Using this approach, a Rule that includes various CoAP options would directly reference the CoAP option numbers rather than abstract FIDs. For instance, the representation of a URI with two path elements (option 11) and two query elements (option 15) might look like (cf. {{fig-proto-id}}):

~~~~ aasvg
+---------------+---+--+--+-----+------+----------+
|      FID      |FL |FP|DI| TV  |  MO  |   CDA    |
+===============+===+==+==+=====+======+==========+
|CoAP.option(11)|len|1 |up|value|equal |not-sent  |
|CoAP.option(11)|len|2 |up|     |ignore|value-sent|
|CoAP.option(15)|len|1 |up|value|equal |not-sent  |
|CoAP.option(15)|len|2 |up|value|equal |not-sent  |
+---------------+---+--+--+-----+------+----------+
~~~~
{: #fig-proto-id title="Rule including options ID." artwork-align="center"}

In this example, option identifiers 11 (URI-Path) and 15 (URI-Query) are directly specified, along with their position and direction, providing clear identification of which CoAP options are being compressed without requiring predefined FIDs for each option type.



## Universal Options {#universal-options}

In RFC 9363 {{RFC9363}}, each entry in the compression structure is referenced
by a tuple composed of a Field ID (FID), a Direction Indicator (DI)
and a position. 

This keying does not guarantee that entries appear in the same
order as the corresponding fields in the header (all the options
will appear at the end of the Rule), whereas RFC 8724 {{RFC8724}}
mandates that order and recent implementations have shown it to be
efficient. In addition, these three- or four-element keys remain
large, which may be a penalty when Rule management is used.

RFC 7950 {{RFC7950}}, Section 11, does not allow the "entry" list's
key, nor the "field-id" leaf, to be changed in place in a published
module ("Otherwise, if the semantics of any previous definition are
changed \[...\] then this MUST be achieved by a new definition with a
new identifier"). This document therefore deprecates the whole
compression Rule entry structure
in favor of a new one, providing a uniform way to reference a field
entry. A new
"entry-index" leaf is introduced as the key of the new
"entry-universal" list. The list is defined as "ordered-by user", and
"entry-index" values are assigned sequentially according to the order of
the entries in that list. The list order therefore continues to convey
the header field order (addressing the second
problem).

"entry-index" is then followed by a choice between a field-id and a
space-id/option-number. The rest of the entry is unchanged, except
for the new "field-length-value" leaf, added to carry the argument
of a Field Length function.

This document therefore leaves the RFC 9363
"compression-rule-entry" grouping, its "compression-content"
grouping, and the "compression" case untouched, and marks them
"status deprecated;" (RFC 8407 recommends a "deprecated" status be
kept for at least one year before moving to "obsolete"). The new
structure is defined in a new "compression-rule-entry-universal" grouping,
used by a new "compression-content-universal" grouping and a new
"compression-universal" case, sitting alongside the deprecated ones in the
"nature" choice. The YANG definitions of these groupings are
specified in {{I-D.toutain-schc-rfc9363bis}}.

{{fig-compression-rule-entry}} shows the resulting YANG tree diagram
for both cases (deprecated "compression" and new "compression-universal").

~~~ ascii-art
x--:(compression) {compression}?
|  x--rw entry* [field-id field-position direction-indicator]
|     +--rw field-id                    fid-type
|     +--rw field-length                union
|     +--rw field-position              uint8
|     +--rw direction-indicator         di-type
|     +--rw target-value* [index]
|     |  +--rw index    uint16
|     |  +--rw value?   binary
|     +--rw matching-operator           mo-type
|     +--rw matching-operator-value* [index]
|     |  +--rw index    uint16
|     |  +--rw value?   binary
|     +--rw comp-decomp-action          cda-type
|     +--rw comp-decomp-action-value* [index]
|        +--rw index    uint16
|        +--rw value?   binary
+--:(compression-universal) {compression or management}?
   +--rw entry-universal* [entry-index]
   |  +--rw entry-index                 uint16
   |  +--rw (field-or-space)
   |  |  +--:(regular-field)
   |  |  |  +--rw field-id              fid-type
   |  |  +--:(universal-option)
   |  |     +--rw space-id              space-id-type
   |  |     +--rw universal-value       uint64
   |  +--rw field-length                union
   |  +--rw field-length-value?         uint16
   |  +--rw field-position              uint8
   |  +--rw direction-indicator         di-type
   |  +--rw target-value* [index]
   |  |  +--rw index    uint16
   |  |  +--rw value?   binary
   |  +--rw matching-operator           mo-type
   |  +--rw matching-operator-value* [index]
   |  |  +--rw index    uint16
   |  |  +--rw value?   binary
   |  +--rw comp-decomp-action          cda-type
   |  +--rw comp-decomp-action-value* [index]
   |     +--rw index    uint16
   |     +--rw value?   binary
   +--rw action?                          action-type
   +--rw action-value* [index]
      +--rw index    uint16
      +--rw value?   binary
~~~
{: #fig-compression-rule-entry title="Compression Rule Entry: deprecated (RFC 9363) and new (-universal) cases"}

Unlike the deprecated "entry" list, keyed by the compound
"field-id"/"field-position"/"direction-indicator", "entry-universal"
is keyed by the single-leaf "entry-index". This makes an entry
cheap and unambiguous to reference from elsewhere within the same
Rule, e.g. the "field-length-value" argument of a Field Length
function, or a management operation targeting that entry.

## Space ID

Several protocols define options: CoAP {{RFC7252}} is one example,
but other protocols define their own options too, each with its own
option numbering. An option number is therefore not enough by
itself; it must be qualified by the option space, i.e. the protocol
it belongs to.

To identify these option spaces, this document creates a
"space-id-base-type" identity and derives one identity per protocol
from it; a "space-id-type" typedef is then created from
"space-id-base-type", following the identityref pattern used
throughout RFC 9363 (e.g. for "rcs-algorithm-type"). At present, the
module only defines one such identity, "space-id-coap", for the CoAP
option space; other protocols that define options can add their own
"space-id-*" identity the same way. {{fig-space-id}} shows this
pattern, mirroring how RFC 9363 introduces its own
identityref-derived types.

~~~ yang
identity space-id-base-type {
  base schc:space-field-id-base-type;
  description
    "Base identity for a Universal Option space. Several
     protocols define options (e.g. CoAP); each such protocol
     is identified by an identity derived from this base type.";
}

identity space-id-coap {
  base space-id-base-type;
  description
    "Space ID identifying the CoAP option space.";
  reference
    "RFC 7252 The Constrained Application Protocol (CoAP)";
}

typedef space-id-type {
  type identityref {
    base space-id-base-type;
  }
  description
    "Space ID type for universal option spaces (CoAP options,
     etc.). Used in the universal-option case of
     compression-rule-entry-universal.";
}
~~~
{: #fig-space-id title="Space ID: base identity, one derived identity per protocol, and typedef"}

"space-id-type" is used, together with "space-id-base-type" and
"fid-base-type"'s common ancestor "space-field-id-base-type", by the
"field-or-space" choice (see {{fig-compression-rule-entry}}).

## Deprecating the Per-Option CoAP FIDs

RFC 9363 defines twenty per-option Field IDs deriving from
"fid-coap-option", one for each CoAP option registered at the time
(see {{tbl-coap-option-fids}}). Now that Universal Options provide a
"space-id-coap"/"universal-value" pair to reference any CoAP option
by its option number, without needing a dedicated FID for it, each
of these twenty identities is superseded and marked deprecated.

RFC 7950 {{RFC7950}}, Section 11, states that "Obsolete definitions
MUST NOT be removed from published modules, since their identifiers
may still be referenced by other modules": once RFC 9363 is
published, these twenty identities can only ever be marked
"deprecated" and then "obsolete"; they can never simply disappear
from a later revision of the module. RFC 8407 {{RFC8407}}, Section
4.7, further recommends that "an object SHOULD be available for at
least one year with a 'deprecated' status before it is changed to
'obsolete'", and that "the status SHOULD NOT be changed from
'current' directly to 'obsolete'": since these twenty identities are
"current" in RFC 9363, this document marks them "deprecated" rather
than "obsolete".

These twenty identities MUST NOT be used in a new "entry-universal"
list ({{fig-compression-rule-entry}}): the "space-id"/
"universal-value" pair introduced in {{universal-options}}
supersedes them there. They remain usable only in the deprecated
"entry" list, for Rules that already reference them.

IANA has allocated the SID range 2550-2949 to the "ietf-schc"
module. At the time of writing, no SID file for "ietf-schc" is
registered in the IETF YANG-SID Modules registry. No SID from that
range is to be allocated to these twenty deprecated identities: a
SID "immutably maps to EXACTLY one YANG name", so allocating one to
an identity already superseded by Universal Options would waste it
permanently, with no way to reclaim it later.

The working copy of the module has been corrected accordingly: the
twenty identities are present, each with "status deprecated;" (see
{{fig-fid-coap-option-deprecated}}).

| RFC 9363 FID | CoAP Option | Replaced by (space-id-coap / universal-value) | Defined in |
|---|---|---|---|
| fid-coap-option-if-match | If-Match | 1 | {{RFC7252}} |
| fid-coap-option-uri-host | Uri-Host | 3 | {{RFC7252}} |
| fid-coap-option-etag | ETag | 4 | {{RFC7252}} |
| fid-coap-option-if-none-match | If-None-Match | 5 | {{RFC7252}} |
| fid-coap-option-observe | Observe | 6 | {{RFC7641}} |
| fid-coap-option-uri-port | Uri-Port | 7 | {{RFC7252}} |
| fid-coap-option-location-path | Location-Path | 8 | {{RFC7252}} |
| fid-coap-option-uri-path | Uri-Path | 11 | {{RFC7252}} |
| fid-coap-option-content-format | Content-Format | 12 | {{RFC7252}} |
| fid-coap-option-max-age | Max-Age | 14 | {{RFC7252}} |
| fid-coap-option-uri-query | Uri-Query | 15 | {{RFC7252}} |
| fid-coap-option-accept | Accept | 17 | {{RFC7252}} |
| fid-coap-option-location-query | Location-Query | 20 | {{RFC7252}} |
| fid-coap-option-block2 | Block2 | 23 | {{RFC7959}} |
| fid-coap-option-block1 | Block1 | 27 | {{RFC7959}} |
| fid-coap-option-size2 | Size2 | 28 | {{RFC7959}} |
| fid-coap-option-proxy-uri | Proxy-Uri | 35 | {{RFC7252}} |
| fid-coap-option-proxy-scheme | Proxy-Scheme | 39 | {{RFC7252}} |
| fid-coap-option-size1 | Size1 | 60 | {{RFC7252}} |
| fid-coap-option-no-response | No-Response | 258 | {{RFC7967}} |
{: #tbl-coap-option-fids title="RFC 9363 per-option CoAP FIDs and their Universal Option replacement"}



~~~ yang
identity fid-coap-option-uri-path {
  base fid-coap-option;
  status deprecated;
  description
    "CoAP option Uri-Path. Deprecated in favor of Universal
     Options (space-id-coap / universal-value).";
  reference
    "RFC 7252 The Constrained Application Protocol (CoAP)";
}
~~~
{: #fig-fid-coap-option-deprecated title="Example: Deprecating a Per-Option CoAP FID (fid-coap-option-uri-path)"}

# New field-length functions {#field-length-functions}

RFC 8824 {{RFC8824}} defines a specific length function for the
Token field: when this function is specified, the value of the
Token Length (TKL) is used to indicate the length of the field.
{{I-D.ietf-schc-8824-update}} extends this principle to the Partial IV field
for OSCORE.

This approach is not scalable: a new function would have to be defined
for every new protocol field whose length needs to be carried this way.
This document instead introduces two new functions, "fl-length-bytes"
and "fl-length-bits". They both take an argument, carried in the new
Rule entry field "field-length-value", that points to the entry where
the length is specified, referenced by its "entry-index" (see
{{fig-compression-rule-entry}}). Having two separate functions, rather
than a single generic one, also conveys the unit of the length value:
bytes for "fl-length-bytes", bits for "fl-length-bits".

* "fl-length-bytes" generalizes "fl-token-length": for example,
  "fl-token-length" is now equivalent to "fl-length-bytes(index)",
  where "index" is the "entry-index" of the CoAP TKL field.
* "fl-length-bits" is the bit-level equivalent of "fl-length-bytes".

This document also introduces "fl-remaining", for an entry that is
always the last one in the Rule, such as the "fid-payload" Field ID
of {{I-D.ietf-schc-icmpv6-compression}}.
Unlike "fl-variable", which prefixes its residue with an explicit
length, "fl-remaining" accepts a
field of any length but does not add that prefix when serializing
the residue: being the last entry, its residue already runs to the
end of the compression residue and needs no delimiter.


# Syntactic compression


SCHC compression typically uses a semantic approach where protocol fields are abstracted into generic representations with Field IDs (FIDs) that don't directly correlate to the protocol's encoding format. While effective for static fields, this creates challenges for dynamic protocol options as previously discussed.

Syntactic compression, as proposed in {{I-D.lampin-lpwan-schc-considerations}}, takes a fundamentally different approach by aligning compression more closely with the protocol's wire format. Instead of abstracting away from the protocol's representation, syntactic compression preserves the original structure of protocol headers, including how options are encoded.

Staying close to the wire format also affects processing performance. The compressor walks the options exactly as they appear in the message, so parsing and compression can be carried out in a single pass, without building an intermediate representation of the header and without looking up a mapping from option numbers to Field IDs. The same code path serves every option, known or not, which keeps the implementation small and its processing cost independent of the number of options the protocol defines. This gain has to be weighed against the compression efficiency examined below.

To understand syntactic compression for CoAP options, it's important to recall how CoAP encodes options in messages:

* Each CoAP option on the wire consists of three components:

  * Option Delta: The difference between the option number of the present option and the option number of the previous option (if any).

  * Option Length: The length of the option value in bytes.

  * Option Value: The actual option content.


* This encoding allows CoAP to efficiently represent options while maintaining extensibility.


In the syntactic approach, instead of representing a CoAP option as a single abstract Field ID, each option is decomposed into its three constituent parts, carried by three consecutive entries "n", "n+1" and "n+2" of the Rule, as shown in {{fig-synt-not-sent}}, where the Field Length of the CoAP.value entry is obtained from the entry "n+1" holding the option length, through the "fl-length-bytes" function:

~~~~ aasvg
+-----+------------+------------+---+----+-------+-------+---------+
|  #  |     FID    |     FL     | FP| DI |  TV   |   MO  |   CDA   |
+=====+============+============+===+====+=======+=======+=========+
| n   |CoAP.option | 16         | 1 | up | delta | equal | not-sent|
| n+1 |CoAP.length | 16         | 1 | up | -     |ignore |value-   |
|     |            |            |   |    |       |       |sent     |
| n+2 |CoAP.value  | length-    | 1 | up | -     |ignore |value-   |
|     |            | bytes(n+1) |   |    |       |       |sent     |
+-----+------------+------------+---+----+-------+-------+---------+
~~~~
{: #fig-synt-not-sent title="representation of an elided option with syntactic representation" artwork-align="center"}

In this representation:

* 'CoAP.option' the delta value as encoded in the CoAP message.

* "CoAP.length" represents the option length field.

* "CoAP.value" contains the actual option value, whose length is
  given by the "CoAP.length" entry.

# Fields of the OSCORE Option for OSCORE and KUDOS {#oscore-kudos-fids}

Universal Options allow any option to be compressed following the
SCHC principle, but compression can be more efficient when the
sub-fields of an option are taken into account individually. This is
why the module introduces specific fields of the OSCORE Option,
separately for OSCORE {{RFC8613}} and
KUDOS {{I-D.ietf-core-oscore-key-update}}: the OSCORE Option value is
split into sub-fields, each with its own identity
("fid-coap-option-oscore-piv", "fid-coap-option-oscore-kid",
"fid-coap-option-oscore-kidctx", "fid-coap-option-kudos-nonce",
etc.).

Note that flag-related fields, such as the OSCORE flag bits for OSCORE
and the x byte for KUDOS, are also distinct and split so that its
length indicator becomes a specific field of its own:
"fid-coap-option-oscore-flags-flagbits"/"fid-coap-option-oscore-flags-n"
for OSCORE, and "fid-coap-option-kudos-x-flagbits"/
"fid-coap-option-kudos-x-m" for KUDOS. This will be exploited by the
new Field Length functions introduced in {{field-length-functions}},
which can reference such a field, by its "entry-index", to determine
the length of another field (e.g. the Partial IV).

KUDOS's "fid-coap-option-kudos-x-m" cannot be treated in the same
way as OSCORE's "fid-coap-option-oscore-flags-n" for this purpose.
The value "m" encodes the Nonce length in bytes minus one; equivalently,
the Nonce length is "m + 1". Since "fl-length-bytes" and
"fl-length-bits" directly use the value of the referenced entry as the
length, they cannot directly represent this relationship. KUDOS
therefore requires separate handling of this length transformation.

"fid-coap-option" itself is kept "current": it is still used,
unchanged, as the base identity of these OSCORE-related and
KUDOS-related sub-field FIDs. Unlike the twenty per-option FIDs deprecated in the previous
subsection, they use "fid-coap-option" as a typing hierarchy for
parts of a single CoAP option (OSCORE), not as a stand-in for
an arbitrary CoAP option, and are therefore unaffected: they remain
"current".

## Field Length Functions {#oscore-field-length}

This is what {{oscore-kudos-fids}} relies on: in the OSCORE Option
value, the flag-related fields for OSCORE and KUDOS are distinct and
split, so that its length indicator
("fid-coap-option-oscore-flags-n", "fid-coap-option-kudos-x-m") is
its own entry; "fl-length-bytes" or "fl-length-bits" can then use
that entry's "entry-index" to derive the length of the corresponding
Partial IV field. This works directly for OSCORE;
{{oscore-kudos-fids}} explains why the same does not hold for
KUDOS's Nonce.

## Example 1: OSCORE outer header

{{fig-example-rule}} shows an example SCHC Compression Rule (Rule ID
2, encoded on 5 bits, i.e. "00010") that compresses, in both
directions, the outer CoAP header of a CoAP/OSCORE message.

The server is located on the Application side and the client on the
Device side. In OSCORE, the client's request carries an OSCORE
option with the security parameters, while the server's response
carries an empty OSCORE option; the Token links the response to its
request.

The first column is new and not defined in RFC 8724 {{RFC8724}}: it
numbers the entries in the rule. The names in the other columns are
the ones defined in the YANG module, without their prefix ("fid-",
"fl-", "mo-", "cda-"), for better legibility.

~~~ aasvg
+----+----------------+----------+----+----+----+--------+----------+
| #  | Field          | FL       | FP | DI | TV | MO     | CDA      |
+----+----------------+----------+----+----+----+--------+----------+
| 0  | coap-version   | 2        | 1  | Bi | 01 | equal  | not-sent |
| 1  | coap-type      | 2        | 1  | Bi | -  | ignore | value-   |
|    |                |          |    |    |    |        | sent     |
| 2  | coap-tkl       | 4        | 1  | Bi | -  | ignore | value-   |
|    |                |          |    |    |    |        | sent     |
| 3  | coap-code      | 8        | 1  | Up | 02 | equal  | not-sent |
| 4  | coap-code      | 8        | 1  | Dw | 44 | equal  | not-sent |
| 5  | coap-mid       | 16       | 1  | Bi | 00 | ignore | value-   |
|    |                |          |    |    |    |        | sent     |
| 6  | coap-token     | length-  | 1  | Bi | -  | ignore | value-   |
|    |                | bytes(2) |    |    |    |        | sent     |
| 7  | coap-option-   | 5        | 1  | Up | 01 | equal  | not-sent |
|    | oscore-flags-  |          |    |    |    |        |          |
|    | flagbits       |          |    |    |    |        |          |
| 8  | coap-option-   | 3        | 1  | Up | 01 | equal  | not-sent |
|    | oscore-flags-n |          |    |    |    |        |          |
| 9  | coap-option-   | length-  | 1  | Up | -  | ignore | value-   |
|    | oscore-piv     | bytes(8) |    |    |    |        | sent     |
| 10 | coap-option-   | 0        | 1  | Up | -  | equal  | not-sent |
|    | oscore-kidctx  |          |    |    |    |        |          |
| 11 | coap-option-   | var      | 1  | Up | -  | ignore | value-   |
|    | oscore-kid     |          |    |    |    |        | sent     |
| 12 | space-id-      | 0        | 1  | Dw | -  | equal  | not-sent |
|    | coap(9)        |          |    |    |    |        |          |
+----+----------------+----------+----+----+----+--------+----------+
~~~
{: #fig-example-rule title="Example Compression Rule for an Outer CoAP/OSCORE Header"}

This rule uses the new capabilities introduced above:

* The Token field's length no longer uses the legacy
  "fl-token-length" function; instead, "fl-length-bytes" is used,
  with its parameter set to 2, referring to entry 2, "coap-tkl".

* The OSCORE option's content is split into sub-fields. Note that
  the Flags field is itself split between "coap-option-oscore-flags-
  flagbits" and "coap-option-oscore-flags-n", the latter carrying the
  length of the Partial IV field.

* The Partial IV length is likewise given by "fl-length-bytes", this
  time with its parameter set to 8, referring to entry 8,
  "coap-option-oscore-flags-n".

* In the other direction, the OSCORE option is empty and is
  compressed using a Universal Option, indicating option number 9.

TO BE DISCUSSED: the "h" flag is set to 0, meaning the KID Context
is normally absent; should entry 10, "coap-option-oscore-kidctx",
still appear in the rule in that case? When present, the KID
Context value is itself encoded as a length byte followed by the
context bytes. Should this document keep it as a single
"coap-option-oscore-kidctx" entry of variable length -- which would
send that length twice, once through SCHC's own variable-length
encoding and once through the length byte already embedded in the
OSCORE encoding -- or split it into two entries, a length indicator
and the context value, with the latter using "fl-length-bytes" to
point to the former?

# Variable Length in Bits {#variable-length-in-bits}

"fl-variable-bits" generalizes RFC 8824's {{RFC8824}} "fl-variable"
to bit-level variable-length fields, the same way
{{I-D.ietf-schc-8824-update}}'s "var_bit" function does. RFC 8724
{{RFC8724}} requires the "MSB" matching operator's parameter to be a
multiple of 8 bits when applied to a byte-counted variable-length
field ("fl-variable"): the residue sent with the "LSB" action would
otherwise not be an integral number of bytes, and "fl-variable"'s
length prefix, itself byte-counted, could not express it.
"fl-variable-bits" removes that restriction by counting the
residue's length prefix in bits instead of bytes.

The length prefix itself is encoded the same way as for
"fl-variable", following RFC 8724 {{RFC8724}}, Section 7.4.2: sizes
between 0 and 14 (in the unit defined by the FL -- bytes for
"fl-variable", bits for "fl-variable-bits") are encoded as a 4-bit
unsigned integer; sizes between 15 and 254 are encoded as 0b1111
followed by an 8-bit unsigned integer; larger sizes are encoded as
0xfff followed by a 16-bit unsigned integer. Only the counted unit
changes between the two functions, not the escape structure of the
length prefix.

## Example, Non-Byte-Aligned MSB Residue

The case "var_bit" and "fl-variable-bits" actually address is a
variable-length field, whose residue needs an explicit length
prefix. {{I-D.ietf-schc-8824-update}} itself has such an example, for
the OSCORE "kid" sub-field (entry "coap-option-oscore-kid" in
{{fig-example-rule}}, which instead sends it unmatched):
{{fig-example-msb}}. "msb(44)" matches the KID's most significant 44
bits against the 6-byte (48-bit) target value; only the remaining 4
bits  are sent with "lsb". "var_bit" (this
document's "fl-variable-bits") carries that 4-bit length in its
residue length prefix; RFC 8824's byte-counted "fl-variable" could
not.

~~~ aasvg
+----------------+-----------+----+----+----------+---------+-----+
| Field          | FL        | FP | DI | TV       | MO      | CDA |
+----------------+-----------+----+----+----------+---------+-----+
| coap-option-   | variable- | 1  | Up | 0x636c69 | msb(44) | lsb |
| oscore-kid     | bits      |    |    | 656e70   |         |     |
+----------------+-----------+----+----+----------+---------+-----+
~~~
{: #fig-example-msb title="Non-Byte-Aligned MSB Residue on a Variable-Length Field (adapted from I-D.ietf-schc-8824-update)"}

# Conclusion

This document proposes the following changes to the SCHC YANG data
model.

The compression Rule entry structure of {{RFC9363}}, keyed by the
tuple (Field ID, Direction Indicator, position), is deprecated in
favor of a more flexible one. An entry is now keyed by a single
"entry-index", the list of entries is "ordered-by user", and a Field
ID can be replaced by a Universal Option, i.e. the pair (Space ID,
option number). Any option of a protocol can therefore be described
in a Rule without allocating a Field ID for it, which also makes the
twenty per-option CoAP Field IDs of {{RFC9363}} unnecessary: they are
deprecated in turn, and the pair that replaces each of them is given
in {{tbl-coap-option-fids}}. The OSCORE and KUDOS suboption Field IDs,
on the other hand, remain current: they describe the parts of a single
option rather than stand for an arbitrary one.

An alternative, closer to the wire format, is defined in the
"ietf-schc-coap-syntactic" module ({{syntactic-model}}). It
decomposes an option into the three fields CoAP carries on the wire
(option delta, option length and option value) which keeps the
processing cost independent of the number of options a protocol
defines, at the price of three entries per option.

Three new Field Length functions support these structures
({{field-length-functions}}):

* "fl-length-bytes", which takes the "entry-index" of the entry
  carrying the length, expressed in bytes, and generalizes
  "fl-token-length";
* "fl-length-bits", its bit-level equivalent;
* "fl-remaining", for the last entry of a Rule, whose residue runs
  to the end of the compression residue and therefore needs no
  explicit length prefix.

The full YANG module, with all the identities, leaves and lists
discussed here, is specified in {{I-D.toutain-schc-rfc9363bis}}.

# Security Considerations

This document does not change the SCHC security model: the security
considerations of {{RFC8724}} and {{RFC9363}} apply unchanged, and a
SCHC Context remains security sensitive, since the decompressor
rebuilds a header from it.

Universal Options, and more so the syntactic representation, let a
Rule describe an option the implementation does not know, and
therefore elide and rebuild it on the sole basis of the Rule. This is
a reason to authenticate the source of a Rule modification, rather
than a new weakness of the compression scheme.

--- back


# YANG Data Model for Syntactic Compression {#syntactic-model}

This appendix defines the work in progress YANG Data Model for the
syntactic approach of {{fig-synt-not-sent}}, adding the three Field
IDs an option is decomposed into.

~~~~~~~~~~~ yang
<CODE BEGINS> file "ietf-schc-coap-syntactic@2025-03-21.yang"
module ietf-schc-coap-syntactic {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-schc-coap-syntactic";
  prefix schc-coap-syntactic;

  import ietf-schc {
      prefix ietf-schc;
  }

  organization
    "IETF Static Context Header Compression (schc) Working Group";
  contact
    "WG Web:   <https://datatracker.ietf.org/wg/schc/about/>
     WG List:  <mailto:schc@ietf.org>
     Editor:   Laurent Toutain
       <mailto:laurent.toutain@imt-atlantique.fr>
     Editor:   Quentin Lampin
       <mailto:quentin.lampin@orange.com>";
  description
     "
     Copyright (c) 2021 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject to
     the license terms contained in, the Simplified BSD License set
     forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (https://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX
     (https://www.rfc-editor.org/info/rfcXXXX); see the RFC itself
     for full legal notices.

     The key words 'MUST', 'MUST NOT', 'REQUIRED', 'SHALL', 'SHALL
     NOT', 'SHOULD', 'SHOULD NOT', 'RECOMMENDED', 'NOT RECOMMENDED',
     'MAY', and 'OPTIONAL' in this document are to be interpreted as
     described in BCP 14 (RFC 2119) (RFC 8174) when, and only when,
     they appear in all capitals, as shown here.

     *************************************************************************

     This module extends the ietf-schc module to include the compound-ack 
     behavior for Ack On Error as defined in RFC YYYY. 
     It introduces a new leaf for Ack on Error defining the format of the
     SCHC Ack and add the possibility to send several bitmaps in a single 
     answer.";

  revision 2025-03-21 {
    description
      "Initial version for RFC YYYY ";
    reference
      "RFC YYYY: OAM";
  }

  identity fid-coap-delta-types {
       base ietf-schc:fid-coap-base-type;
       description
         "test for new options";
       reference
         "RFC 7252 The Constrained Application Protocol (CoAP)";
     }

  identity fid-coap-length {
       base ietf-schc:fid-coap-base-type;
       description
         "test for new options";
       reference
         "RFC 7252 The Constrained Application Protocol (CoAP)";
     }

   identity fid-coap-value {
       base ietf-schc:fid-coap-base-type;
       description
         "test for new options";
       reference
         "RFC 7252 The Constrained Application Protocol (CoAP)";
     }
}
<CODE ENDS>
~~~~~~~~~~~
{: sourcecode-name="ietf-schc-coap-syntactic@2025-03-21.yang" sourcecode-markers="true"}

# Acknowledgments # {#acknowledgments}
{:unnumbered}

The authors sincerely thank

This work was supported by the Sweden's Innovation Agency VINNOVA within the EUREKA CELTIC-NEXT project CYPRESS.

The authors also thank Claude (Anthropic) for rewriting parts of
this document in more correct English and for checking its
consistency.
