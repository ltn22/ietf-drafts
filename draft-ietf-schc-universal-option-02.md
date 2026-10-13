---
title: "Options representation in SCHC YANG Data Models"
abbrev: "SCHC Universal Options"
docname: draft-ietf-schc-universal-option-02
category: std
submissiontype: IETF
v: 3
area: Internet
workgroup: SCHC
keyword:
 - SCHC
 - CoAP
 - options
 - YANG
author:
 -
    ins: Q. Lampin
    name: Quentin Lampin
    org: Orange
    email: quentin.lampin@orange.com
 -
    ins: A. Minaburo
    name: Ana Minaburo
    org: Consultant
    street: Rue de Rennes
    city: Cesson-Sevigne
    code: "35510"
    country: France
    email: anaminaburo@gmail.com
 -
    ins: M. Tiloca
    name: Marco Tiloca
    org: RISE AB
    street: Isafjordsgatan 22
    city: Kista
    code: SE-16440
    country: Sweden
    email: marco.tiloca@ri.se
 -
    ins: L. Toutain
    name: Laurent Toutain
    org: IMT Atlantique
    street: "CS 17607, 2 rue de la Chataigneraie"
    city: Cesson-Sevigne Cedex
    code: "35576"
    country: France
    email: Laurent.Toutain@imt-atlantique.fr

normative:
  RFC2119:
  RFC8174:
  RFC7252:
  RFC8724:
  RFC9363:
  I-D.toutain-schc-rfc9363bis:

informative:
  RFC7950:
  RFC8407:
  RFC8613:
  RFC8824:
  I-D.ietf-schc-8824-update:
  I-D.ietf-core-oscore-key-update:
  I-D.lampin-lpwan-schc-considerations:
  GPC-SPE-207:
    title: "Remote Application Management over CoAP - Amendment M v1.0"
    author:
      org: GlobalPlatform
    target: https://globalplatform.org/specs-library/amendment-m-remote-application-mgmt-over-coap/

--- abstract

Keeping option identifiers in SCHC Rules simplifies interoperability and the evolution of SCHC compression when a protocol introduces options that the current SCHC implementation does not know.
This document explains why the Field Identifiers (FIDs) defined for CoAP options in RFC 9363 do not scale, and describes how Rule entries refer to an option by the pair (option space, option number), called a Universal Option.
It defines how entries are indexed, ordered and referenced from one another, and how existing Rules are migrated.
The YANG definitions supporting this mechanism are part of the module specified in draft-toutain-schc-rfc9363bis, which is intended to be incorporated into a revision of RFC 9363.
This document does not define a YANG module.

--- middle

# Introduction {#intro}

The Static Context Header Compression (SCHC) framework {{RFC8724}} compresses headers with Rules shared by two end-points.
The YANG data model {{RFC9363}} identifies each field of a Rule with a Field ID (FID), an identity defined in the module.
CoAP options {{RFC7252}} are handled in the same way, through {{RFC8824}}: {{RFC9363}} defines one FID for each option known when it was published.

This works in static deployments, where the set of options is known in advance and rarely changes.
It fails when an option is defined after a SCHC implementation has been deployed, or when a private option is used: there is no FID for it, and nothing guarantees that two implementations that each create one will choose the same.

This document keeps, in the Rule, the identifier that the protocol itself uses for the option.
Since several protocols define options, each with its own numbering, the option number is qualified by an option space.
The pair is called a Universal Option.
It lets any option of any protocol be referenced in a Rule without allocating a dedicated FID.

Previous revisions of this document defined a separate YANG module that augmented the module of {{RFC9363}} with a new list for options.
The discussions in the working group showed that this approach does not keep the entries in the order of the header fields, and imposes a large compound key.
All the changes to the data model, including those needed by Universal Options, are therefore collected in {{I-D.toutain-schc-rfc9363bis}}.
This document and {{I-D.toutain-schc-rfc9363bis}} are meant to be read together:

* {{I-D.toutain-schc-rfc9363bis}} contains the YANG module, with all the identities, leaves and lists used here.
* This document gives the motivation, the semantics of the Rule entries, the constraints on their order, and the procedures to migrate existing Rules.

# Terminology {#terminology}

{::boilerplate bcp14-tagged}

This document reuses the terminology of {{RFC8724}} and {{RFC9363}}, and adds the following terms:

Option Space:
: The numbering space of the options of one protocol, identified in the data model by a Space ID.

Universal Option:
: An option referenced in a Rule entry by the pair (Space ID, option number), where the option number is the one assigned by the protocol.

