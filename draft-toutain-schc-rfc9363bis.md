---
title: "A YANG Data Model for Static Context Header Compression (SCHC), bis"
abbrev: "SCHC YANG Model bis"
category: std
docname: draft-toutain-schc-rfc9363bis-00
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
  RFC7252:
  RFC7641:
  RFC7959:
  RFC7967:
  RFC8407:
  I-D.ietf-schc-universal-option:
  I-D.ietf-schc-8824-update:
  I-D.ietf-schc-icmpv6-compression:
  I-D.ietf-core-oscore-key-update:

--- abstract

This document obsoletes RFC 9363, "A YANG Data Model for Static
Context Header Compression (SCHC)". It redefines the SCHC
compression rules to provide more flexibility through the use of
Universal Options, which allow identifiers to be added to or
removed from a Rule Description depending on the Universal Options
in use. This document also introduces new length-related Matching
Operators (MOs) and Compression/Decompression Actions (CDAs),
defines the RPCs used for SCHC management, and specifies a
mechanism for the manual allocation of YANG Schema Item
iDentifiers (SIDs).

--- middle

# Introduction

RFC 9363 {{RFC9363}} defines a YANG data model for the Static
Context Header Compression (SCHC) {{RFC8724}}. This document
updates that data model.

TODO: describe the motivation for this bis document (what changed
since RFC 9363, why an update is needed, and what stays compatible).

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

Two problems were identified with this approach. First, these keys
remain large, which may be a penalty when Rule management is used
(see {{rule-management}}). Second, this key-based approach does not
guarantee that entries appear in the same order as the
corresponding fields in the header, whereas RFC 8724 {{RFC8724}}
mandates that order and recent implementations have shown it to be
efficient.

