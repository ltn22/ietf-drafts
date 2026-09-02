---
title: "Toward RFC 9363bis: Changes to the SCHC YANG Data Model"
abbrev: "Toward SCHC YANG Model bis"
category: std
docname: draft-toutain-schc-toward-rfc9363bis-00
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Independent Submission"
keyword:
 - SCHC
 - YANG
 - header compression
venue:
  group: "SCHC"
  type: "Working Group"
  mail: "schc@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/schc/"
  github: ""
  latest: ""

author:
 -
    ins: A. Minaburo
    name: Ana Minaburo
    org: Consultant
    email: ana@minaburo.com
 -
    ins: L. Toutain
    name: Laurent Toutain
    org: Institut MINES TELECOM; IMT Atlantique
    street: 2 rue de la Chataigneraie CS 17607
    city: Cesson-Sevigne Cedex
    code: 35576
    country: France
    email: Laurent.Toutain@imt-atlantique.fr

normative:
  RFC2119:
  RFC8174:
  RFC7950:
  RFC9595:

informative:
  RFC9363:
  RFC8724:
  RFC8824:
  RFC9441:
  RFC8613:
  RFC4443:
  RFC7252:
  RFC7641:
  RFC7959:
  RFC7967:
  RFC8407:
  I-D.ietf-schc-universal-option:
  I-D.ietf-schc-8824-update:
  I-D.ietf-schc-icmpv6-compression:
  I-D.ietf-core-oscore-key-update:
  I-D.toutain-schc-sid-allocation:
  I-D.toutain-core-private-sid-translation:

--- abstract

This document is not a revision of RFC 9363, "A YANG Data Model for
Static Context Header Compression (SCHC)": it identifies changes --
additions to, and removals from, its YANG data model -- motivated by
discussions in the SCHC working group and by drafts published since
RFC 9363. These changes include more flexible compression Rule
entries through the use of Universal Options, which allow
identifiers to be added to or removed from a Rule Description
depending on the Universal Options in use; some new Field Length
functions, and new Matching Operators (MOs) and
Compression/Decompression Actions (CDAs); and a mechanism for the
manual allocation of YANG Schema Item iDentifiers (SIDs). Once the
working group agrees on the resulting wording, these changes are
intended to be incorporated into a future revision of RFC 9363.

--- middle

# Introduction

This document is not itself a revision of RFC 9363 {{RFC9363}}, the
YANG data model for Static Context Header Compression (SCHC)
{{RFC8724}}. Instead, it identifies the items that should be added
to, or removed from, that data model, taking into account
discussions held in the SCHC working group and drafts published
since RFC 9363 (e.g. {{I-D.ietf-schc-universal-option}},
{{I-D.ietf-schc-8824-update}}, and
{{I-D.ietf-schc-icmpv6-compression}}). Once the working group agrees
on the resulting wording, a new revision of RFC 9363 will be issued
to formally incorporate it.

This document also introduces the framework for Rule management; the
details of Rule management are left to separate documents.

# Universal Options {#universal-options}

In RFC 9363 {{RFC9363}}, each field found in a header is referenced
by a globally unique identifier called a Field ID (FID). The SCHC
YANG module defines an identityref for each FID. This static
allocation approach breaks down when a protocol carries options,
such as CoAP {{RFC7252}}: new options appear regularly, and the
mapping between an option number and a FID is not trivial.

{{I-D.ietf-schc-universal-option}} augments the compression Rule
entry, in which the FID is replaced by a tuple made of a space ID
and the option number, so that any option of a given protocol can
be referenced without allocating a dedicated FID for it. In this
approach, the key used to access a regular field entry remains the
FID, the Direction Indicator, and the position, while the key used
to access an option entry becomes the space ID, the option number,
and the position.

This key-based approach does not guarantee that entries appear in
the same order as the corresponding fields in the header (all the
options will appear at the end of the rule), whereas RFC 8724
{{RFC8724}} mandates that order and recent implementations have
shown it to be efficient. In addition, these three- or four-element
keys remain large, which may be a penalty when Rule management is
used (see {{rule-management}}).

This document deprecates the whole compression Rule entry structure
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
for the new "field-length-value" leaf, added to carry the
entry-index argument discussed later.