Entry Index:
: The integer that identifies an entry in a compression Rule. It is the key of the entry in the data model.

# Problem Statement {#problem}

## Option Representation {#problem-repr}

In {{RFC9363}}, CoAP options are predefined FIDs.
Each option is abstracted away from its protocol identifier and receives a SCHC-specific identifier.
The mapping between the option number defined by the protocol and the FID is neither standardized nor predictable.
When a CoAP option is defined after a SCHC implementation has been deployed, the implementation has no way to give it an identifier, nor to tell another implementation which identifier it chose.

## Rule Management {#problem-mgmt}

The limitation is especially visible when Rules are managed between two SCHC end-points ({{fig-mgmt}}).

~~~
                       rule mngt
                      +----------+
                      |          |
                      v          v
         A <-------> S1 <~~~~~~> S2 <-----> B
~~~
{: #fig-mgmt title="Rule Management between two SCHC end-points"}

Device A generates CoAP packets that may carry options, including recently defined or private ones.
S1 and S2 compress and decompress the traffic with Rules they share.
S1 can parse a new option, since the CoAP option encoding is generic, and could derive a more specific Rule to compress it better.
It then has to name that option in the Rule in a way that S2 understands.
For a known option such as Uri-Path, {{RFC9363}} provides an identity that both ends know.
For a new or private option, no identity exists.

## Interoperability Consequences {#problem-interop}

The disconnection between protocol option identifiers and SCHC FIDs has four consequences:

* New options cannot be compressed efficiently without updating the SCHC implementation.
* Two implementations may assign different FIDs to the same new option, which makes Rule exchange difficult.
* Implementations have to maintain mappings between protocol options and FIDs, and these mappings may differ.
* Deploying SCHC in environments where protocols evolve requires frequent software updates.

# Approaches Considered {#approaches}

Two approaches were considered to keep protocol option identifiers in Rules.

## Syntactic Representation {#syntactic}

The syntactic approach {{I-D.lampin-lpwan-schc-considerations}} follows the wire format.
Each CoAP option is split into three fields: the option number (or delta), the length, and the value.
Any option, known or not, can then be handled by three generic FIDs.

The cost is a factor three on the number of entries needed to describe an option, and a less efficient compression when the option has to be sent: the length is carried twice, once as a field and once as the length prefix of the variable-length value.
For a 4-byte option value, the residue is 6.5 bytes, against 4.5 bytes with the approach described next.
{{evaluation}} gives measurements on a complete example.

## Semantic Representation with Option Identifiers {#semantic}

The semantic approach keeps one entry per option and replaces the FID by the protocol's option identifier, qualified by the Option Space.
The Option Space is needed because protocols reuse numbers: option 8 is Location-Path in CoAP and Timestamp in TCP.
The compression efficiency is the one of the FID-based representation, and any option can be named.
This is the approach retained in this document and called Universal Options.

# Universal Options {#universal}

## Option Spaces {#spaces}

An Option Space is identified in the data model by a Space ID.
The module of {{I-D.toutain-schc-rfc9363bis}} defines a base identity for Space IDs and one derived identity per protocol.
At the time of writing, a single Space ID is defined, for the CoAP option space.
The option number is then the number registered by CoAP {{RFC7252}}.

A document that makes SCHC applicable to another protocol with options defines a new Space ID by deriving an identity from the Space ID base identity, in the same way as the existing one.
That document specifies:

* the protocol, and the registry that assigns the option numbers;
* how the options are parsed in the header, so that they can be matched against Rule entries.

The option number is carried as an unsigned integer, so that numbering spaces wider than the one of CoAP can be accommodated.

## Rule Entries {#entries}

A compression Rule describes the header as a list of entries.
For Universal Options, the data model provides a new list of entries, which is described in {{I-D.toutain-schc-rfc9363bis}} and replaces the list of {{RFC9363}} (see {{deprecation}}).
An entry identifies the header element it describes with a choice:

* a regular field, identified by a Field ID, as in {{RFC9363}};
* an option, identified by the pair made of a Space ID and the option number (the Universal Option).

The other elements of an entry keep their semantics from {{RFC8724}}: field length, field position, direction indicator, Target Value, Matching Operator, Compression/Decompression Action, and their arguments.
One element is added, to carry the argument of the Field Length functions of {{fl}}.

### Entry Index {#entry-index}

The entries of a Rule are keyed by a single integer, the Entry Index.
The list of entries is ordered by the user, and Entry Index values are assigned sequentially according to the order of the entries, starting from 0.
This is the numbering used in the examples of this document.

A one-element key makes an entry cheap and unambiguous to reference from elsewhere in the same Rule (see {{fl}}) or from a management operation (see {{mgmt}}).
It replaces the compound keys of {{RFC9363}} (field identifier, position and direction) and of previous revisions of this document (Space ID, option number, position and direction).

### Order of the Entries {#order}

{{RFC8724}} requires that the Field Descriptors of a Rule be listed in the order in which the fields appear in the packet header.
This document makes the requirement explicit for the data model:

* The entries of a compression Rule MUST be listed in the order of the corresponding fields in the packet header.
* Because a list entry is ordered by the Entry Index, the order of the entries conveys the header field order, and the order MUST be preserved when a Rule is transmitted from one end-point to another.
* The residues of the entries are serialized in the order of the entries.

For CoAP, options are sent in increasing option number {{RFC7252}}, so entries describing options appear in that order, and entries describing a repeated option follow each other.

Previous revisions required that entries for regular fields be serialized before entries for options.
That requirement does not exist any more: regular fields and options are in a single list, in header order, so an option can be placed between two regular fields when the protocol header requires it.

### Field Position of Repeated Options {#position}

The field position keeps its meaning of {{RFC8724}}: it distinguishes the successive occurrences of the same element in a header.
{{fig-uri}} shows a Rule fragment for a URI with two path elements (option 11) and two query elements (option 15).

| # | Field | FL | FP | DI | TV | MO | CDA |
|:--|:--|:--|:--|:--|:--|:--|:--|
| 0 | space-id-coap(11) | var | 1 | Up | value | equal | not-sent |
| 1 | space-id-coap(11) | var | 2 | Up | - | ignore | value-sent |
| 2 | space-id-coap(15) | var | 1 | Up | value | equal | not-sent |
| 3 | space-id-coap(15) | var | 2 | Up | value | equal | not-sent |
{: #fig-uri title="Rule fragment with Universal Options for Uri-Path (11) and Uri-Query (15)"}

The names are the ones of the data model, without their prefix.
The first column, not defined in {{RFC8724}}, is the Entry Index.

## Referencing Another Entry in the Field Length {#fl}

In {{RFC8824}}, a specific Field Length function gives the length of the Token from the value of the Token Length field.
{{I-D.ietf-schc-8824-update}} does the same for the OSCORE Partial IV.
This does not scale: a new function would be needed for every field whose length is carried in another field.

The module of {{I-D.toutain-schc-rfc9363bis}} defines two functions that take the Entry Index of the entry holding the length as argument:

* `fl-length-bytes`: the value of the referenced entry is the length of the field, in bytes.
* `fl-length-bits`: the value of the referenced entry is the length of the field, in bits.

The argument is carried in the `field-length-value` element of the entry.
The previous function for the Token is equivalent to `fl-length-bytes` pointing to the entry that describes the CoAP Token Length.

The entry that is referenced:

* MUST precede the entry that refers to it, since its value is needed to parse the residue of the latter;
* MUST apply in the same directions as the entry that refers to it.

{::comment}
AUTHORS: neither constraint is stated in toward-00; confirm and, if kept, add them to toward as well.
{:/comment}

Using this mechanism requires that the field carrying the length be an entry of its own.
This is why the Flags fields of the OSCORE option {{RFC8613}} and of the KUDOS option {{I-D.ietf-core-oscore-key-update}} are split in {{I-D.toutain-schc-rfc9363bis}}, so that the length indicator (the `n` bits for OSCORE) is a separate entry.
KUDOS needs a further step: the Nonce length is the value `m` plus one, and `fl-length-bytes` and `fl-length-bits` use the referenced value as is (see {{open}}).

## Example {#example}

{{fig-oscore}} shows a Rule (Rule ID 2, on 5 bits) that compresses the outer CoAP header of a CoAP/OSCORE message {{RFC8613}} in both directions.
The server is on the Application side and the client on the Device side.
The request carries an OSCORE option with the security parameters, and the response an empty OSCORE option.

| # | Field | FL | FP | DI | TV | MO | CDA |
|:--|:--|:--|:--|:--|:--|:--|:--|
| 0 | coap-version | 2 | 1 | Bi | 01 | equal | not-sent |
| 1 | coap-type | 2 | 1 | Bi | - | ignore | value-sent |
| 2 | coap-tkl | 4 | 1 | Bi | - | ignore | value-sent |
| 3 | coap-code | 8 | 1 | Up | 02 | equal | not-sent |
| 4 | coap-code | 8 | 1 | Dw | 44 | equal | not-sent |
| 5 | coap-mid | 16 | 1 | Bi | 00 | ignore | value-sent |
| 6 | coap-token | length-bytes(2) | 1 | Bi | - | ignore | value-sent |
| 7 | coap-option-oscore-flags-flagbits | 5 | 1 | Up | 01 | equal | not-sent |
| 8 | coap-option-oscore-flags-n | 3 | 1 | Up | 01 | equal | not-sent |
| 9 | coap-option-oscore-piv | length-bytes(8) | 1 | Up | - | ignore | value-sent |
| 10 | coap-option-oscore-kidctx | 0 | 1 | Up | 00 | equal | not-sent |
| 11 | coap-option-oscore-kid | var | 1 | Up | - | ignore | value-sent |
| 12 | space-id-coap(9) | 0 | 1 | Dw | - | equal | not-sent |
{: #fig-oscore title="Compression Rule for the outer header of a CoAP/OSCORE message"}

The Rule illustrates the mechanisms of this document:

* The Token length uses `fl-length-bytes` with argument 2, which refers to entry 2, `coap-tkl`.
* The Partial IV length uses `fl-length-bytes` with argument 8, which refers to entry 8, `coap-option-oscore-flags-n`.
* The empty OSCORE option of the response has no sub-field entries: it is described as a Universal Option, with option number 9 in the CoAP Option Space.

# Deprecation of the Per-Option CoAP FIDs {#deprecation}

## Status and Timeline {#dep-status}

Universal Options make the twenty per-option FIDs of {{RFC9363}} redundant.
{{I-D.toutain-schc-rfc9363bis}} keeps them in the module, marked as deprecated, and gives for each of them the pair (Space ID, option number) that replaces it.
It also keeps the entry list of {{RFC9363}} and the structures that use it, marked as deprecated, and adds the new list of entries beside them.

The identities and the list are not removed, for the following reasons:

* {{RFC7950}} states that obsolete definitions MUST NOT be removed from published modules, and that the key of a list cannot be changed in place.
* {{RFC8407}} recommends that an object stay deprecated for at least one year before it becomes obsolete, and that its status not change directly from current to obsolete.

The deprecated identities MUST NOT be used in the new list of entries.
They remain usable only in Rules that use the deprecated list.
The sub-field FIDs of the OSCORE and KUDOS options are not deprecated: they split one option into parts and are not stand-ins for an arbitrary option.

## Converting an Existing Rule {#converting}

A Rule that uses the deprecated list is converted to the new list as follows:

1. Order the entries as the corresponding fields appear in the packet header, field positions in increasing order. The deprecated list is not ordered by the user, so its order cannot be taken as the header order.
2. For an entry that uses one of the twenty deprecated FIDs, replace the FID by the Space ID of CoAP and the option number given for that FID in {{I-D.toutain-schc-rfc9363bis}}.
3. For any other entry, keep the FID.
4. Assign the Entry Index sequentially, from 0, following the order obtained in step 1.
5. Copy the field length, field position, direction indicator, Target Values, Matching Operator, Compression/Decompression Action and their arguments. An entry that uses the Token Length function MAY be converted to `fl-length-bytes`, with the Entry Index of the CoAP Token Length entry as argument.

A converted Rule produces the same residue as the original, provided that the original listed its entries in header order.

## Coexistence {#coexistence}

The deprecated list and the new list are two cases of the Rule nature, and Rule IDs are shared between Rules of both cases.
A SCHC context can therefore hold Rules of both kinds during a migration.
An end-point that only implements {{RFC9363}} does not understand the new list; how end-points learn that they can use it is out of scope of this document (see {{open}}).
Management Rules use only the new list.

# Rule Management {#mgmt}

In the scenario of {{fig-mgmt}}, S1 can now describe any option it sees in a Rule it derives, with a Space ID and the option number, and S2 knows what is meant: both use the identifier of the protocol.
This applies to recently defined options, and to private ones.

The Entry Index gives management operations a short way to designate one entry of a Rule, a single integer instead of a compound key.
The management structures and operations are described in {{I-D.toutain-schc-rfc9363bis}} and in separate documents.

# Open Issues {#open}

This section is to be removed before publication.

* KUDOS Nonce: its length is `m + 1`, where `m` is carried in the Flags field. `fl-length-bytes` and `fl-length-bits` cannot express the offset. A function with an offset argument, or a dedicated treatment, is needed.
* OSCORE KID Context: when present, it is carried with its own length byte. Keeping it as a single variable-length entry sends the length twice. Splitting it into a length entry and a value entry that uses `fl-length-bytes` avoids this.
* Capability signalling: how an end-point announces that it supports the new list of entries, and what a management peer does when it does not.
* Option Space registration: whether a registry is needed, or whether defining an identity in the document that adds the protocol is enough.
* Whether this document is marked as updating {{RFC9363}}, or only as input to its revision.

# Security Considerations {#security}

Universal Options do not change how a value is matched or reconstructed.
A Rule that elides an option with the `equal` Matching Operator and the `not-sent` action reproduces exactly the value in the Target Value, whether or not the implementation knows the semantics of the option.
Compressing an unknown option therefore brings no risk that known options do not already have.

An end-point that parses options it does not know MUST bound the parsing (number of options, lengths) as for any other option, since an attacker can build packets with arbitrary options.

The Entry Index makes Rules able to refer to one another's entries.
An implementation MUST reject a Rule in which the entry referenced by a Field Length function does not exist, does not precede the entry that refers to it, or is the entry itself, since such a Rule cannot be decompressed.

Rules can be installed by a management peer.
A peer that is allowed to install Rules can describe any option, so the access control and authentication of management operations apply, and are described outside this document.

The deprecated identities remain usable.
Keeping them does not add a threat, but delays the removal of the code that handles them.

{::comment}
AUTHORS: first draft of this section, to be reviewed by the working group.
{:/comment}

# IANA Considerations {#iana}

This document has no IANA actions.
The registration of the YANG module and of its SIDs is specified in {{I-D.toutain-schc-rfc9363bis}}.

--- back

# Evaluation of the Approaches {#evaluation}

Previous revisions compared the approaches on a CoAP request with several options, one of which is not standard (SCP82-Param, option 2055 {{GPC-SPE-207}}):

~~~
CON 0x0001 GET
> Uri-path : accelerometers
> Uri-path : maximum
> Uri-query : date=today
> Uri-query : unit=m/s^2
> Accept : 60
> No-Response : 2
> SCP82-Param : TLV
~~~
{: #fig-test title="Reference CoAP message"}

The Rule sends the residues of Uri-Path, Uri-Query and SCP82-Param, and elides the other fields.
Three measures are given: the size of the CBOR-serialized Rule, the size of the CORECONF query payload that reads the Target Value of the Accept option, and the size of the SCHC packet.

| | RFC 9363 | Universal Options (-01) | Merged | Ordered SIDs | Syntactic | entry-universal |
|:--|:--|:--|:--|:--|:--|:--|
| Rule (bytes) | 357 | 481 | 400 | 376 | 718 | TBD |
| Query (bytes) | 14 | 15 | 15 | 15 | 14 | TBD |
| SCHC packet (bytes) | 49 | 49 | 49 | 49 | 52 | TBD |
{: #tab-eval title="Measures from revision -01, and the new representation"}

The values of revision -01 were obtained with the module of that revision and the SIDs described there.
The last column is the representation of this document, with the module and the SID allocation of {{I-D.toutain-schc-rfc9363bis}}, and has to be measured.
The CBOR serializations and the SID allocation tables of revision -01 are not repeated here, since they depend on a module that this document no longer defines.

{::comment}
AUTHORS: restore the detailed serializations from -01 (Appendix C and D) once the new column is measured.
{:/comment}

# Changes Since -01 {#changes}

This section is to be removed before publication.

* The YANG module `ietf-schc-opt` is removed. The definitions are in {{I-D.toutain-schc-rfc9363bis}}.
* The augmentation with a list of option entries keyed by Space ID, option number, position and direction is replaced by a single list of entries keyed by an Entry Index, which includes regular fields and options.
* The requirement that regular entries be serialized before option entries is removed. The order is the header order.
* The requirement for an "ordered-by user" statement, and the pointer to the SCHC architecture document, are replaced by the order requirements of {{order}}.
* New sections: Field Length functions that refer to an entry ({{fl}}), an OSCORE example ({{example}}), the status and timeline of the deprecation, the conversion procedure and the coexistence of the two lists ({{deprecation}}), Security Considerations and Open Issues.
* The detailed CBOR examples are replaced by a summary, with a column to be measured for the new representation.
* The option number is an unsigned integer without a stated width, instead of a 32-bit integer.

# Acknowledgments
{:numbered="false"}

The authors sincerely thank the members of the SCHC working group who reviewed and discussed previous revisions of this document.

This work was supported by the Sweden's Innovation Agency VINNOVA within the EUREKA CELTIC-NEXT project CYPRESS.