This document redefines the whole compression Rule entry structure
to provide a uniform way to reference a field entry. A new
"entry-index" leaf is introduced, incremented sequentially for each
entry, which becomes the key of the new "entry-universal" list; the
"entry-universal" list is defined as "ordered-by user" so that entry order
can still convey the header field order (addressing the second
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
+--:(compression-universal) {compression}?
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

TODO: give a worked example (e.g. a CoAP option) showing how a Rule
entry is expressed with a Universal Option instead of a per-option
FID.

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

The working copy of the module has been corrected accordingly: the
twenty identities are present, each with "status deprecated;".

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

YANG (RFC 7950, Section 7.21.2) discourages the use of a definition
by adding a "status deprecated;" substatement to it. Applied to
"fid-coap-option-uri-path":

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

TODO: apply the same pattern to the other nineteen "fid-coap-option-*"
identities listed in {{tbl-coap-option-fids}}.

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
Partial IV or nonce field.

TODO: give a worked example (e.g. the OSCORE Partial IV, whose
length is carried in "fid-coap-option-oscore-flags-n") showing the
"field-length"/"field-length-value" pair in a full Rule entry.

## Other Changes

"fl-variable-bits" generalizes RFC 8824's "fl-variable" to bit-level
variable-length fields.

TODO: list and describe other, smaller changes to the module that do
not warrant their own subsection (e.g. the new "mo-rule-match",
"mo-rev-rule-match", "cda-compress-sent", "cda-rev-compress-sent",
and the generic "fid-unused"/"fid-payload" Field IDs).

# Rule Management {#rule-management}

The single-leaf "entry-index" key introduced in {{universal-options}}
is significantly cheaper to reference than the RFC 9363 compound key
when a management operation needs to target a specific Rule entry,
e.g. a CORECONF iPatch carried in the "duplicate-rule" RPC.

TODO: introduce Rule management (the "management" feature, the
"nature-management" Rule nature, and the "duplicate-rule" RPC), and
elaborate on how "entry-index" is used in management operations.

# Changes from RFC 9363

This document makes the following changes to the SCHC YANG data
model defined in RFC 9363 {{RFC9363}}:

* Compression rule entries are redefined to introduce more
  flexibility through the use of Universal Options
  {{I-D.ietf-schc-universal-option}}: a "compression-rule-entry" now
  chooses between a "regular-field" (a "field-id" as in RFC 9363,
  e.g. "fid-ipv6-version") and a "universal-option" (a "space-id",
  e.g. "space-id-coap", plus a numeric "universal-value" identifying
  the option within that space). This lets identifiers be added to,
  or removed from, a Rule Description as the set of Universal
  Options in use evolves, without requiring a new Field ID for every
  protocol option.

* New Field Length (FL) functions are introduced: "fl-length-bytes"
  and "fl-length-bits" generalize "fl-token-length" by returning the
  length of a field (in bytes or in bits, respectively) designated
  by its "entry-index", and "fl-variable-bits" generalizes
  "fl-variable" to bit-level variable-length fields.

* New Matching Operators ("mo-rule-match", "mo-rev-rule-match") and
  Compression/Decompression Actions ("cda-compress-sent",
  "cda-rev-compress-sent") are introduced, allowing a Target Value
  to be matched, or a compressed value to be sent, while keeping or
  reversing the Up/Down direction.

* A "management" feature and Rule nature ("nature-management") are
  introduced, together with the "duplicate-rule" RPC, used to
  duplicate an existing Rule under a new RuleID, optionally modified
  through a CORECONF iPatch.

* A mechanism for the manual allocation of YANG Schema Item
  iDentifiers (SIDs) is defined, so that SIDs already assigned in a
  previous revision of the module remain stable even when items are
  added or deprecated (e.g. the twenty per-option CoAP FIDs
  deprecated in favor of Universal Options).

TODO: expand each item above with a short rationale and a pointer
to the corresponding section of this document. Confirm whether the
following model additions, also present in the working copy of the
module but not explicitly requested for this abstract, are in scope
for this bis document or belong in a separate draft:

* Field IDs for ICMPv6 {{I-D.ietf-schc-icmpv6-compression}}.
* Field IDs for CoAP/OSCORE {{RFC8613}}, including the KUDOS
  suboptions {{I-D.ietf-core-oscore-key-update}}
  {{I-D.ietf-schc-8824-update}}.
* Compound ACK bitmap format support ({{RFC9441}}).
* Generic "fid-unused" and "fid-payload" Field IDs.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# SCHC YANG Data Model

This section describes the "ietf-schc" YANG module. It follows the
structure of RFC 9363 {{RFC9363}}, Section 4, updated as described
in {{changes-from-rfc-9363}}.

TODO: bring over / update the narrative text, tree diagrams, and
examples from RFC 9363 Section 4 for each subsection below, checking
against RFC 9363 that nothing already defined there is dropped.

## Features

Carried over unchanged from RFC 9363: "compression", "fragmentation".
New in this document: "management" (guards the "management" Rule
nature and the "duplicate-rule" RPC).

## Field ID (FID), Space ID, and Field Length (FL) Identities {#fl-identities}

* "fid-base-type" and its derived identities: unchanged from
  RFC 9363 for IPv6 ("fid-ipv6-*"), UDP ("fid-udp-*"), and CoAP
  ("fid-coap-*", including the OSCORE suboptions).

  TODO: confirm every RFC 9363 "fid-*" identity is still present
  and unchanged (name, base, description) in the working module.

* "space-field-id-base-type": new common base for "fid-base-type"
  and the new "space-id-base-type", used to unify regular fields
  and Universal Options in "compression-rule-entry" (see
  {{compression-rule-entry}}).

* "space-id-base-type" and its derived identities (e.g.
  "space-id-coap"): new, identify a Universal Option space; used
  together with a numeric "universal-value" in place of a Field ID.

* "fl-base-type" and its derived identities: "fl-variable" and
  "fl-token-length" are unchanged from RFC 9363. New identities:
  "fl-variable-bits", "fl-length-bytes", "fl-length-bits".

## Matching Operators and Compression/Decompression Actions

* "mo-base-type": "mo-equal", "mo-ignore", "mo-msb", and
  "mo-match-mapping" are unchanged from RFC 9363. New identities:
  "mo-rule-match", "mo-rev-rule-match".

* "cda-base-type": "cda-not-sent", "cda-value-sent", "cda-lsb",
  "cda-mapping-sent", "cda-compute", "cda-deviid", and "cda-appiid"
  are unchanged from RFC 9363. New identities: "cda-compress-sent",
  "cda-rev-compress-sent".

## Compression Rule Entry {#compression-rule-entry}

See {{universal-options}} for the redefinition of the compression
Rule entry key (the "entry-index" leaf) and of the "field-or-space"
choice, and for why this is introduced as a new, parallel
"compression-rule-entry-universal"/"compression-content-universal"/"compression-universal"
structure rather than by modifying the RFC 9363 structure in place.

TODO: describe the migration path from an RFC 9363 Rule (using the
deprecated "compression" case) to the new "compression-universal" case.

## Fragmentation Rule Content

Unchanged from RFC 9363: fragmentation modes, timers, and
Compound ACK parameters ({{RFC9441}}).

TODO: confirm against RFC 9363 that no fragmentation node was
dropped.

## Management Rule Content

New "management-content" grouping (guard-period timer), used by the
new "nature-management" Rule nature and gated by the "management"
feature.

## RPCs

New "duplicate-rule" RPC: duplicates an existing Rule (input
"from") into a new RuleID (input "to"), optionally modified via a
CORECONF iPatch ("ipatch-sequence"), and returns a "status".

TODO: describe the RPC's semantics and error cases in prose.

## Manual SID Allocation

TODO: describe the manual SID allocation mechanism: SIDs are first
generated automatically (e.g. via "pyang --sid-generate-file"), then
remapped to a stable, manually curated allocation table (module,
namespace, identifier) so that previously published SIDs do not
shift when the module evolves (fields added, removed, or
deprecated). Reference {{RFC9595}} for the SID file format and
describe the augmentation used to carry type information per SID
item.

## Full YANG Module

TODO: insert the complete, up-to-date "ietf-schc" YANG module here
once the changes above are finalized and validated with "pyang".

~~~ yang
module ietf-schc {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-schc";
  prefix schc;

  /* TODO: insert full module body */
}
~~~

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