RFC 7950 {{RFC7950}}, Section 11, does not allow the "entry" list's
key, nor the "field-id" leaf, to be changed in place in a published
module ("Otherwise, if the semantics of any previous definition are
changed [...] then this MUST be achieved by a new definition with a
new identifier"). This document therefore leaves the RFC 9363
"compression-rule-entry" grouping, its "compression-content"
grouping, and the "compression" case untouched, and marks them
"status deprecated;" (RFC 8407 recommends a "deprecated" status be
kept for at least one year before moving to "obsolete"). The new
structure is defined in a new "compression-rule-entry-universal" grouping,
used by a new "compression-content-universal" grouping and a new
"compression-universal" case, sitting alongside the deprecated ones in the
"nature" choice.

{{fig-compression-rule-entry}} shows the resulting YANG tree diagram
for both cases (deprecated "compression" and new "compression-universal").

~~~
x--:(compression) {compression}?
|  x--rw entry* [field-id field-position direction-indicator]
|     +--rw field-id                    fid-type
|     +--rw field-length                union
|     +--rw field-position              uint8
|     +--rw direction-indicator         di-type
|     +--rw target-value* [index]
|     +--rw matching-operator           mo-type
|     +--rw matching-operator-value* [index]
|     +--rw comp-decomp-action          cda-type
|     +--rw comp-decomp-action-value* [index]
+--:(compression-universal) {compression or management}?
   +--rw entry-universal* [entry-index]
      +--rw entry-index                 uint16
      +--rw (field-or-space)
      |  +--:(regular-field)
      |  |  +--rw field-id?             fid-type
      |  +--:(universal-option)
      |     +--rw space-id?             space-id-type
      |     +--rw universal-value?      uint64
      +--rw field-length                union
      +--rw field-length-value?         uint16
      +--rw field-position              uint8
      +--rw direction-indicator         di-type
      +--rw target-value* [index]
      +--rw matching-operator           mo-type
      +--rw matching-operator-value* [index]
      +--rw comp-decomp-action          cda-type
      +--rw comp-decomp-action-value* [index]
~~~
{: #fig-compression-rule-entry title="Compression Rule Entry: deprecated (RFC 9363) and new (-universal) cases"}

Unlike the deprecated "entry" list, keyed by the compound
"field-id"/"field-position"/"direction-indicator", "entry-universal"
is keyed by the single-leaf "entry-index". This makes an entry
cheap and unambiguous to reference from elsewhere within the same
Rule, e.g. the "field-length-value" argument of "fl-length-bytes"
and "fl-length-bits" ({{field-length-functions}}), or a management
operation targeting it ({{rule-management}}).

## Space ID

Several protocols define options: CoAP {{RFC7252}} is one example,
but other protocols define their own options too, each with its own
option numbering. A single "space-id" value is therefore not enough
by itself; it must be qualified by the protocol (option space) it
belongs to.

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
"field-or-space" choice (see {{compression-rule-entry}}).

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
registered in the IETF YANG-SID Modules registry. None from the range 
registered for RFC 9363 MUST be allocated to these twenty deprecated 
identities: a SID "immutably maps to EXACTLY one YANG name", so 
allocating one to an identity already superseded by Universal Options 
would waste it permanently, with no way to reclaim it later.

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



## OSCORE and KUDOS Suboptions {#oscore-kudos-fids}

Universal Options allow any option to be compressed following the
SCHC principle, but compression can be more efficient when the
sub-fields of an option are taken into account individually. This is
why the module introduces specific fields for OSCORE {{RFC8613}} and
KUDOS {{I-D.ietf-core-oscore-key-update}}: each option is split into
sub-fields, each with its own identity ("fid-coap-option-oscore-piv",
"fid-coap-option-oscore-kid", "fid-coap-option-oscore-kidctx",
"fid-coap-option-kudos-nonce", etc.).

Note that the Flags field, for both OSCORE and KUDOS, is also split
so that its length indicator becomes a specific field of its own:
"fid-coap-option-oscore-flags-flags"/"fid-coap-option-oscore-flags-n"
for OSCORE, and "fid-coap-option-kudos-x-flags"/
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
unchanged, as the base identity of these OSCORE and KUDOS suboption
FIDs. Unlike the twenty per-option FIDs deprecated in the previous
subsection, they use "fid-coap-option" as a typing hierarchy for
parts of a single CoAP option (OSCORE, KUDOS), not as a stand-in for
an arbitrary CoAP option, and are therefore unaffected: they remain
"current".

## Field Length Functions {#field-length-functions}

RFC 8824 {{RFC8824}} defines a specific length function for the
Token field: when this function is specified, the value of the
Token Length (TKL) is used to indicate the length of the field.
{{I-D.ietf-schc-8824-update}} extends this to the Partial IV field
for OSCORE.

This approach is not scalable: a new function would have to be
defined for every new protocol field whose length needs to be
carried this way. This document instead introduces two new
functions, "fl-length-bytes" and "fl-length-bits". They both take an
argument, carried in the new Rule entry field "field-length-value",
that points to the entry where the length is specified, referenced
by its "entry-index" (see {{compression-rule-entry}}). Having two
separate functions, rather than a single generic one, also conveys
the unit of the length value: bytes for "fl-length-bytes", bits for
"fl-length-bits".

* "fl-length-bytes" generalizes "fl-token-length": for example,
  "fl-token-length" is now equivalent to "fl-length-bytes(index)",
  where "index" is the "entry-index" of the CoAP TKL field.
* "fl-length-bits" is the bit-level equivalent of "fl-length-bytes".

This is what {{oscore-kudos-fids}} relies on: the OSCORE and KUDOS
Flags field is split so that its length indicator
("fid-coap-option-oscore-flags-n", "fid-coap-option-kudos-x-m") is
its own entry; "fl-length-bytes" or "fl-length-bits" can then use
that entry's "entry-index" to derive the length of the corresponding
Partial IV field. This works directly for OSCORE;
{{oscore-kudos-fids}} explains why the same does not hold for
KUDOS's Nonce.

This document also introduces "fl-any", for an entry that is always
the last one in the Rule (see {{icmpv6}}'s "fid-payload"). Unlike
"fl-variable", which prefixes its residue with an explicit length
(see {{variable-length-in-bits}}), "fl-any" accepts a field of any
length but does not add that prefix when serializing the residue:
being the last entry, its residue already runs to the end of the
SCHC packet and needs no delimiter.

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

~~~
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
|    | flags          |          |    |    |    |        |          |
| 8  | coap-option-   | 3        | 1  | Up | 01 | equal  | not-sent |
|    | oscore-flags-n |          |    |    |    |        |          |
| 9  | coap-option-   | length-  | 1  | Up | -  | ignore | value-   |
|    | oscore-piv     | bytes(8) |    |    |    |        | sent     |
| 10 | coap-option-   | 0        | 1  | Up | 00 | equal  | not-sent |
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
  flags" and "coap-option-oscore-flags-n", the latter carrying the
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

~~~
+----------------+-----------+----+----+----------+---------+-----+
| Field          | FL        | FP | DI | TV       | MO      | CDA |
+----------------+-----------+----+----+----------+---------+-----+
| coap-option-   | variable- | 1  | Up | 0x636c69 | msb(44) | lsb |
| oscore-kid     | bits      |    |    | 656e70   |         |     |
+----------------+-----------+----+----+----------+---------+-----+
~~~
{: #fig-example-msb title="Non-Byte-Aligned MSB Residue on a Variable-Length Field (adapted from I-D.ietf-schc-8824-update)"}

# ICMPv6 {#icmpv6}

{{I-D.ietf-schc-icmpv6-compression}} defines how SCHC can interact
with ICMPv6 {{RFC4443}}, either to compress an ICMPv6 message or to
generate one. This document focuses only on the resulting impact on
the YANG Data Model. Since an ICMPv6 message may itself carry an
IPv6 header -- e.g. the offending packet embedded in an error
message -- the draft introduces new Matching Operators and
Compression/Decompression Actions to compress that payload; we open
the discussion on an alternative behavior for it below.

## New FID

{{I-D.ietf-schc-icmpv6-compression}} introduces eight new Field IDs
for ICMPv6 {{RFC4443}}:

* "fid-icmpv6-type", "fid-icmpv6-code", "fid-icmpv6-checksum":
  present in every ICMPv6 message.
* "fid-icmpv6-mtu": present in the Packet Too Big message.
* "fid-icmpv6-pointer": present in the Parameter Problem message.
* "fid-icmpv6-identifier", "fid-icmpv6-sequence": present in the
  Echo Request/Reply message.
* "fid-icmpv6-payload": the data following the ICMPv6 header.

Two more generic Field IDs, not specific to ICMPv6, are found in the
working copy of the module alongside them. Being generic, they can
be used as-is when compressing other protocols:

* "fid-unused": as its name suggests, used to skip an unused part of
  a header. This matters when parsing and compression happen on the
  fly, and reinforces the constraint that fields appear in the same
  order as in the header. Setting its Target Value to 0, its
  Matching Operator to "ignore", and its Compression/Decompression
  Action to "not-sent" is RECOMMENDED.
* "fid-payload": MUST be the last entry of the Rule, matching its
  position as the packet's trailing payload, and uses the new
  "fl-any" ({{field-length-functions}}), which accepts any length
  without prefixing the residue with one, since the last entry's
  residue already runs to the end of the SCHC packet. With Matching
  Operator "ignore" and Compression/Decompression Action
  "value-sent", it behaves exactly as the usual, implicit SCHC
  behavior, where the payload simply follows the compression
  residue. But it can also be used to intercept the payload:
  Compression/Decompression Action "not-sent" then elides it, or
  another CDA can apply a special treatment to compress it, instead
  of sending it verbatim.

## New Matching Operators and Compression/Decompression Actions

{{I-D.ietf-schc-icmpv6-compression}} is also the origin of the
"mo-rule-match"/"mo-rev-rule-match" Matching Operators and the
"cda-compress-sent"/"cda-rev-compress-sent" Compression/
Decompression Actions, already listed in
{{changes-from-rfc-9363}}. "mo-rule-match" returns true if the
Target Value matches another Rule, keeping the Up/Down direction;
"mo-rev-rule-match" does the same but reversing that direction;
"cda-compress-sent" and "cda-rev-compress-sent" send a compressed
version of the Target Value, using respectively the matched Rule or
its direction-reversed counterpart.

The reversed forms exist because, per RFC 4443 {{RFC4443}}, an
ICMPv6 error message carries back as much as possible of the IPv6
packet that triggered it -- a packet that was sent in the opposite
direction from the error message itself. Compressing that embedded
copy therefore means matching it against, and generating its
residue from, a Rule for the reverse direction.

### Example, ICMPv6 Error with Reverse Compression

{{fig-example-icmpv6}} adapts the "Time Exceeded" Rule from
{{I-D.ietf-schc-icmpv6-compression}}, prefixed with the outer IPv6
header carrying the ICMPv6 message itself (as in {{fig-alt-icmpv6}}
below): the Type and Code identify the error, the Checksum is
recomputed on decompression, the 32-bit "Unused" field mandated by
RFC 4443 {{RFC4443}} for this message is elided with "fid-unused"
({{icmpv6}}), and the Payload -- the offending IPv6 packet -- is
matched and compressed against its own (Up) Rule with
"rev-rule-match" and "rev-compress-sent", instead of being sent in
full.

~~~
+--------------+-----------+----+----+--------+----------+-----------+
| Field        | FL        | FP | DI | TV     | MO       | CDA       |
+--------------+-----------+----+----+--------+----------+-----------+
+------------------------ Outer IPv6 header -------------------------+
| ipv6-version | 4         | 1  | Dw | 6      | equal    | not-sent  |
| ipv6-        | 8         | 1  | Dw | 0      | ignore   | not-sent  |
| trafficclass |           |    |    |        |          |           |
| ipv6-        | 20        | 1  | Dw | 0      | ignore   | not-sent  |
| flowlabel    |           |    |    |        |          |           |
| ipv6-        | 16        | 1  | Dw | -      | ignore   | compute   |
| payload-     |           |    |    |        |          |           |
| length       |           |    |    |        |          |           |
| ipv6-        | 8         | 1  | Dw | 58     | equal    | not-sent  |
| nextheader   |           |    |    |        |          |           |
| ipv6-        | 8         | 1  | Dw | 1      | equal    | not-sent  |
| hoplimit     |           |    |    |        |          |           |
| ipv6-        | 64        | 1  | Dw | aaaa:: | equal    | not-sent  |
| devprefix    |           |    |    |        |          |           |
| ipv6-deviid  | 64        | 1  | Dw | ::zzzz | equal    | not-sent  |
| ipv6-        | 64        | 1  | Dw | -      | ignore   | value-    |
| appprefix    |           |    |    |        |          | sent      |
| ipv6-appiid  | 64        | 1  | Dw | -      | ignore   | value-    |
|              |           |    |    |        |          | sent      |
+-------------------------- ICMPv6 header ---------------------------+
| icmpv6-type  | 8         | 1  | Dw | 3      | equal    | not-sent  |
| icmpv6-code  | 8         | 1  | Dw | [0,1]  | match-   | mapping-  |
|              |           |    |    |        | mapping  | sent      |
| icmpv6-      | 16        | 1  | Dw | -      | ignore   | compute   |
| checksum     |           |    |    |        |          |           |
| unused       | 32        | 1  | Dw | 0      | ignore   | not-sent  |
| icmpv6-      | variable- | 1  | Dw | 0      | rev-     | rev-      |
| payload      | bits      |    |    |        | rule-    | compress- |
|              |           |    |    |        | match    | sent      |
+--------------+-----------+----+----+--------+----------+-----------+
~~~
{: #fig-example-icmpv6 title="ICMPv6 Error Compressed Against a Reverse-Direction Rule (adapted from I-D.ietf-schc-icmpv6-compression)"}

{{fig-example-icmpv6}} shows a Rule inspired by
{{I-D.ietf-schc-icmpv6-compression}}. One difference here is the
addition of the "unused" entry, skipping the 32 bits following the
Checksum. The remaining bytes are assigned to "icmpv6-payload" and
contain the original (invoking) header. If a Rule in the context
matches that payload, compression applies to it as well.

{{fig-icmpv6-residue}} shows the resulting residue: the RuleID of
the ICMPv6 message, followed by its own compression residue, then a
length for the variable-length structure, the RuleID of the Rule
that compresses the original header, that Rule's compression
residue, the remaining, uncompressed payload, and, since this
concludes the SCHC packet, the final padding bits ({{RFC8724}}) that
must bring it to a byte boundary, since "Length" here counts bytes,
not bits: spanning the inner RuleID, header residue, and payload
together, it is typically well above the 14-byte escape threshold
({{variable-length-in-bits}}), so counting it in bytes keeps that
length prefix itself compact.

~~~
                             |----------------Length-----------------|
+---------+---------+--------+---------+---------+---------+---------+
| RuleID  | IPv6/   | Length | RuleID  | IPv6/   | Payload | Padding |
| (outer) | ICMPv6  |        | (inner) | UDP     |         |         |
|         | residue |        |         | residue |         |         |
+---------+---------+--------+---------+---------+---------+---------+
~~~
{: #fig-icmpv6-residue title="Residue Layout for an ICMPv6 Error Compressing an Embedded, Compressed Header"}

## Alternative: Include IPv6 in Header Format

An alternative to {{I-D.ietf-schc-icmpv6-compression}} is to
continue the compression process inside the ICMPv6 payload, instead
of matching it against a separate Rule. {{fig-alt-icmpv6}} shows the
resulting Rule. 

The invoking header's Field Descriptors are distinct from the outer
header's, and are designed from the ICMPv6 message's point of view.

Fields designed with an application or device role remain unchanged
(e.g. "ipv6-deviid" or "udp-app-port"), but the direction is
reversed. Field Position is also incremented if a field is repeated,
as for the IPv6 fields in the example.

The compression mechanism MUST also include the port numbers: an
ICMPv6 error message exists to inform the source that a given flow
failed to reach its destination, and the port numbers are part of
that flow's identification. It is RECOMMENDED to be able to fully
reconstruct the Layer 4 header this way, not just the ports.

~~~
+--------------+-----+----+----+--------+--------+----------+
| Field        | FL  | FP | DI | TV     | MO     | CDA      |
+--------------+-----+----+----+--------+--------+----------+
+-------------------- Outer IPv6 header --------------------+
| ipv6-version | 4   | 1  | Bi | 6      | equal  | not-sent |
| ipv6-        | 8   | 1  | Bi | 0      | ignore | not-sent |
| trafficclass |     |    |    |        |        |          |
| ipv6-        | 20  | 1  | Bi | 0      | ignore | not-sent |
| flowlabel    |     |    |    |        |        |          |
| ipv6-        | 16  | 1  | Bi | -      | ignore | compute  |
| payload-     |     |    |    |        |        |          |
| length       |     |    |    |        |        |          |
| ipv6-        | 8   | 1  | Bi | 58     | equal  | not-sent |
| nextheader   |     |    |    |        |        |          |
| ipv6-        | 8   | 1  | Up | -      | ignore | value-   |
| hoplimit     |     |    |    |        |        | sent     |
| ipv6-        | 8   | 1  | Dw | 1      | equal  | not-sent |
| hoplimit     |     |    |    |        |        |          |
| ipv6-        | 64  | 1  | Bi | aaaa:: | equal  | not-sent |
| devprefix    |     |    |    |        |        |          |
| ipv6-deviid  | 64  | 1  | Bi | ::zzzz | equal  | not-sent |
| ipv6-        | 64  | 1  | Bi | -      | ignore | value-   |
| appprefix    |     |    |    |        |        | sent     |
| ipv6-appiid  | 64  | 1  | Bi | -      | ignore | value-   |
|              |     |    |    |        |        | sent     |
+---------------------- ICMPv6 header ----------------------+
| icmpv6-type  | 8   | 1  | Bi | 1      | equal  | not-sent |
| icmpv6-code  | 8   | 1  | Bi | 4      | equal  | not-sent |
| icmpv6-      | 16  | 1  | Bi | 0      | ignore | compute  |
| checksum     |     |    |    |        |        |          |
| unused       | 32  | 1  | Bi | 0      | ignore | not-sent |
+------------- Invoking (embedded) IPv6 header -------------+
| ipv6-version | 4   | 2  | Bi | 6      | equal  | not-sent |
| ipv6-        | 8   | 2  | Bi | 0      | ignore | not-sent |
| trafficclass |     |    |    |        |        |          |
| ipv6-        | 20  | 2  | Bi | 0      | ignore | not-sent |
| flowlabel    |     |    |    |        |        |          |
| ipv6-        | 16  | 2  | Bi | -      | ignore | compute  |
| payload-     |     |    |    |        |        |          |
| length       |     |    |    |        |        |          |
| ipv6-        | 8   | 2  | Bi | 17     | equal  | not-sent |
| nextheader   |     |    |    |        |        |          |
| ipv6-        | 8   | 2  | Dw | 1      | equal  | not-sent |
| hoplimit     |     |    |    |        |        |          |
| ipv6-        | 8   | 2  | Up | -      | ignore | value-   |
| hoplimit     |     |    |    |        |        | sent     |
| ipv6-        | 64  | 2  | Bi | aaaa:: | equal  | not-sent |
| devprefix    |     |    |    |        |        |          |
| ipv6-deviid  | 64  | 2  | Bi | ::zzzz | equal  | not-sent |
| ipv6-        | 64  | 2  | Bi | -      | ignore | value-   |
| appprefix    |     |    |    |        |        | sent     |
| ipv6-appiid  | 64  | 2  | Bi | -      | ignore | value-   |
|              |     |    |    |        |        | sent     |
+------------- Invoking (embedded) UDP header --------------+
| udp-dev-port | 16  | 1  | Bi | 5683   | equal  | not-sent |
| udp-app-port | 16  | 1  | Bi | -      | ignore | value-   |
|              |     |    |    |        |        | sent     |
| udp-length   | 16  | 1  | Bi | 0      | ignore | compute  |
| udp-checksum | 16  | 1  | Bi | 0      | ignore | compute  |
+--------------+-----+----+----+--------+--------+----------+
| payload      | any | 1  | Bi | -      | ignore | not-sent |
+--------------+-----+----+----+--------+--------+----------+
~~~
{: #fig-alt-icmpv6 title="Alternative ICMPv6 Rule Embedding the Invoking IPv6/UDP Header In Place"}


The compression result should be better, since there is no need to
send a variable-length residue and a second RuleID, at the cost of
a more complex Rule definition.

~~~
+----------+------------------------------------------+
|  RuleID  |       IPv6/ICMPv6/IPv6/UDP residue       |
+----------+------------------------------------------+
~~~
{: #fig-alt-icmpv6-residue title="Residue Layout for the Alternative ICMPv6 Rule"}

# Compound ACK {#compound-ack}

RFC 9441 {{RFC9441}}, "Static Context Header Compression (SCHC)
Compound Acknowledgement (ACK)", includes its own YANG module,
"ietf-schc-compound-ack", which augments the "ack-on-error"
fragmentation mode
("/schc/rule/nature/fragmentation/mode/ack-on-error") with two new
leaves. This document's module revision incorporates them:

* "bitmap-format": how bitmaps are carried in a SCHC ACK message: an
  identityref to "bitmap-format-type", defaulting to
  "bitmap-RFC8724".
* "last-bitmap-compression": a boolean, true by default, indicating
  whether the last bitmap in a SCHC ACK message can be compressed.

The corresponding new identities are:

* "bitmap-format-base-type": base identity for how a bitmap is
  formed in ACK messages.
* "bitmap-RFC8724": the default bitmap format, as already defined in
  RFC 8724 {{RFC8724}}.
* "bitmap-compound-ack": allows several bitmaps within a single ACK
  message.

# Rule Management {#rule-management}

The SCHC YANG data model provides support for Rule management
through the "management" feature and the "nature-management" Rule
nature. A management Rule uses the compression Rule structure, 
but is kept separate from regular compression Rules through the
"nature-management" Rule nature: IPv6 addresses and port
numbers are specially dedicated to identifying it, and MAY overlap
with values used by regular compression Rules; the nature of the
Rule is what allows distinguishing them. Management Rules are
the only ones allowed to access the Static Context, to read, create,
update, or delete Rules. Only the "entry-universal" structure
supports management Rules; the deprecated "entry" list does not.

"guard-period", the management timer, is not really a property of
any single Rule: it applies to the SCHC context as a whole. Nesting
it inside "list rule" (whether inside one case or as a sibling of
"choice nature") would raise the same reachability problem "case
management" would have had, since it would need repeating, and
re-keying under SIDs, in every case that might need it. The revision
instead moves "guard-period" out of "list rule" entirely, into a new
top-level "context" container, a sibling of "schc", present only
"if-feature management". Being outside any Rule, it no longer needs
a "rule-nature"-based "must" at all.

~~~ yang
grouping compression-content-universal {
  list entry-universal {
    must "derived-from-or-self(../rule-nature,
                                'nature-compression') or
          derived-from-or-self(../rule-nature,
                                'nature-management')" {
      error-message
        "Rule nature must be compression or management";
    }
    /* ... unchanged: key, ordered-by, uses, description ... */
  }
}

container context {
  if-feature "management";
  uses management-content;
  description
    "Management-related parameters that apply to the whole SCHC
     context rather than to a single Rule.";
}

list rule {
  /* ... unchanged: key, rule-id-type, rule-nature ... */
  choice nature {
    case fragmentation {
      if-feature "fragmentation";
      uses fragmentation-content;
    }
    case compression {
      /* ... unchanged, still deprecated ... */
    }
    case compression-universal {
      if-feature "compression or management";
      uses compression-content-universal;
    }
  }
}
~~~
{: #fig-management-in-compression-universal title="Widening compression-universal to Also Serve Management Rules (module revision 2026-08-29)"}

For management, a management context is added: its values are
common to all SCHC Rules in that management instance, rather than
specific to any one Rule. Currently, the context contains a guard
period, defining the time before a RuleID can be reused by
management when a new Rule is created.

Different management operations can be defined to act on Rules or on
individual elements of a Rule. For example, the "duplicate-rule" RPC
can be used to create a new Rule from an existing one under a
different RuleID, with selected elements of the new Rule modified as
part of the operation.

This document defines the YANG structures needed to support Rule
management. The complete set of management operations -- their
semantics, encoding, exchange, and the procedures used to apply them
to a SCHC Context -- is specified separately.

# Manual SID Allocation

The mapping between YANG identifiers and SID {{RFC9595}} values can
be generated automatically with the "pyang" tool (e.g. via "pyang
--sid-generate-file"). This automatic allocation, however, is not
optimized.

The first goal of a manual allocation is to minimize the delta
between SIDs used together as CBOR/CORECONF keys, so that delta
encodes on a single byte (i.e., a value between -24 and +23).
{{I-D.toutain-schc-sid-allocation}} shows that pyang's automatic,
alphabetical assignment defeats this: for example, "rule-id-value",
"rule-id-length", and "rule-nature", present in every Rule, end
up with a delta higher than 23 from their base, so every single Rule
pays for a 2-byte delta where a 1-byte one would do.

{{I-D.toutain-schc-sid-allocation}} makes two recommendations to
keep this delta small. First, keep data-carrying and
identity-carrying nodes in separate SID ranges, since they are
rarely encoded together; the distance between the two can then be as
large as 255, allowing a 2-byte delta only where it does not matter.
Second, leave some SIDs unused around the SCHC Rule identifiers, so
the module can be augmented later (i.e. new leaves added) without
pushing any of these frequently co-occurring identifiers' deltas
past the single-byte threshold.

TODO: describe the resulting manual SID allocation mechanism: SIDs
are first generated automatically, then remapped to a stable,
manually curated allocation table (module, namespace, identifier) so
that previously published SIDs do not shift when the module evolves
(fields added, removed, or deprecated). Reference {{RFC9595}} for
the SID file format and describe the augmentation used to carry type
information per SID item.

The delta encoding above optimizes SIDs used as CBOR map keys, but
does nothing for a SID used as a value, for instance an identityref
leaf's value, such as a Rule entry's "matching-operator" or
"comp-decomp-action". {{I-D.toutain-core-private-sid-translation}}
addresses this with a new Compression/Decompression Action,
"cda-sid-translation": it replaces such a SID value with a "private
SID", a small negative number, computed from the real SID, an
"entry-point", and an offset. It reports a worked IPv6/UDP/CoAP
compression Rule example where this reduces a 3994-byte file to
3057 bytes, a 23% reduction.

With this technique, an identity's first 24 possible values (private
SIDs -1 to -24) each encode on a single byte -- valuable for the
identityref values used most intensively across a Rule, such as
"mo-equal", "mo-ignore", "cda-value-sent", and "cda-not-sent". The
next 232 values (private SIDs -25 to -256) still encode on 2 bytes,
an improvement over an untranslated SID from the RFC SID range,
which always takes 3 bytes in the "ietf-schc" data model. Beyond
that, translation has no effect: the private SID also takes 3 bytes,
same as the original. {{fig-sid-translation}} summarizes this.

~~~
+------------------+------------+-------------+-----------+
|   Private SID    | Identities | Private SID | Plain SID |
|      range       |  covered   |    bytes    |   bytes   |
+------------------+------------+-------------+-----------+
|    -1 to -24     |     24     |      1      |     3     |
|   -25 to -256    |    232     |      2      |     3     |
|   beyond -256    |     --     |      3      |     3     |
+------------------+------------+-------------+-----------+
~~~
{: #fig-sid-translation title="Private SID Encoding Size vs. Plain SID (RFC SID Range)"}

Beyond "entry_point + 256", a private SID can instead specify a
Rule's entry-index; beyond that, the remaining space is shared
between identity and data values, with no further reserved
distinction.

## Full YANG Module

{{fig-full-module}} shows the working copy of the "ietf-schc"
module, kept up to date with every change discussed in this
document, taken from the project's working file
("ietf-schc@2026-08-29.yang"), validated with "pyang" (no
errors; see the pre-existing RFC 8407 style findings noted in its
own revision history for remaining gaps).

~~~ yang
module ietf-schc {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-schc";
  prefix schc;

  organization
    "IETF IPv6 over Low Power Wide-Area Networks (lpwan) Working
     Group";
  contact
    "WG Web:   <https://datatracker.ietf.org/wg/lpwan/about/>
     WG List:  <mailto:lp-wan@ietf.org>
     Editor:   Laurent Toutain
       <mailto:laurent.toutain@imt-atlantique.fr>
     Editor:   Ana Minaburo
       <mailto:ana@ackl.io>";
  description
    "Copyright (c) 2023 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.
     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject to
     the license terms contained in, the Revised BSD License set
     forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (https://trustee.ietf.org/license-info).
     This version of this YANG module is part of RFC 9363
     (https://www.rfc-editor.org/info/rfc9363); see the RFC itself
     for full legal notices.
     The key words 'MUST', 'MUST NOT', 'REQUIRED', 'SHALL', 'SHALL
     NOT', 'SHOULD', 'SHOULD NOT', 'RECOMMENDED', 'NOT RECOMMENDED',
     'MAY', and 'OPTIONAL' in this document are to be interpreted as
     described in BCP 14 (RFC 2119) (RFC 8174) when, and only when,
     they appear in all capitals, as shown here.
     ***************************************************************
     Generic data model for the Static Context Header Compression
     Rule for SCHC, based on RFCs 8724 and 8824.  Including
     compression, no-compression, and fragmentation Rules.

     This module is a YANG data model for SCHC Rules (RFCs 8724 and
     8824).  RFC 8724 describes compression Rules in an abstract
     way through a table.
 |-----------------------------------------------------------------|
 |  (FID)            Rule 1                                        |
 |+-------+--+--+--+------------+-----------------+---------------+|
 ||Field 1|FL|FP|DI|Target Value|Matching Operator|Comp/Decomp Act||
 |+-------+--+--+--+------------+-----------------+---------------+|
 ||Field 2|FL|FP|DI|Target Value|Matching Operator|Comp/Decomp Act||
 |+-------+--+--+--+------------+-----------------+---------------+|
 ||...    |..|..|..|   ...      | ...             | ...           ||
 |+-------+--+--+--+------------+-----------------+---------------+|
 ||Field N|FL|FP|DI|Target Value|Matching Operator|Comp/Decomp Act||
 |+-------+--+--+--+------------+-----------------+---------------+|
 |-----------------------------------------------------------------|
     This module specifies a global data model that can be used for
     Rule exchanges or modification.  It specifies both the data
     model format and the global identifiers used to describe some
     operations in fields.
     This data model applies to both compression and fragmentation.";

  revision 2026-08-29 {
    description
      "Wire the 'nature-management' Rule nature into the data
       model: widen 'compression-content-universal''s
       'entry-universal' must to accept 'nature-compression' or
       'nature-management', instead of adding a separate, SID-
       duplicating 'case management' for an identical structure;
       widen 'case compression-universal''s if-feature to
       'compression or management' accordingly; and move
       'management-content' (the 'guard-period' timer) out of
       'list rule' into a new top-level 'context' container, gated
       by 'if-feature management', since it applies to the whole
       SCHC context rather than to a single Rule.";
  }

  revision 2026-08-20 {
    description
      "Add kudos FID defined in draft-ietf-core-oscore-key-update.
       Restore the twenty per-CoAP-option FIDs (fid-coap-option-*)
       that had been removed in an earlier revision, marking them
       'status deprecated' instead of deleting them, per RFC 7950
       Section 11 ('Obsolete definitions MUST NOT be removed from
       published modules') and RFC 8407 Section 4.7 (an object
       SHOULD stay 'deprecated' for at least one year before moving
       to 'obsolete'). They are superseded by Universal Options
       (space-id-coap / universal-value). Rename the '-v2'
       compression-rule-entry/compression-content/case suffix to
       '-universal'. Fix the space-id-coap identity description and
       reference, mistakenly copy-pasted from fid-ipv6-*. Remove the
       unused fid-oscore-base-type identity: it was never used as a
       base by any other identity (the OSCORE suboption FIDs derive
       directly from fid-coap-option) and was never part of a
       published RFC, so it can simply be dropped rather than
       deprecated.";
  }

  revision 2026-05-07 {
    description
      "add two generic fid unused and payload to be used in any
      protocol and for padding and payload definition.";
  }

  revision 2026-04-05 {
    description
      "Alternative: use choice/case to structurally distinguish regular
       fields (field-id only) from universal options (space-id + mandatory
       universal-value).";
  }

  revision 2026-02-24 {
    description
      "
      - Add Management feature to support management of SCHC rules.
      - Introduce RPCs to manage rules.";
  }

  revision 2026-01-12 {
    description 
    "test module to unify universal option and regular entries:
    - all options are identified with a space-id and an option-id.
      For regular fields like fid-ipv6-version the space-id is set 
      to 0.
    - the entries are identified by an index, reducing the keys to
      a single element.
    - introduction of a new function to define the length. This option
      takes an entry-index as parameter to refer to a field indicating
      length.
      for instance fl-token-length is now equivalent to:
      fl-length_byte(index) where index refers to an entry-index for 
      TKL field.";
  }

  revision 2025-11-24 {
    description
      "This version includes new developments in the SCHC architecture
       and RFCs published since the initial version of this module.
       It includes:
       * Data model for Compound Ack as described in RFC9441
       * new Field IDs defined for CoAP OSCORE support as described
       in draft-ietf-schc-8824-update.
       * ICMPv6 FIDs defined in draft-ietf-schc-icmpv6.
       * Universal options for CoAP and other protocols options parsing
         as described in draft-ietf-schc-universal-options.
       * the coap-option FIDs are deprecated in favor of the more generic
         universal-option Field IDs.
      ";
    reference
      "RFC 9363 A YANG Data Model for Static Context Header
                Compression (SCHC)
       RFC 9441  Static Context Header Compression (SCHC) 
                Compound Acknowledgement (ACK)";
  }

  revision 2023-03-01 {
    description
      "Initial version from RFC 9363.";
    reference
      "RFC 9363 A YANG Data Model for Static Context Header
                Compression (SCHC)";
  }


  feature compression {
    description
      "SCHC compression capabilities are taken into account.";
  }

  feature fragmentation {
    description
      "SCHC fragmentation capabilities are taken into account.";
  }

  feature management {
    description
      "SCHC compression capabilities for rule management.";
  }

  // -------------------------
  //  Field ID type definition
  //--------------------------
  // generic value TV definition

  identity space-field-id-base-type {
    description
      "Field ID base type for all fields.";
  }

  identity fid-base-type {
    base space-field-id-base-type;
    description
      "Field ID base type for all fields.";
  }

 identity fid-unused {
    base fid-base-type;
    description
      "Padding field in any protocol.";
  }

  identity fid-payload {
    base fid-base-type;
    description
      "Payload field in any protocol. This field contains the remaining 
       bytes after the header fields.";
  }

  identity fid-ipv6-base-type {
    base fid-base-type;
    description
      "Field ID base type for IPv6 headers described in RFC 8200.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-version {
    base fid-ipv6-base-type;
    description
      "IPv6 version field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-trafficclass {
    base fid-ipv6-base-type;
    description
      "IPv6 Traffic Class field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-trafficclass-ds {
    base fid-ipv6-trafficclass;
    description
      "IPv6 Traffic Class field: Diffserv field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification,
       RFC 3168 The Addition of Explicit Congestion Notification
                (ECN) to IP";
  }

  identity fid-ipv6-trafficclass-ecn {
    base fid-ipv6-trafficclass;
    description
      "IPv6 Traffic Class field: ECN field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification,
       RFC 3168 The Addition of Explicit Congestion Notification
                (ECN) to IP";
  }

  identity fid-ipv6-flowlabel {
    base fid-ipv6-base-type;
    description
      "IPv6 Flow Label field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-payload-length {
    base fid-ipv6-base-type;
    description
      "IPv6 Payload Length field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-nextheader {
    base fid-ipv6-base-type;
    description
      "IPv6 Next Header field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-hoplimit {
    base fid-ipv6-base-type;
    description
      "IPv6 Next Header field.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-devprefix {
    base fid-ipv6-base-type;
    description
      "Corresponds to either the source address or the destination
       address prefix of RFC 8200 depending on whether it is an
       uplink or a downlink message.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-deviid {
    base fid-ipv6-base-type;
    description
      "Corresponds to either the source address or the destination
       address IID of RFC 8200 depending on whether it is an uplink
       or a downlink message.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-appprefix {
    base fid-ipv6-base-type;
    description
      "Corresponds to either the source address or the destination
       address prefix of RFC 8200 depending on whether it is an
       uplink or a downlink message.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-ipv6-appiid {
    base fid-ipv6-base-type;
    description
      "Corresponds to either the source address or the destination
       address IID of RFC 8200 depending on whether it is an uplink
       or a downlink message.";
    reference
      "RFC 8200 Internet Protocol, Version 6 (IPv6) Specification";
  }

  identity fid-udp-base-type {
    base fid-base-type;
    description
      "Field ID base type for UDP headers described in RFC 768.";
    reference
      "RFC 768 User Datagram Protocol";
  }

  identity fid-udp-dev-port {
    base fid-udp-base-type;
    description
      "UDP source or destination port, if uplink or downlink
       communication, respectively.";
    reference
      "RFC 768 User Datagram Protocol";
  }

  identity fid-udp-app-port {
    base fid-udp-base-type;
    description
      "UDP destination or source port, if uplink or downlink
       communication, respectively.";
    reference
      "RFC 768 User Datagram Protocol";
  }

  identity fid-udp-length {
    base fid-udp-base-type;
    description
      "UDP length.";
    reference
      "RFC 768 User Datagram Protocol";
  }

  identity fid-udp-checksum {
    base fid-udp-base-type;
    description
      "UDP length.";
    reference
      "RFC 768 User Datagram Protocol";
  }

  identity fid-coap-base-type {
    base fid-base-type;
    description
      "Field ID base type for UDP headers described.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-version {
    base fid-coap-base-type;
    description
      "CoAP version.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-type {
    base fid-coap-base-type;
    description
      "CoAP type.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-tkl {
    base fid-coap-base-type;
    description
      "CoAP token length.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-code {
    base fid-coap-base-type;
    description
      "CoAP code.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-code-class {
    base fid-coap-code;
    description
      "CoAP code class.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-code-detail {
    base fid-coap-code;
    description
      "CoAP code detail.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-mid {
    base fid-coap-base-type;
    description
      "CoAP message ID.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-token {
    base fid-coap-base-type;
    description
      "CoAP token.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option {
    base fid-coap-base-type;
    description
      "Generic CoAP option.";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-if-match {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option If-Match. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 1).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-uri-host {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Uri-Host. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 3).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-etag {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option ETag. Deprecated in favor of Universal Options
       (space-id-coap / universal-value = 4).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-if-none-match {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option if-none-match. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 5).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-observe {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Observe. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 6).";
    reference
      "RFC 7641 Observing Resources in the Constrained Application
                Protocol (CoAP)";
  }

  identity fid-coap-option-uri-port {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Uri-Port. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 7).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-location-path {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Location-Path. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 8).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-uri-path {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Uri-Path. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 11).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-content-format {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Content Format. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 12).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-max-age {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Max-Age. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 14).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-uri-query {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Uri-Query. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 15).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-accept {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Accept. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 17).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-location-query {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Location-Query. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 20).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-block2 {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Block2. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 23).";
    reference
      "RFC 7959 Block-Wise Transfers in the Constrained Application
                Protocol (CoAP)";
  }

  identity fid-coap-option-block1 {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Block1. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 27).";
    reference
      "RFC 7959 Block-Wise Transfers in the Constrained Application
                Protocol (CoAP)";
  }

  identity fid-coap-option-size2 {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Size2. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 28).";
    reference
      "RFC 7959 Block-Wise Transfers in the Constrained Application
                Protocol (CoAP)";
  }

  identity fid-coap-option-proxy-uri {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Proxy-Uri. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 35).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-proxy-scheme {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Proxy-Scheme. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 39).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-size1 {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option Size1. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 60).";
    reference
      "RFC 7252 The Constrained Application Protocol (CoAP)";
  }

  identity fid-coap-option-no-response {
    base fid-coap-option;
    status deprecated;
    description
      "CoAP option No response. Deprecated in favor of Universal
       Options (space-id-coap / universal-value = 258).";
    reference
      "RFC 7967 Constrained Application Protocol (CoAP) Option for
                No Server Response";
  }

  identity fid-coap-option-oscore-flags {
    base fid-coap-option;
    description
      "CoAP option OSCORE flags.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP) (see
                Section 6.4)";
  }

   identity fid-coap-option-oscore-flags-flags {
    base fid-coap-option-oscore-flags;
    description
      "First 5 bits of the OSCORE flags field forming flags subfield.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP) (see
                Section 6.4)";
  }

  identity fid-coap-option-oscore-flags-n {
    base fid-coap-option-oscore-flags;
    description
      "last 3 bits of the OSCORE flags field giving the length 
      of Partial IV.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP) (see
                Section 6.4)";
  }


  identity fid-coap-option-oscore-piv {
    base fid-coap-option;
    description
      "CoAP option OSCORE Partial IV.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP) (see
                Section 6.4)";
  }

  identity fid-coap-option-oscore-kid {
    base fid-coap-option;
    description
      "CoAP option OSCORE Key ID.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP) (see
                Section 6.4)";
  }

  identity fid-coap-option-oscore-kidctx {
    base fid-coap-option;
    description
      "CoAP option OSCORE Key ID Context.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP)(see
                Section 6.4)";
  }

  identity fid-coap-option-kudos-x {
    base fid-coap-option;
    description
      "x field contained in the kudos option defined in 
      draft-ietf-core-oscore-key-update.";
    reference
      "draft-ietf-core-oscore-key-update";
  }

  identity fid-coap-option-kudos-x-flags {
    base fid-coap-option-kudos-x;
    description
      "4 first bits of the x field in the kudos option defined in 
      draft-ietf-core-oscore-key-update.";
    reference
      "draft-ietf-core-oscore-key-update";
  }

  identity fid-coap-option-kudos-x-m {
    base fid-coap-option-kudos-x;
    description
      "4 last bits of the x field in the kudos option defined in 
      draft-ietf-core-oscore-key-update.";
    reference
      "draft-ietf-core-oscore-key-update";
  }

  identity fid-coap-option-kudos-nonce {
    base fid-coap-option;
    description
      "CoAP option kudos carrying the nonce.";
    reference
      "draft-ietf-core-oscore-key-update";
  }


 identity fid-icmpv6-base-type {
   base schc:fid-base-type;
   description
     "Field IP base type for ICMPv6 headers described in RFC 4443";
   reference
     "RFC 4443   Internet Control Message Protocol (ICMPv6)
                 for the Internet Protocol Version 6 (IPv6)
                 Specification";
 }
 
  identity fid-icmpv6-type {
    base fid-icmpv6-base-type;
    description
      "ICMPv6 code field present in all ICMPv6 messages.";
 }

 identity fid-icmpv6-code {
   base fid-icmpv6-base-type;
   description
     "ICMPv6 code field present in all ICMPv6 messages.";
 }

 identity fid-icmpv6-checksum {
   base fid-icmpv6-base-type;
   description
     "ICMPv6 checksum field present in all ICMPv6 messages.";
 }

 identity fid-icmpv6-mtu {
   base fid-icmpv6-base-type;
   description
     "ICMPv6 MTU, present in Packet Too Big message.";
 }

 identity fid-icmpv6-pointer {
   base fid-icmpv6-base-type;
   description
     "ICMPv6 Pointer, present in Parameter Problem message.";
 }

 identity fid-icmpv6-identifier {
   base fid-icmpv6-base-type;
   description
     "ICMPv6 identifier field, present in Echo Request/Reply
     message.";
 }

 identity fid-icmpv6-sequence {
   base fid-icmpv6-base-type;
   description
     "ICMPv6 sequence number field, present in Echo Request/Reply
      message.";
 }

 identity fid-icmpv6-payload {
   base fid-icmpv6-base-type;
   description
     "ICMPv6 payload following ICMPv6 header.
      If payload is empty, this field exists with a length of 0.";
 }

  //----------------------------------
  // Universal Option space IDs
  //----------------------------------

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

  //----------------------------------
  // Field Length type definition
  //----------------------------------

  identity fl-base-type {
    description
      "Used to extend Field Length functions.";
  }

  identity fl-variable {
    base fl-base-type;
    description
      "Residue length in bytes is sent as defined for CoAP.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP) (see
                Section 5.3)";
  }

 identity fl-variable-bits {
    base fl-base-type;
    description
      "Residue length in bits is sent as defined for CoAP.";
    reference
      "NOT YET DEFINED IN RFCs. This function is a generalization of fl-variable to support bit-level length definition.";
  }


  identity fl-token-length {
    base fl-base-type;
    description
      "Residue length in bytes is sent as defined for CoAP.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP) (see
                Section 4.5)";
  }

  identity fl-length-bytes {
    base fl-base-type;

    description "This function return the length a field 
    by its index in bytes. This is a generalation of fl-token-length.";
  }

  identity fl-length-bits {
    base fl-base-type;

    description "This function return the length in bits of a field 
    by its index in bits. ";
  }

  identity fl-any {
    base fl-base-type;
    
    description
      "Field length is not explicitly carried in the Compression Residue.
      This function is used for a field that extends to the end of the
      SCHC packet, such as a trailing payload.";
}

  //---------------------------------
  // Direction Indicator type
  //---------------------------------

  identity di-base-type {
    description
      "Used to extend Direction Indicators.";
  }

  identity di-bidirectional {
    base di-base-type;
    description
      "Direction Indicator of bidirectionality.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.1)";
  }

  identity di-up {
    base di-base-type;
    description
      "Direction Indicator of uplink.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.1)";
  }

  identity di-down {
    base di-base-type;
    description
      "Direction Indicator of downlink.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.1)";
  }

  //----------------------------------
  // Matching Operator type definition
  //----------------------------------

  identity mo-base-type {
    description
      "Matching Operator: used in the Rule selection process
       to check if a Target Value matches the field's value.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.2)";
  }

  identity mo-equal {
    base mo-base-type;
    description
      "equal MO.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.3)";
  }
  

  identity mo-ignore {
    base mo-base-type;
    description
      "ignore MO.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.3)";
  }

  identity mo-msb {
    base mo-base-type;
    description
      "MSB MO.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.3)";
  }

  identity mo-match-mapping {
    base mo-base-type;
    description
      "match-mapping MO.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.3)";
  }

  identity mo-rule-match {
    base schc:mo-base-type;
    description
      "Macthing operator return true, if the TV matches a rule
        keeping UP and DOWN direction.";
 }

 identity mo-rev-rule-match {
   base schc:mo-base-type;
   description
     "Macthing operator return true, if the TV matches a rule
      reversing UP and DOWN direction.";
 }
  //------------------------------
  // CDA type definition
  //------------------------------

  identity cda-base-type {
    description
      "Compression Decompression Actions. Specify the action to
       be applied to the field's value in a specific Rule.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.2)";
  }

  identity cda-not-sent {
    base cda-base-type;
    description
      "not-sent CDA.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.4)";
  }

  identity cda-value-sent {
    base cda-base-type;
    description
      "value-sent CDA.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.4)";
  }

  identity cda-lsb {
    base cda-base-type;
    description
      "Least Significant Bit (LSB) CDA.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.4)";
  }

  identity cda-mapping-sent {
    base cda-base-type;
    description
      "mapping-sent CDA.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.4)";
  }

  identity cda-compute {
    base cda-base-type;
    description
      "compute-* CDA.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.4)";
  }

  identity cda-deviid {
    base cda-base-type;
    description
      "DevIID CDA.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.4)";
  }

  identity cda-appiid {
    base cda-base-type;
    description
      "Application Interface Identifier (AppIID) CDA.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context
                Header Compression and Fragmentation (see
                Section 7.4)";
  }

 identity cda-compress-sent {
   base schc:cda-base-type;
   description
     "Send a compressed version of TV keeping UP and
      DOWN direction.";
 }

 identity cda-rev-compress-sent {
   base schc:cda-base-type;
   description
     "Send a compressed version of TV reversing UP and
      DOWN direction.";
 }

  // -- type definition

  typedef space-field-id-type {
    type identityref {
      base space-field-id-base-type;
    }
    description
      "Field ID generic type.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  typedef fid-type {
    type identityref {
      base fid-base-type;
    }
    description
      "Field ID type for regular protocol fields (IPv6, UDP, CoAP, etc.).
       Used in the regular-field case of compression-rule-entry.";
  }

  typedef space-id-type {
    type identityref {
      base space-id-base-type;
    }
    description
      "Space ID type for universal option spaces (CoAP options, etc.).
       Used in the universal-option case of compression-rule-entry.";
  }

  typedef fl-type {
    type identityref {
      base fl-base-type;
    }
    description
      "Function used to indicate Field Length.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  typedef di-type {
    type identityref {
      base di-base-type;
    }
    description
      "Direction in LPWAN network: up when emitted by the device,
       down when received by the device, or bi when emitted or
       received by the device.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  typedef mo-type {
    type identityref {
      base mo-base-type;
    }
    description
      "Matching Operator (MO) to compare field values with
       Target Values.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  typedef cda-type {
    type identityref {
      base cda-base-type;
    }
    description
      "Compression Decompression Action to compress or
       decompress a field.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  // -- FRAGMENTATION TYPE
  // -- fragmentation modes

  identity fragmentation-mode-base-type {
    description
      "Define the fragmentation mode.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  identity fragmentation-mode-no-ack {
    base fragmentation-mode-base-type;
    description
      "No-ACK mode.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  identity fragmentation-mode-ack-always {
    base fragmentation-mode-base-type;
    description
      "ACK-Always mode.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  identity fragmentation-mode-ack-on-error {
    base fragmentation-mode-base-type;
    description
      "ACK-on-Error mode.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  typedef fragmentation-mode-type {
    type identityref {
      base fragmentation-mode-base-type;
    }
    description
      "Define the type used for fragmentation mode in Rules.";
  }

  // -- Ack behavior

  identity ack-behavior-base-type {
    description
      "Define when to send an Acknowledgment.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  identity ack-behavior-after-all-0 {
    base ack-behavior-base-type;
    description
      "Fragmentation expects ACK after sending All-0 fragment.";
  }

  identity ack-behavior-after-all-1 {
    base ack-behavior-base-type;
    description
      "Fragmentation expects ACK after sending All-1 fragment.";
  }

  identity ack-behavior-by-layer2 {
    base ack-behavior-base-type;
    description
      "Layer 2 defines when to send an ACK.";
  }

  typedef ack-behavior-type {
    type identityref {
      base ack-behavior-base-type;
    }
    description
      "Define the type used for ACK behavior in Rules.";
  }

  // -- All-1 with data types

  identity all-1-data-base-type {
    description
      "Type to define when to send an Acknowledgment message.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  identity all-1-data-no {
    base all-1-data-base-type;
    description
      "All-1 contains no tiles.";
  }

  identity all-1-data-yes {
    base all-1-data-base-type;
    description
      "All-1 MUST contain a tile.";
  }

  identity all-1-data-sender-choice {
    base all-1-data-base-type;
    description
      "Fragmentation process chooses to send tiles or not in All-1.";
  }

  typedef all-1-data-type {
    type identityref {
      base all-1-data-base-type;
    }
    description
      "Define the type used for All-1 format in Rules.";
  }

  // -- RCS algorithm types

  identity rcs-algorithm-base-type {
    description
      "Identify which algorithm is used to compute RCS.
       The algorithm also defines the size of the RCS field.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  identity rcs-crc32 {
    base rcs-algorithm-base-type;
    description
      "CRC32 defined as default RCS in RFC 8724.  This RCS is
       4 bytes long.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  typedef rcs-algorithm-type {
    type identityref {
      base rcs-algorithm-base-type;
    }
    description
      "Define the type for RCS algorithm in Rules.";
  }

  // --------- COMPOUND ACK TYPE DEFINITION ---------

  identity bitmap-format-base-type {
    description
      "Define how the bitmap is formed in ACK messages. ";
    reference
       "RFC9441 Static Context Header Compression (SCHC)
                 Compound Acknowledgement (ACK)";
  }

  identity bitmap-RFC8724 {
    base bitmap-format-base-type;
    description
      "Bitmap by default as defined in RFC 8724.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation.
       RFC9441 Static Context Header Compression (SCHC)
                 Compound Acknowledgement (ACK)";
  }

  identity bitmap-compound-ack {
    base bitmap-format-base-type;
    description
      "Compound ACK allows several bitmaps in an ACK message.";
    reference
       "RFC9441 Static Context Header Compression (SCHC)
                 Compound Acknowledgement (ACK)";  }

  typedef bitmap-format-type {
    type identityref {
      base bitmap-format-base-type;
    }
    description
      "Type of bitmap used in Rules.";
    reference
       "RFC9441 Static Context Header Compression (SCHC)
                 Compound Acknowledgement (ACK)";
  }

  // --------  RULE ENTRY DEFINITION ------------

  grouping tv-struct {
    description
      "Defines the Target Value element.  If the header field
       contains a text, the binary sequence uses the same encoding.
       field-id allows the conversion to the appropriate type.";
    leaf index {
      type uint16;
      description
        "Index gives the position in the matching list.  If only one
         element is present, index is 0.  Otherwise, index is the
         order in the matching list, starting at 0.";
    }
    leaf value {
      type binary;
      description
        "Target Value content as an untyped binary value.";
    }
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  grouping compression-rule-entry {
    status deprecated;
    description
      "These entries define a compression entry (i.e., a line),
       as defined in RFC 8724.
 +-------+--+--+--+------------+-----------------+---------------+
 |Field 1|FL|FP|DI|Target Value|Matching Operator|Comp/Decomp Act|
 +-------+--+--+--+------------+-----------------+---------------+
       An entry in a compression Rule is composed of 7 elements:
       - Field ID: the header field to be compressed
       - Field Length : either a positive integer or a function
       - Field Position: a positive (and possibly equal to 0)
         integer
       - Direction Indicator: an indication in which direction the
         compression and decompression process is effective
       - Target Value: a value against which the header field is
         compared
       - Matching Operator: the comparison operation and optional
         associate parameters
       - Comp./Decomp. Action: the compression or decompression
         action and optional parameters

       Deprecated in favor of 'compression-rule-entry-universal', which
       adds support for Universal Options.";
    reference
      "RFC 9363 A YANG Data Model for Static Context Header
                Compression (SCHC)";
    leaf field-id {
      type schc:fid-type;
      mandatory true;
      description
        "Field ID, identify a field in the header with a YANG
         identity reference.";
    }
    leaf field-length {
      type union {
      type uint8;
      type schc:fl-type;
        }
      mandatory true;
      description
        "Field Length, expressed in number of bits if the length is
         known when the Rule is created or through a specific
         function if the length is variable.";
    }
    leaf field-position {
      type uint8;
      mandatory true;
      description
        "Field Position in the header is an integer.  Position 1
         matches the first occurrence of a field in the header,
         while incremented position values match subsequent
         occurrences.
         Position 0 means that this entry matches a field
         irrespective of its position of occurrence in the
         header.
         Be aware that the decompressed header may have
         position-0 fields ordered differently than they
         appeared in the original packet.";
    }
    leaf direction-indicator {
      type schc:di-type;
      mandatory true;
      description
        "Direction Indicator, indicate if this field must be
         considered for Rule selection or ignored based on the
         direction (bidirectional, only uplink, or only
         downlink).";
    }
    list target-value {
      key "index";
      uses tv-struct;
      description
        "A list of values to compare with the header field value.
         If Target Value is a singleton, position must be 0.
         For use as a matching list for the mo-match-mapping Matching
         Operator, index should take consecutive values starting
         from 0.";
    }
    leaf matching-operator {
      type schc:mo-type;
      must "../target-value or derived-from-or-self(.,
                                                   'mo-ignore')" {
        error-message
          "mo-equal, mo-msb, and mo-match-mapping need target-value";
        description
          "target-value is not required for mo-ignore.";
      }
      must "not (derived-from-or-self(., 'mo-msb')) or
            ../matching-operator-value" {
        error-message "mo-msb requires length value";
      }
      mandatory true;
      description
        "MO: Matching Operator.";
      reference
        "RFC 8724 SCHC: Generic Framework for Static Context Header
                  Compression and Fragmentation (see Section 7.3)";
    }
    list matching-operator-value {
      key "index";
      uses tv-struct;
      description
        "Matching Operator Arguments, based on TV structure to allow
         several arguments.
         In RFC 8724, only the MSB Matching Operator needs arguments
         (a single argument, which is the number of most significant
         bits to be matched).";
    }
    leaf comp-decomp-action {
      type schc:cda-type;
      must "../target-value or
                derived-from-or-self(., 'cda-value-sent') or
                derived-from-or-self(., 'cda-compute') or
                derived-from-or-self(., 'cda-appiid') or
                derived-from-or-self(., 'cda-deviid')" {
        error-message
          "cda-not-sent, cda-lsb, and cda-mapping-sent need
           target-value";
        description
          "target-value is not required for some CDA.";
      }
      mandatory true;
      description
        "CDA: Compression Decompression Action.";
      reference
        "RFC 8724 SCHC: Generic Framework for Static Context Header
                  Compression and Fragmentation (see Section 7.4)";
    }
    list comp-decomp-action-value {
      key "index";
      uses tv-struct;
      description
        "CDA arguments, based on a TV structure, in order to allow
         for several arguments.  The CDAs specified in RFC 8724
         require no argument.";
    }
  }

  grouping compression-rule-entry-universal {
    description
      "These entries define a compression entry (i.e., a line),
       as defined in RFC 8724. The format has been generalized
       to include universal options.

       An entry in a compression Rule is composed of 7 elements:
       - entry-index: position in the rule.
       - space field ID: the identityref of a specific field or 
         space ID followed by an universal value.
       - universal-value: the value associated to the space field ID to identify
         a specific field in the space. For regular fields, this
         value is 0 and the space field ID is the field ID. For
         universal options, the space field ID identifies the type
         of option and the universal value identifies the specific
         option.
       - Field Length : either a positive integer or a function
       - Field Position: a positive (and possibly equal to 0)
         integer
       - Direction Indicator: an indication in which direction the
         compression and decompression process is effective
       - Target Value: a value against which the header field is
         compared
       - Matching Operator: the comparison operation and optional
         associate parameters
       - Comp./Decomp. Action: the compression or decompression
         action and optional parameters
      ";

    leaf entry-index {
      type uint16;

      description 
        "Sequential position of the entry in the user-ordered Rule list.";
    }

    choice field-or-space {
      mandatory true;
      description
        "Distinguishes a regular protocol field from a universal option.
         Exactly one case must be present:
         - regular-field: a standard protocol field identified by a field ID
           (e.g., fid-ipv6-version, fid-udp-dev-port).
         - universal-option: a generic option identified by a space ID
           (e.g., space-id-coap) plus a numeric option value (e.g., the
           CoAP option number).";

      case regular-field {
        description
          "The entry refers to a regular protocol field.
           No universal-value is needed.";
        leaf field-id {
          type schc:fid-type;
          mandatory true;
          description
            "Field ID of a regular protocol field.";
        }
      }

      case universal-option {
        description
          "The entry refers to a universal option within a given space.
           Both space-id and universal-value are required.";
        leaf space-id {
          type schc:space-id-type;
          mandatory true;
          description
            "Space ID identifying the protocol option space
             (e.g., space-id-coap for CoAP options).";
        }
        leaf universal-value {
          type uint64;
          mandatory true;
          description
            "Numeric value identifying the specific option within the space
             (e.g., the CoAP option number).";
        }
      } // case universal-option
    } // choice field-or-space

    leaf field-length {
      type union {
      type uint8;
      type schc:fl-type;
        }
      mandatory true;
      description
        "Field Length, expressed in number of bits if the length is
         known when the Rule is created or through a specific
         function if the length is variable.";
    }
    leaf field-length-value {
      type uint16;

      description 
      "Argument used by Field Length functions. For
      fl-length-bytes and fl-length-bits, this value is the
      entry-index of the Rule entry whose numerical field value
      specifies the length.";    
      }
    leaf field-position {
      type uint8;
      mandatory true;
      description
        "Field Position in the header is an integer.  Position 1
         matches the first occurrence of a field in the header,
         while incremented position values match subsequent
         occurrences.
         Position 0 means that this entry matches a field
         irrespective of its position of occurrence in the
         header.
         Be aware that the decompressed header may have
         position-0 fields ordered differently than they
         appeared in the original packet.";
    }
    leaf direction-indicator {
      type schc:di-type;
      mandatory true;
      description
        "Direction Indicator, indicate if this field must be
         considered for Rule selection or ignored based on the
         direction (bidirectional, only uplink, or only
         downlink).";
    }
    list target-value {
      key "index";
      uses tv-struct;
      description
        "A list of values to compare with the header field value.
         If Target Value is a singleton, position must be 0.
         For use as a matching list for the mo-match-mapping Matching
         Operator, index should take consecutive values starting
         from 0.";
    }
    leaf matching-operator {
      type schc:mo-type;
      must "../target-value or derived-from-or-self(.,
                                                   'mo-ignore')" {
        error-message
          "mo-equal, mo-msb, and mo-match-mapping need target-value";
        description
          "target-value is not required for mo-ignore.";
      }
      must "not (derived-from-or-self(., 'mo-msb')) or
            ../matching-operator-value" {
        error-message "mo-msb requires length value";
      }
      mandatory true;
      description
        "MO: Matching Operator.";
      reference
        "RFC 8724 SCHC: Generic Framework for Static Context Header
                  Compression and Fragmentation (see Section 7.3)";
    }
    list matching-operator-value {
      key "index";
      uses tv-struct;
      description
        "Matching Operator Arguments, based on TV structure to allow
         several arguments.
         In RFC 8724, only the MSB Matching Operator needs arguments
         (a single argument, which is the number of most significant
         bits to be matched).";
    }
    leaf comp-decomp-action {
      type schc:cda-type;
      must "../target-value or
                derived-from-or-self(., 'cda-value-sent') or
                derived-from-or-self(., 'cda-compute') or
                derived-from-or-self(., 'cda-appiid') or
                derived-from-or-self(., 'cda-deviid')" {
        error-message
          "cda-not-sent, cda-lsb, and cda-mapping-sent need
           target-value";
        description
          "target-value is not required for some CDA.";
      }
      mandatory true;
      description
        "CDA: Compression Decompression Action.";
      reference
        "RFC 8724 SCHC: Generic Framework for Static Context Header
                  Compression and Fragmentation (see Section 7.4)";
    }
    list comp-decomp-action-value {
      key "index";
      uses tv-struct;
      description
        "CDA arguments, based on a TV structure, in order to allow
         for several arguments.  The CDAs specified in RFC 8724
         require no argument.";
    }
  }


  // --Rule nature

  identity nature-base-type {
    description
      "A Rule, identified by its RuleID, is used for a single
       purpose.  RFC 8724 defines 3 natures:
       compression, no-compression, and fragmentation.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation (see Section 6)";
  }

  identity nature-compression {
    base nature-base-type;
    description
      "Identify a compression Rule.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation (see Section 6)";
  }

  identity nature-no-compression {
    base nature-base-type;
    description
      "Identify a no-compression Rule.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation (see Section 6)";
  }

  identity nature-fragmentation {
    base nature-base-type;
    description
      "Identify a fragmentation Rule.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation (see Section 6)";
  }

  typedef nature-type {
    type identityref {
      base nature-base-type;
    }
    description
      "Defines the type to indicate the nature of the Rule.";
  }

  identity nature-management {
    base nature-base-type;
    description
      "Identify a management Rule. Management Rules use the
     compression Rule structure to operate on the SCHC Context.";
  /* Reference to be added when the Rule Management specification
     is available. */
  }

  grouping compression-content {
    status deprecated;
    list entry {
      status deprecated;
      must "derived-from-or-self(../rule-nature,
                                        'nature-compression')" {
        error-message "Rule nature must be compression";
      }
      key "field-id field-position direction-indicator";

      uses compression-rule-entry {
        status deprecated;
      }
      description
        "A compression Rule is a list of Rule entries, each
         describing a header field.  An entry is identified
         through a field-id, its position in the packet, and
         its direction.";
    }

    description
      "Define a compression Rule composed of a list of entries.
       Deprecated in favor of 'compression-content-universal', which
       adds support for Universal Options.";
    reference
      "RFC 9363 A YANG Data Model for Static Context Header
                Compression (SCHC)";
  }

  grouping compression-content-universal {
    list entry-universal {
      must "derived-from-or-self(../rule-nature,
                                  'nature-compression') or
            derived-from-or-self(../rule-nature,
                                  'nature-management')" {
        error-message
          "Rule nature must be compression or management";
      }
      key "entry-index";
      ordered-by user;

      uses compression-rule-entry-universal;
      description
        "A compression Rule is a list of Rule entries, each
         describing a header field or Universal Option. An entry
         is identified by its entry-index.";
    }

    description
      "Define a compression Rule composed of a list of entries,
       using the Universal-Option-aware 'compression-rule-entry-universal'.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  grouping fragmentation-content {
    description
      "This grouping defines the fragmentation parameters for
       all the modes (No ACK, ACK Always, and ACK on Error) specified
       in RFC 8724.";
    leaf fragmentation-mode {
      type schc:fragmentation-mode-type;
      must "derived-from-or-self(../rule-nature,
                                        'nature-fragmentation')" {
        error-message "Rule nature must be fragmentation";
      }
      mandatory true;
      description
        "Which fragmentation mode is used (No ACK, ACK Always, or
         ACK on Error).";
    }
    leaf l2-word-size {
      type uint8;
      default "8";
      description
        "Size, in bits, of the Layer 2 Word.";
    }
    leaf direction {
      type schc:di-type;
      must "derived-from-or-self(., 'di-up') or
            derived-from-or-self(., 'di-down')" {
        error-message
          "Direction for fragmentation Rules are up or down.";
      }
      mandatory true;
      description
        "MUST be up or down, bidirectional MUST NOT be used.";
    }
    // SCHC Frag header format
    leaf dtag-size {
      type uint8;
      default "0";
      description
        "Size, in bits, of the DTag field (T variable from
         RFC 8724).";
    }
    leaf w-size {
      when "derived-from-or-self(../fragmentation-mode,
                                'fragmentation-mode-ack-on-error')
            or
            derived-from-or-self(../fragmentation-mode,
                                'fragmentation-mode-ack-always') ";
      type uint8;
      description
        "Size, in bits, of the window field (M variable from
         RFC 8724).";
    }
    leaf fcn-size {
      type uint8;
      mandatory true;
      description
        "Size, in bits, of the FCN field (N variable from
         RFC 8724).";
    }
    leaf rcs-algorithm {
      type rcs-algorithm-type;
      default "schc:rcs-crc32";
      description
        "Algorithm used for RCS.  The algorithm specifies the RCS
         size.";
    }
    // SCHC fragmentation protocol parameters
    leaf maximum-packet-size {
      type uint16;
      default "1280";
      description
        "When decompression is done, packet size must not
         strictly exceed this limit, expressed in bytes.";
    }
    leaf window-size {
      type uint16;
      description
        "By default, if not specified, the FCN value is 2^w-size - 1.
         This value should not be exceeded.  Possible FCN values
         are between 0 and window-size - 1.";
    }
    leaf max-interleaved-frames {
      type uint8;
      default "1";
      description
        "Maximum of simultaneously fragmented frames.  Maximum value
         is 2^dtag-size.  All DTag values can be used, but more than
         max-interleaved-frames MUST NOT be active at any time.";
    }
    container inactivity-timer {
      leaf ticks-duration {
        type uint8;
        default "20";
        description
          "Duration of one tick in microseconds:
              2^ticks-duration/10^6 = 1.048s.";
      }
      leaf ticks-numbers {
        type uint16 {
          range "0..max";
        }
        description
          "Timer duration = ticks-numbers*2^ticks-duration / 10^6.";
      }

      description
        "Duration in seconds of the Inactivity Timer; 0 indicates
         that the timer is disabled.

         Allows a precision from microsecond to year by sending the
         tick-duration value. For instance:

        tick-duration: smallest value   <-> highest value

        20: 00y 000d 00h 00m 01s.048575<->00y 000d 19h 05m 18s.428159
        21: 00y 000d 00h 00m 02s.097151<->00y 001d 14h 10m 36s.856319
        22: 00y 000d 00h 00m 04s.194303<->00y 003d 04h 21m 13s.712639
        23: 00y 000d 00h 00m 08s.388607<->00y 006d 08h 42m 27s.425279
        24: 00y 000d 00h 00m 16s.777215<->00y 012d 17h 24m 54s.850559
        25: 00y 000d 00h 00m 33s.554431<->00y 025d 10h 49m 49s.701119

         Note that the smallest value is also the incrementation
         step.";
    }
    container retransmission-timer {
      leaf ticks-duration {
        type uint8;
        default "20";
        description
          "Duration of one tick in microseconds:
              2^ticks-duration/10^6 = 1.048s.";
      }
      leaf ticks-numbers {
        type uint16 {
          range "1..max";
        }
        description
          "Timer duration = ticks-numbers*2^ticks-duration / 10^6.";
      }
      when "derived-from-or-self(../fragmentation-mode,
                                'fragmentation-mode-ack-on-error')
            or
            derived-from-or-self(../fragmentation-mode,
                                'fragmentation-mode-ack-always') ";
      description
        "Duration in seconds of the Retransmission Timer.
         See the Inactivity Timer.";
    }
    leaf max-ack-requests {
      when "derived-from-or-self(../fragmentation-mode,
                                'fragmentation-mode-ack-on-error')
            or
            derived-from-or-self(../fragmentation-mode,
                                'fragmentation-mode-ack-always') ";
      type uint8 {
        range "1..max";
      }
      description
        "The maximum number of retries for a specific SCHC ACK.";
    }
    choice mode {
      case no-ack;
      case ack-always;
      case ack-on-error {
        leaf tile-size {
          when "derived-from-or-self(../fragmentation-mode,
                             'fragmentation-mode-ack-on-error')";
          type uint8;
          description
            "Size, in bits, of tiles.  If not specified or set to 0,
             tiles fill the fragment.";
        }
        leaf tile-in-all-1 {
          when "derived-from-or-self(../fragmentation-mode,
                             'fragmentation-mode-ack-on-error')";
          type schc:all-1-data-type;
          description
            "Defines whether the sender and receiver expect a tile in
             All-1 fragments or not, or if it is left to the sender's
             choice.";
        }
        leaf ack-behavior {
          when "derived-from-or-self(../fragmentation-mode,
                             'fragmentation-mode-ack-on-error')";
          type schc:ack-behavior-type;
          description
            "Sender behavior to acknowledge, after All-0 or All-1 or
             when the LPWAN allows it.";
        }

        leaf bitmap-format {
          when "derived-from-or-self(../fragmentation-mode,
                            'fragmentation-mode-ack-on-error')";
          type bitmap-format-type;
          default "bitmap-RFC8724";
          description
            "How the bitmaps are included in the SCHC ACK message.
             Defined in RFC 9441.";
          reference
            "RFC 9441 Static Context Header Compression (SCHC)
                      Compound Acknowledgement (ACK)";
        }

        leaf last-bitmap-compression {
          when "derived-from-or-self(../fragmentation-mode,
                            'fragmentation-mode-ack-on-error')";
          type boolean;
          default "true";
          description
            "When true, the ultimate bitmap in the SCHC ACK message
            can be compressed.  Default behavior from RFC 8724. 
            Defined in RFC 9441.";
          reference
            "RFC 8724 SCHC: Generic Framework for Static Context Header
                      Compression and Fragmentation.
             RFC 9441 Static Context Header Compression (SCHC)
                       Compound Acknowledgement (ACK)";
        }
      }
      description
        "RFC 8724 defines 3 fragmentation modes.";
    }
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }


  grouping management-content {
    description
      "This group contains parameters to control management procedure.";

    container guard-period {
      leaf ticks-duration {
        type uint8;
        default "20";
        description
          "Duration of one tick in microseconds:
              2^ticks-duration/10^6 = 1.048s.";
      }
      leaf ticks-numbers {
        type uint16 {
          range "0..max";
        }
        description
          "Timer duration = ticks-numbers*2^ticks-duration / 10^6.";
      }
    }
  }


  // Define RuleID.  RuleID is composed of a RuleID value and a
  // RuleID length

  grouping rule-id-type {
    leaf rule-id-value {
      type uint32;
      description
        "RuleID value.  This value must be unique, considering its
         length.";
    }
    leaf rule-id-length {
      type uint8 {
        range "0..32";
      }
      description
        "RuleID length, in bits.  The value 0 is for implicit
         Rules.";
    }
    description
      "A RuleID is composed of a value and a length, expressed in
       bits.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  // SCHC table for a specific device.

  container context {
    if-feature "management";
    uses management-content;
    description
      "Management-related parameters that apply to the whole SCHC
       context rather than to a single Rule.";
  }

  container schc {
    list rule {
      key "rule-id-value rule-id-length";
      uses rule-id-type;
      leaf rule-nature {
        type nature-type;
        mandatory true;
        description
          "Specify the Rule's nature.";
      }
      choice nature {
        case fragmentation {
          if-feature "fragmentation";
          uses fragmentation-content;
        }
        case compression {
          status deprecated;
          if-feature "compression";
          uses compression-content {
            status deprecated;
          }
        }
        case compression-universal {
          if-feature "compression or management";
          uses compression-content-universal;
        }
        description
          "A Rule is for compression, no-compression, fragmentation,
          or management.";
      }
      description
        "Set of compression, no-compression, or fragmentation
         Rules identified by their rule-id.";
    }
    description
      "A SCHC set of Rules is composed of a list of Rules used for
   compression, no-compression, fragmentation, or management.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }



    rpc duplicate-rule {
        input {
          container from {
            uses schc:rule-id-type;
            description 
              "Source Rule ID";
          }
          container to {
            uses schc:rule-id-type;
            description
              "Destination Rule ID";
          }
          leaf ipatch-sequence {
            type binary;

            description 
              "CBOR sequence for an CORECONF iPatch used to modify the 
               newly created Rule.
               This parameter is optional, and set by default to 0xF6 (CBOR null)";
          }
        }
        output {
          leaf status {
            type enumeration {
              enum success-with-validation {
                value 0;
              }
              enum success-not-validated {
                value 1;
              }
              enum to-rule-already-exists {
                value -1;
              }
              enum from-rule-not-found {
                value -2;
              }
              enum invalid-ipatch {
                value -3;
              }
            }
            description 
              "Return the status of the RPC. TO BE DEFINED MORE PRECISELY";
          }
        }
        description 
          "This RPC duplicate a rule, from a existing one given in leaf 'from' to a new
          non existing rule defined by 'to'. The content of the new rule may be updated
          with a iPatch, where the CORECONF payload contained in 'ipatch-sequence'.";
      }
}
~~~
{: #fig-full-module title="Working Copy of the ietf-schc YANG Module"}

# Security Considerations

TODO Security

# IANA Considerations

TODO: register the (possibly updated) YANG module and namespace URI
with IANA, referencing this document instead of RFC 9363.

--- back

# Acknowledgments
{:numbered="false"}

This document reuses text and structure from RFC 9363. The authors
thank the contributors and reviewers of that document.

The authors also thank Claude (Anthropic) for rewriting parts of
this document in more correct English and for checking its
consistency.
