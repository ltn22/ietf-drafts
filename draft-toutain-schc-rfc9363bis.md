---
title: "A YANG Data Model for Static Context Header Compression (SCHC)"
abbrev: "SCHC YANG Data Model"
docname: draft-toutain-schc-rfc9363bis-00
obsoletes: 9363
category: std
ipr: trust200902
submissiontype: IETF
consensus: true
v: 3
area: "int"
workgroup: "schc"
keyword:
 - Header Compression
 - Fragmentation
 - SCHC Rule
 - IPv6
 - UDP
 - CoAP
 - OSCORE
author:
 -
    ins: A. Minaburo
    name: Ana Minaburo
    org: "Acklio"
    street: "1137A avenue des Champs Blancs"
    city: "Cesson-Sevigne Cedex"
    code: "35510"
    country: "France"
    email: ana@ackl.io
 -
    ins: L. Toutain
    name: Laurent Toutain
    org: "Institut MINES TELECOM; IMT Atlantique"
    street: "2 rue de la Chataigneraie CS 17607"
    city: "Cesson-Sevigne Cedex"
    code: "35576"
    country: "France"
    email: Laurent.Toutain@imt-atlantique.fr
normative:
  RFC0768:
  RFC2119:
  RFC3688:
  RFC6020:
  RFC6241:
  RFC6242:
  RFC7136:
  RFC7252:
  RFC7950:
  RFC8040:
  RFC8174:
  RFC8200:
  RFC8341:
  RFC8342:
  RFC8446:
  RFC8613:
  RFC8724:
  RFC8824:
  I-D.ietf-schc-universal-option:
  I-D.ietf-schc-icmpv6-compression:
  I-D.ietf-schc-8824-update:
informative:
  RFC9441:
  I-D.ietf-lpwan-architecture:
  I-D.toutain-schc-coreconf-management:
  I-D.ietf-core-oscore-key-update:
  RFC4443:
  RFC7967:
  RFC8376:
  RFC9011:
  RFC9363:

--- abstract

This document describes a YANG data model for the Static Context Header Compression (SCHC)
compression and fragmentation Rules.

This document formalizes the description of the Rules for better
interoperability between SCHC instances either to exchange a set of Rules or to
modify the parameters of some Rules.

This document updates the SCHC data model defined in RFC 9363 to take into
account the evolution of SCHC usages. It introduces a flexible way to describe
protocol options, such as the CoAP options, and includes the identifiers needed
for the compression of ICMPv6 and features for the management of the Rules.

--- middle

# Introduction {#Introduction}

SCHC is a compression and fragmentation mechanism for constrained networks
defined in {{RFC8724}}. It is based on a static context shared by two entities
at the boundary of the constrained network. {{RFC8724}} provides an informal
representation of the Rules used either for compression/decompression (C/D) or
fragmentation/reassembly (F/R), and {{RFC9363}} a YANG data model dedicated to
LPWAN environments, where the Rules were most of the time installed directly in
the SCHC end-points.

This document defines a YANG data model {{RFC7950}} to represent compression,
fragmentation, and management Rules, which leads to common representation for
values for all the Rules' elements. This data model is a revision of the
`ietf-schc` module of {{RFC9363}}, and not an augmentation such as the one
defined in {{I-D.ietf-schc-universal-option}}, since the key of the compression
Rule entries is changed. A single entry index replaces the Field ID, the Field
Position and the Direction Indicator, which makes the key shorter and keeps the
entries strictly ordered as the fields in the header {{RFC8724}}. Since the key
of a list cannot be changed in a published module ({{Section 11 of RFC7950}}),
a new list is defined and the previous one is deprecated.

The goal of this document is to update the formal description of the Rules, to
extend SCHC to new environments, and to offer:

* the same definition on both ends, even if the internal representation is
  different,
* an update of the other end to set up some specific values (e.g., IPv6 prefix,
  destination address, etc.), and
* the management of the Rules by the SCHC instances themselves, which create
  and delete Rules using CORECONF
  {{I-D.toutain-schc-coreconf-management}}.


# Requirements Language {#requirements-language}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{RFC2119}} {{RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Terminology {#Term}

This section defines the terminology and acronyms used in this document.
It extends the terminology of {{RFC8376}}.

Action:
: Processing applied to a packet matching a compression Rule. By default, the
  packet is compressed and forwarded.

App:
: Low-Power WAN (LPWAN) Application, as defined by {{RFC8376}}. An application sending/receiving packets to/from the Dev.

Bi:
: Bidirectional. Characterizes a Field Descriptor that applies to headers of packets traveling in either direction (Up and Dw; see this glossary).

CDA:
: Compression/Decompression Action. Describes the pair of actions that are performed at the compressor to compress a header field and at the decompressor to recover the original value of the header field.

Context:
: A set of Rules used to compress/decompress headers.

Dev:
: Device, as defined by {{RFC8376}}.

DevIID:
: Device Interface Identifier. The IID that identifies the Dev interface.

DI:
: Direction Indicator. This field tells which direction of packet travel (Up, Dw, or Bi) a Field Descriptor applies to. This allows for asymmetric processing, using the same Rule.

Dw:
: Downlink direction for compression/decompression, from SCHC C/D in the network to SCHC C/D in the Dev.

Entry Index:
: Sequential position of an entry in a compression Rule, starting at 0. It is
  the key of the list of entries, and follows the order of the fields in the
  header.

FID:
: Field Identifier or Field ID. This identifies the protocol and field a Field Descriptor applies to.

FL:
: Field Length. This is the length of the original packet header field. It is expressed as a number of bits for header fields of fixed lengths or as a type (e.g., variable, token length, ...) for Field Lengths that are unknown at the time of Rule creation. The length of a header field is defined in the corresponding protocol specification (such as IPv6 or UDP).

FP:
: Field Position. When a field is expected to appear multiple times in a header, the Field Position specifies the occurrence this Field Descriptor applies to
  (for example, first Uri-Path option, second Uri-Path, etc. in a Constrained Application Protocol (CoAP) header), counting from 1. The value 0 is special and means "don't care" (see {{Section 7.2 of RFC8724}}).

Guard Period:
: Time during which a deleted Rule ID cannot be reused, so that a SCHC Packet
  sent with the previous Rule cannot be decoded with a new one (see
  {{I-D.toutain-schc-coreconf-management}}).

IID:
: Interface Identifier. See the IPv6 addressing architecture {{RFC7136}}.

L2 Word:
: This is the minimum subdivision of payload data that the Layer 2 (L2) will carry. In most L2 technologies, the L2 Word is an octet.
  In bit-oriented radio technologies, the L2 Word might be a single bit.
  The L2 Word size is assumed to be constant over time for each device.

Management Rule:
: Rule used to carry the management traffic between SCHC instances. Only the
  management Rules can access the Set of Rules (see
  {{I-D.toutain-schc-coreconf-management}}).

MO:
: Matching Operator. An operator used to match a value contained in a header field with a value contained in a Rule.

RuleID:
: Rule Identifier. An identifier for a Rule. SCHC C/D on both sides share the same RuleID for a given packet. A set of RuleIDs are used to support SCHC F/R functionality.

Rule Status:
: State of a Rule in the management protocol: origin, valid, candidate,
  invalid, or free (see {{I-D.toutain-schc-coreconf-management}}).

Space ID:
: Identifier of a protocol option space, such as the CoAP options.

TV:
: Target Value. A value contained in a Rule that will be matched with the value of a header field.

Universal Option:
: Option referenced in a compression Rule by its Space ID and its option
  number, instead of a dedicated FID.

Up:
: Uplink direction for compression/decompression, from the Dev SCHC C/D to the network SCHC C/D.

# SCHC Rules {#schc-rules}

SCHC compression is generic; the main mechanism does not refer to a specific
protocol. Any header field is abstracted through a Field Identifier (FID), a
position (FP), a direction (DI), and a value that can be a numerical value or a
string.

{{RFC8724}}, {{RFC8824}}, and {{I-D.ietf-schc-universal-option}}
specify fields for IPv6 {{RFC8200}}, UDP {{RFC0768}}, and CoAP {{RFC7252}},
including options defined for no server response {{RFC7967}} and Object
Security for Constrained RESTful Environments (OSCORE) {{RFC8613}}, with its
Key Update (KUDOS) extension {{I-D.ietf-core-oscore-key-update}}. For the
latter, {{RFC8824}} and {{I-D.ietf-schc-universal-option}} split this field
into subfields.
{{I-D.ietf-schc-icmpv6-compression}} specifies fields for ICMPv6 {{RFC4443}}.

SCHC fragmentation requires a set of common parameters that are included in a
Rule. These parameters are defined in {{RFC8724}}.

The YANG data model enables the compression, the fragmentation, and the
management selection using the feature statement.

## Main Updates from RFC 9363 {#update-from-rfc9363}

The main updates of the YANG data model defined in {{RFC9363}} are:

* Compression Rule entries: a new list of entries, keyed by an entry index,
  replaces the previous one, which is deprecated (see {{Introduction}} and
  {{compression-rule}}).
* Universal Options: an entry can reference a protocol option by a Space ID
  and its option number, and the per-option CoAP Field IDs are deprecated (see
  {{convention-for-space-identifier}}).
* New identifiers: Field IDs for ICMPv6 {{I-D.ietf-schc-icmpv6-compression}}
  and for the OSCORE and KUDOS subfields (see
  {{convention-for-field-identifier}}), Field Length functions (see
  {{convention-for-field-length}}), Matching Operators (see
  {{convention-for-matching-operator}}), and Compression Decompression Actions
  (see {{convention-for-compression-decompression-actions}}).
* Action: a compression Rule can carry an Action, applied to the packets
  matching it {{I-D.ietf-schc-icmpv6-compression}} (see
  {{convention-for-action}}).
* Rule management: a new feature, a new Rule nature, a Guard period, and a
  status for each Rule {{I-D.toutain-schc-coreconf-management}} (see
  {{management-rule}} and {{yang-tree}}).

## Compression Rules {#comp_types}

{{RFC8724}} proposes an informal representation of the compression Rule.
A compression context for a device is composed of a set of Rules. Each Rule contains information to
describe a specific field in the header to be compressed.

~~~ ascii-art
  +-----------------------------------------------------------------+
  |                      Rule N                                     |
 +-----------------------------------------------------------------+|
 |                    Rule i                                       ||
+-----------------------------------------------------------------+||
|  (FID)            Rule 1                                        |||
|+-------+--+--+--+------------+-----------------+---------------+|||
||Field 1|FL|FP|DI|Target Value|Matching Operator|Comp/Decomp Act||||
|+-------+--+--+--+------------+-----------------+---------------+|||
||Field 2|FL|FP|DI|Target Value|Matching Operator|Comp/Decomp Act||||
|+-------+--+--+--+------------+-----------------+---------------+|||
||...    |..|..|..|   ...      | ...             | ...           ||||
|+-------+--+--+--+------------+-----------------+---------------+||/
||Field N|FL|FP|DI|Target Value|Matching Operator|Comp/Decomp Act|||
|+-------+--+--+--+------------+-----------------+---------------+|/
|                                                                 |        
\-----------------------------------------------------------------/  
~~~
{: #Fig-ctxt title="Compression Decompression Context"}

## Identifier Generation {#identifier-generation}

Identifiers used in the SCHC YANG data model are from the identityref statement to ensure global uniqueness and easy augmentation if needed.  The principle to define a new type based on a group of identityref is the following:

* Define a main identity ending with the keyword `base-type`.
* Derive all the identities used in the data model from this base type.
* Create a typedef from this base type.

The example below ({{Fig-identityref}}) shows how an identityref is created for Reassembly Check Sequence (RCS) algorithms used during SCHC fragmentation.

~~~
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
~~~
{: #Fig-identityref title="Principle to Define a Type Based on identityref"}

## Convention for Field Identifier {#convention-for-field-identifier}

In the process of compression, the headers of the original packet are first parsed to create a list of fields. This list of fields is matched against the Rules to find the appropriate Rule and apply compression.  {{RFC8724}}  does not state how the Field ID value is constructed.
In examples, identification is done through a string indexed by the protocol name (e.g., IPv6.version, CoAP.version, etc.).

The current YANG data model includes field definitions found in {{RFC8724}}
and {{RFC8824}}, with the following updates:

* The per-option CoAP Field IDs are deprecated, since CoAP options are
  represented as Universal Options {{I-D.ietf-schc-universal-option}} (see
  {{convention-for-space-identifier}} and {{compression-rule}}).
* New sub-fields of the OSCORE option are defined, for the OSCORE flags
  (`fid-coap-option-oscore-flags-flagbits`, `fid-coap-option-oscore-flags-n`)
  and for KUDOS (`fid-coap-option-kudos-x`, `fid-coap-option-kudos-x-flagbits`,
  `fid-coap-option-kudos-x-m`, `fid-coap-option-kudos-nonce`)
  {{I-D.ietf-schc-universal-option}}.
* Field IDs are defined for ICMPv6, derived from `fid-icmpv6-base-type`
  {{I-D.ietf-schc-icmpv6-compression}}.
* Two Field IDs independent of any protocol are defined, `fid-unused` and
  `fid-payload` {{I-D.ietf-schc-icmpv6-compression}}.

Using the YANG data model, each field MUST be identified through a global YANG identityref.

A YANG Field ID for the protocol is always derived from the `fid-base-type`. Then, an identity
for each protocol is specified using the naming convention `fid-<<protocol name>>-base-type`.
All possible fields for this protocol MUST derive from the protocol identity. The naming
convention is `fid-` followed by the protocol name and the field name. If a field has
to be divided into subfields, the field identity serves as a base.

The full `field-id` definition is found in {{annexA}}. A type is defined for the IPv6 protocol, and each
field is based on it. Note that the Diffserv bits derive from the Traffic Class identity.

## Convention for Space Identifier {#convention-for-space-identifier}

Some protocols, such as CoAP {{RFC7252}}, carry options identified by a number
in an option space specific to the protocol. Instead of defining a Field ID for
each option, a compression Rule entry can reference an option by a Space ID,
which identifies the option space, and by the option number in this space,
carried in the `universal-value` leaf {{I-D.ietf-schc-universal-option}}. New
options can then be compressed without any update of the YANG data model.

A Space ID is derived from the `space-id-base-type` identity. The naming
convention is `space-id-` followed by the protocol name. The current YANG data
model defines `space-id-coap` for the CoAP options; other protocols that define
options can add their own identity the same way. The `space-id-type` typedef is
defined from `space-id-base-type`, following the principle shown in
{{Fig-identityref}}.

`fid-base-type` and `space-id-base-type` derive from a common ancestor,
`space-field-id-base-type`. A compression Rule entry contains either a
`field-id` or a `space-id` with its `universal-value` (see
{{compression-rule}}).

This document deprecates some CoAP Field IDs originally defined in {{RFC9363}}
(from `fid-coap-option-if-match` to `fid-coap-option-no-response`), which are
replaced by Universal Options using `space-id-coap` and the option number.

## Convention for Field Length {#convention-for-field-length}

The Field Length is either an integer giving the size of a field in bits or a
specific function. {{RFC8724}} defines the "var" function, which allows
variable-length fields (whose length is expressed in bytes), and {{RFC8824}}
defines the "tkl" function for managing the CoAP Token Length field.

{{I-D.ietf-schc-universal-option}} and {{I-D.ietf-schc-icmpv6-compression}}
define the `fl-length-bytes`, `fl-length-bits`, `fl-variable-bits`, and
`fl-remaining` functions. `fl-variable-bits` corresponds to the "var_bit"
function of {{I-D.ietf-schc-8824-update}}.

A Field Length function can take a numerical argument, carried in the
`field-length-value` leaf of the Rule entry. For `fl-length-bytes` and
`fl-length-bits`, this argument is the entry index of the entry giving the
length. The other functions do not use it.

The naming convention is `fl-` followed by the function name.

The Field Length function can be defined as an identityref, as described in
{{annexA}}. Therefore, the type for the Field Length is a union between an
integer giving the size of the length in bits and the identityref.

## Convention for Field Position {#convention-for-field-position}

The Field Position is a positive integer that gives the occurrence times of a
specific field from the header start.  The default value is 1 and is incremented at each repetition.
Value 0 indicates that the position is not important and is not considered during the Rule selection process.

The Field Position is a positive integer. The type is uint8.

## Convention for Direction Indicator {#convention-for-direction-indicator}

The Direction Indicator is used to tell if a field appears in both directions (Bi) or only uplink (Up) or Downlink (Dw). The naming convention is "di" followed by the Direction Indicator name.

The type is `di-type`.

## Convention for Target Value {#target_value}

The Target Value is a list of binary sequences of any length, aligned to the left. In the Rule, the structure will be used as a list, with the index as a key. The highest index value is used to compute the size of the index sent in residue for the match-mapping Compression Decompression Action (CDA). The index can specify several values:

* For equal and most significant bits (MSBs), the Target Value contains a single element. Therefore, the index is set to 0.
* For match-mapping, the Target Value can contain several elements. Index values MUST start from 0 and MUST be contiguous.

If the header field contains text, the binary sequence uses the same encoding.

## Convention for Matching Operator {#convention-for-matching-operator}

The Matching Operator (MO) is a function applied between a field value provided
by the parsed header and the Target Value. {{RFC8724}} defines 4 MOs.
{{I-D.ietf-schc-icmpv6-compression}} defines `mo-rule-match` and
`mo-rev-rule-match`.

The naming convention is `mo-` followed by the MO name.

The type is `mo-type`.

### Matching Operator Arguments {#matching-operator-arguments}

They are viewed as a list, built with a `tv-struct` (see {{target_value}}).

## Convention for Compression Decompression Actions {#convention-for-compression-decompression-actions}

The Compression Decompression Action (CDA) identifies the function to use for
compression or decompression. {{RFC8724}} defines 7 CDAs.
{{I-D.ietf-schc-icmpv6-compression}} defines `cda-compress-sent` and
`cda-rev-compress-sent`.

The naming convention is `cda-` followed by the CDA name.

### Compression Decompression Action Arguments {#compression-decompression-action-arguments}

Currently no CDA requires arguments, but some CDAs may require one or several arguments in the future.
They are viewed as a list of `target-value` type.

## Convention for Action {#convention-for-action}

When a compression Rule is selected, {{RFC8724}} compresses the packet and
sends the resulting SCHC Packet to the lower layer. An Action, introduced by
{{I-D.ietf-schc-icmpv6-compression}}, changes this behavior: the Rule is still
selected through its entries, but the packet is handed over to the process
named by the Action. The Action applies to the Rule as a whole, and not to a
single entry.

The naming convention is `action-` followed by the Action name. The type is
`action-type`. The default Action is `action-forward`: the packet is sent to
the lower layer, as specified in {{RFC8724}}, so a Rule without an Action
behaves as in {{RFC8724}}. {{I-D.ietf-schc-icmpv6-compression}} defines
`action-proxy-ping`.

### Action Arguments {#action-arguments}

The Action is carried in the `action` leaf of the compression Rule, and its
arguments in the `action-value` list of the same Rule. The arguments are viewed
as a list, built with a `tv-struct` (see {{target_value}}). `action-forward`
takes no argument.

## Fragmentation Rule {#frag_types}

Fragmentation is optional in the data model and depends on the presence of the `fragmentation` feature.

The SCHC Compound ACK {{RFC9441}} augments this data model with the
`ietf-schc-compound-ack` module, to acknowledge several windows in a single
SCHC ACK message in the ACK-on-Error mode. This augmentation remains valid
for the data model defined in this document, since the fragmentation part of
the module, augmented by {{RFC9441}}, is unchanged.

Most of the fragmentation parameters are listed in {{Appendix D of RFC8724}}.

Since fragmentation Rules work for a specific direction, they MUST contain a mandatory Direction Indicator.
The type is the same as the one used in compression entries, but bidirectional MUST NOT be used.

### Fragmentation Mode {#fragmentation-mode}

{{RFC8724}} defines 3 fragmentation modes:

* No ACK: This mode is unidirectional; no acknowledgment is sent back.
* ACK Always: Each fragmentation window must be explicitly acknowledged before going to the next.
* ACK on Error: A window is acknowledged only when the receiver detects some missing fragments.

The type is `fragmentation-mode-type`.
The naming convention is `fragmentation-mode-` followed by the fragmentation mode name.

### Fragmentation Header {#fragmentation-header}

A data fragment header, starting with the RuleID, can be sent in the fragmentation direction.
{{RFC8724}} indicates that the SCHC header may be composed of the following (cf. {{Fig-frag-header-8724}}):

* a Datagram Tag (DTag) identifying the datagram being fragmented if the fragmentation applies concurrently on several datagrams. This field is optional, and its length is defined by the Rule.
* a Window (W) used in ACK-Always and ACK-on-Error modes. In ACK-Always, its size is 1. In ACK-on-Error, it depends on the Rule. This field is not needed in No-ACK mode.
* a Fragment Compressed Number (FCN) indicating the fragment/tile position within the window. This field is mandatory on all modes defined in {{RFC8724}}, and its size is defined by the Rule.

~~~ ascii-art
|-- SCHC Fragment Header ----|
         |-- T --|-M-|-- N --|
+-- ... -+- ... -+---+- ... -+--------...-------+~~~~~~~~~~~~~~~~~~~~
| RuleID | DTag  | W |  FCN  | Fragment Payload | padding (as needed)
+-- ... -+- ... -+---+- ... -+--------...-------+~~~~~~~~~~~~~~~~~~~~
~~~
{: #Fig-frag-header-8724 title="Data Fragment Header from RFC 8724"}

### Last Fragment Format {#last-fragment-format}

The last fragment of a datagram is sent with a Reassembly Check Sequence (RCS) field to detect residual
transmission errors and possible losses in the last window. {{RFC8724}} defines a single algorithm based on Ethernet
CRC computation.

The naming convention is `rcs-` followed by the algorithm name.

For ACK-on-Error mode, the All-1 fragment may just contain the RCS or can include a tile. The following parameters define the
behavior:

* `all-1-data-no`: The last fragment contains no data, just the RCS.
* `all-1-data-yes`: The last fragment includes a single tile and the RCS.
* `all-1-data-sender-choice`: The last fragment may or may not contain a single tile. The receiver can detect if a tile is present.

The naming convention is `all-1-data-` followed by the behavior identifier.

### Acknowledgment Behavior {#acknowledgment-behavior}

The acknowledgment fragment header goes in the opposite direction of data. {{RFC8724}} defines the header, which is composed of the following (see {{Fig-frag-ack}}):

* a DTag (if present).
* a mandatory window, as in the data fragment.
* a C bit giving the status of RCS validation.  In case of failure, a bitmap follows, indicating the received tile.

~~~ ascii-art
|--- SCHC ACK Header ----|
         |-- T --|-M-| 1 |
+-- ... -+- ... -+---+---+~~~~~~~~~~~~~~~~~~
| RuleID |  DTag | W |C=1| padding as needed                (success)
+-- ... -+- ... -+---+---+~~~~~~~~~~~~~~~~~~

+-- ... -+- ... -+---+---+------ ... ------+~~~~~~~~~~~~~~~
| RuleID |  DTag | W |C=0|Compressed Bitmap| pad. as needed (failure)
+-- ... -+- ... -+---+---+------ ... ------+~~~~~~~~~~~~~~~
~~~
{: #Fig-frag-ack title="Acknowledgment Fragment Header for RFC 8724"}

For ACK-on-Error, SCHC defines when an acknowledgment can be sent. This can be at any time defined by the Layer 2, at the end of a window (FCN all-0),
or as a response to receiving the last fragment (FCN all-1). The naming convention is `ack-behavior` followed by the algorithm name.

### Timer Values {#timer-values}

The state machine requires some common values to handle fragmentation correctly.

* The Retransmission Timer gives the duration before sending an ACK request (cf. {{Section 8.2.2.4 of RFC8724}}). If specified, the value MUST be strictly positive.
* The Inactivity Timer gives  the duration before aborting a fragmentation session (cf.  {{Section 8.2.2.4 of RFC8724}}). The value 0 explicitly indicates that this timer is disabled.

{{RFC8724}} does not specify any range for these timers. {{RFC9011}} recommends a duration of 12 hours. In fact, the value range should be between milliseconds for real-time systems to several days for worse-than-best-effort systems. To allow a large range of applications, two parameters must be specified:

* the duration of a tick. It is computed by this formula: 2<sup>tick-duration</sup>/10<sup>6</sup>. When tick-duration is set to 0, the unit is the microsecond. The default value of 20 leads to a unit of 1.048575 seconds. A value of 32 leads to a tick-duration of about 1 hour 11 minutes.
* the number of ticks in the predefined unit. With the default tick-duration value of 20, the timers can cover a range between 1.0 second and 19 hours, as recommended in {{RFC9011}}.

### Fragmentation Parameter {#fragmentation-parameter}

The SCHC fragmentation protocol specifies the number of attempts before aborting through the parameter:

* `max-ack-requests`  (cf. {{Section 8.2.2.4 of RFC8724}})

### Layer 2 Parameters {#layer-2-parameters}

The data model includes two parameters needed for fragmentation:

* `l2-word-size`: {{RFC8724}} base fragmentation, in bits, on a Layer 2 Word that can be of any length. The default value is 8 and corresponds
  to the default value for the byte-aligned Layer 2. A value of 1 will indicate that there is no alignment and no need for padding.
* `maximum-packet-size`: defines the maximum size of an uncompressed datagram. By default, the value is set to 1280 bytes.

They are defined as unsigned integers; see {{annexA}}.

# Rule Definition {#rule-definition}

A Rule is identified by a unique Rule Identifier (RuleID) comprising both a RuleID value and a RuleID length.
The YANG grouping `rule-id-type` defines the structure used to represent a RuleID. A length of 0 is allowed to represent an implicit Rule.

Three natures of Rules are defined in {{RFC8724}}:

* Compression: A compression Rule is associated with the RuleID.
* No-compression: This identifies the default Rule used to send a packet integrally when no-compression Rule was found (see {{Section 6 of RFC8724}}).
* Fragmentation: Fragmentation parameters are associated with the RuleID. Fragmentation is optional, and the feature `fragmentation` should be set.

{{I-D.toutain-schc-coreconf-management}} defines a fourth nature:

* Management: A management Rule carries the management traffic between SCHC
  instances. It uses the compression Rule structure, and only the
  `entry-universal` list. Management is optional, and the feature
  `management` should be set.

The YANG data model respectively introduces these four identities:

* `nature-compression`
* `nature-no-compression`
* `nature-fragmentation`
* `nature-management`

The naming convention is `nature-` followed by the nature identifier.

To access a specific Rule, the RuleID length and value are used as a key. The
Rule is either a compression, a no-compression, a fragmentation, or a
management Rule.

## Compression Rule {#compression-rule}

A compression Rule is composed of entries describing its processing. An entry
contains all the information defined in {{Fig-ctxt}} with the types defined
above.

One main change in this data model is the deprecation of the
`compression-content` grouping, used by the `compression` case, in favor of
the `compression-content-universal` grouping, used by the
`compression-universal` case. {{I-D.ietf-schc-universal-option}} explains this
change.

Universal Options give more flexibility in the management of the options.
Defining a new SCHC Field ID identityref for each option requires the SCHC
data model to be updated each time a protocol defines a new option. With
Universal Options, only a Space ID identity is defined in the SCHC data model,
and the entry carries the option number used by the protocol (see
{{convention-for-space-identifier}}). Any new option defined by a protocol can
then be added to a Rule without updating the data model.

To avoid long keys to access a specific entry, the `entry-index` leaf is the
key of the `entry-universal` list, instead of the Field ID, the Field Position
and the Direction Indicator. It also guarantees the order of the entries
inside a Rule, an order reinforced by the "ordered-by user" statement of the
list. Each entry of `entry-universal` represents a line in {{Fig-ctxt}}, and
contains either a `field-id` or a `space-id` with its `universal-value`.

The Field Length also includes a parameter, the `field-length-value` leaf,
used by the new functions `fl-length-bytes` and `fl-length-bits`, which
generalize functions such as `fl-token-length` (see
{{convention-for-field-length}}).

The deprecated `compression-content` grouping defines a list of
`compression-rule-entry`, indexed by their Field ID, position, and direction.
Their type reflects the identifier types defined in {{comp_types}}.

This change also deprecates the per-option CoAP Field IDs, since a CoAP option
can be represented as a Universal Option with the tuple (`space-id-coap`,
option number). Only the Field IDs describing the sub-fields of a single CoAP
option, for OSCORE and KUDOS, remain current: they describe the parts of the
option rather than stand for an arbitrary one, and use `fid-coap-option` as
their base.

A deprecated definition is still valid: following {{Section 7.21.2 of
RFC7950}}, it may still be implemented, to keep interoperability with existing
implementations. The Rules defined with the data model of {{RFC9363}} therefore
remain valid.

However, the deprecated per-option CoAP Field IDs MUST NOT be used in the
`entry-universal` list: a CoAP option is then represented by `space-id-coap`
and its option number. They remain usable only in the deprecated `entry` list.

The identities defined by this revision of the data model (Field IDs, Field
Length functions, Matching Operators, and Compression Decompression Actions)
MUST NOT be used in the deprecated `entry` list, which is only kept for the
Rules defined with {{RFC9363}}.

Some checks are performed on the values, both in `entry-universal` and in the
deprecated `entry` list:

* When MO is ignore, no Target Value is needed; for other MOs, there MUST be a
  Target Value present.
* When MSB MO is specified, the `matching-operator-value` must be present.
* When the CDA is not-sent, LSB, or mapping-sent, there MUST be a Target Value
  present; it is not needed for value-sent, compute, DevIID, AppIID,
  compress-sent, and rev-compress-sent.
* The `entry-universal` list is only used in Rules whose nature is
  `nature-compression` or `nature-management`; the deprecated `entry` list is
  only used in Rules whose nature is `nature-compression`.

The following checks only apply to the `entry-universal` list:

* When the Field Length is `fl-length-bytes` or `fl-length-bits`, the
  `field-length-value` leaf MUST be present.
* The deprecated per-option CoAP Field IDs MUST NOT be used as `field-id`.

## Fragmentation Rule {#fragmentation-rule}

A fragmentation Rule is composed of entries describing the protocol behavior.
Some of them are numerical entries, others are identifiers defined in
{{frag_types}}.

The fragmentation Rule is kept unchanged from {{RFC9363}}. The SCHC Compound
ACK {{RFC9441}} augments it to allow several bitmaps in a SCHC ACK message.

## Management Rule {#management-rule}

Management Rules are defined in {{I-D.toutain-schc-coreconf-management}}. They
allow a SCHC instance to modify the Set of Rules they belong to, except the
origin Rules, which include the management Rules themselves (see below). A
management Rule uses the compression Rule structure, with the `entry-universal`
list, and the `nature-management` nature.

{{I-D.toutain-schc-coreconf-management}} also introduces a Guard period,
during which a deleted Rule ID MUST NOT be reused. This information is global
to all the Rules: the `guard-period` container is carried in the `context`
container, and not in a Rule. It uses the same coding as the fragmentation
timers, with `ticks-duration` and `ticks-numbers` (see {{timer-values}}).

{{I-D.toutain-schc-coreconf-management}} also introduces a Rule status, to
indicate whether a Rule can be used or not. The `rule-status` leaf is only
present when the `management` feature is activated. Its type is
`status-type`, and its values are:

* `status-origin`: the Rule belongs to the initial Set of Rules, provisioned
  before the start of the SCHC instance. It cannot be modified or deleted by
  management, and it is the only kind of Rule kept when the context is reset.
  This is the default value: a provisioned Rule without `rule-status` is an
  origin Rule, while a Rule created by management always carries its status.
* `status-valid`: the Rule belongs to the Set of Rules and can be used. Only
  the Rules with this status, or with `status-origin`, which is derived from
  it, can be used for compression, decompression, fragmentation, and
  reassembly.
* `status-candidate`: the Rule has been created by management, but the other
  end has not validated it yet. It cannot be used.
* `status-invalid`: the Rule has been deleted by management, and the Guard
  period is running. It cannot be used, and its Rule ID cannot be reused.
* `status-free`: the Rule ID can be used to create a new Rule. A Rule with
  this status is processed as if it did not exist in the Set of Rules.

Management Rules are origin Rules: their status is always `status-origin`.

The naming convention is `status-` followed by the status name.

The value of `rule-status` is a local view of the state of the management
protocol machine of the node for this Rule. It is managed locally by the node,
and any value sent by the other end is overwritten. In particular,
`status-origin` is only set by provisioning, never by management. The
procedures, including the reset of the context, are described in
{{I-D.toutain-schc-coreconf-management}}.

## YANG Tree {#yang-tree}

The YANG data model described in this document conforms to the
Network Management Datastore Architecture defined in {{RFC8342}}.

~~~ yangtree
module: ietf-schc
  +--rw context {management}?
  |  +--rw guard-period
  |     +--rw ticks-duration?   uint8
  |     +--rw ticks-numbers?    uint16
  +--rw schc
     +--rw rule* [rule-id-value rule-id-length]
        +--rw rule-id-value                   uint32
        +--rw rule-id-length                  uint8
        +--rw rule-nature                     nature-type
        +--rw rule-status?                    status-type
        |       {management}?
        +--rw (nature)?
           +--:(fragmentation) {fragmentation}?
           |  +--rw fragmentation-mode
           |  |       schc:fragmentation-mode-type
           |  +--rw l2-word-size?             uint8
           |  +--rw direction                 schc:di-type
           |  +--rw dtag-size?                uint8
           |  +--rw w-size?                   uint8
           |  +--rw fcn-size                  uint8
           |  +--rw rcs-algorithm?            rcs-algorithm-type
           |  +--rw maximum-packet-size?      uint16
           |  +--rw window-size?              uint16
           |  +--rw max-interleaved-frames?   uint8
           |  +--rw inactivity-timer
           |  |  +--rw ticks-duration?   uint8
           |  |  +--rw ticks-numbers?    uint16
           |  +--rw retransmission-timer
           |  |  +--rw ticks-duration?   uint8
           |  |  +--rw ticks-numbers?    uint16
           |  +--rw max-ack-requests?         uint8
           |  +--rw (mode)?
           |     +--:(no-ack)
           |     +--:(ack-always)
           |     +--:(ack-on-error)
           |        +--rw tile-size?          uint8
           |        +--rw tile-in-all-1?      schc:all-1-data-type
           |        +--rw ack-behavior?       schc:ack-behavior-type
           x--:(compression) {compression}?
           |  x--rw entry*
           |          [field-id field-position direction-indicator]
           |     +--rw field-id                    schc:fid-type
           |     +--rw field-length                union
           |     +--rw field-position              uint8
           |     +--rw direction-indicator         schc:di-type
           |     +--rw target-value* [index]
           |     |  +--rw index    uint16
           |     |  +--rw value?   binary
           |     +--rw matching-operator           schc:mo-type
           |     +--rw matching-operator-value* [index]
           |     |  +--rw index    uint16
           |     |  +--rw value?   binary
           |     +--rw comp-decomp-action          schc:cda-type
           |     +--rw comp-decomp-action-value* [index]
           |        +--rw index    uint16
           |        +--rw value?   binary
           +--:(compression-universal) {compression or management}?
              +--rw entry-universal* [entry-index]
              |  +--rw entry-index                 uint16
              |  +--rw (field-or-space)
              |  |  +--:(regular-field)
              |  |  |  +--rw field-id              schc:fid-type
              |  |  +--:(universal-option)
              |  |     +--rw space-id              schc:space-id-type
              |  |     +--rw universal-value       uint64
              |  +--rw field-length                union
              |  +--rw field-length-value?         uint16
              |  +--rw field-position              uint8
              |  +--rw direction-indicator         schc:di-type
              |  +--rw target-value* [index]
              |  |  +--rw index    uint16
              |  |  +--rw value?   binary
              |  +--rw matching-operator           schc:mo-type
              |  +--rw matching-operator-value* [index]
              |  |  +--rw index    uint16
              |  |  +--rw value?   binary
              |  +--rw comp-decomp-action          schc:cda-type
              |  +--rw comp-decomp-action-value* [index]
              |     +--rw index    uint16
              |     +--rw value?   binary
              +--rw action?                   action-type
              +--rw action-value* [index]
                 +--rw index    uint16
                 +--rw value?   binary
~~~
{: #Fig-model-overview title="Overview of the SCHC Data Model"}

# YANG Data Model {#annexA}

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

  revision 2026-10-27 {
    description
      "This revision updates the data model of RFC 9363:
       - Compression Rule entries: the 'entry-universal' list, keyed
         by 'entry-index', replaces the 'entry' list. An entry
         contains either a 'field-id' or a 'space-id' with its
         'universal-value', and a 'field-length-value' leaf. The
         'entry' list, the 'compression-content' and
         'compression-rule-entry' groupings, and the 'compression'
         case are deprecated.
       - Universal Options: new 'space-field-id-base-type',
         'space-id-base-type' and 'space-id-coap' identities, and
         'space-field-id-type' and 'space-id-type' typedefs. The
         per-option CoAP Field IDs and 'fid-oscore-base-type' are
         deprecated.
       - New Field IDs: OSCORE and KUDOS sub-fields, ICMPv6 fields,
         'fid-unused' and 'fid-payload'.
       - New Field Length functions: 'fl-length-bytes',
         'fl-length-bits', 'fl-variable-bits' and 'fl-remaining'.
       - New Matching Operators 'mo-rule-match' and
         'mo-rev-rule-match', and Compression Decompression Actions
         'cda-compress-sent' and 'cda-rev-compress-sent'.
       - Action: 'action' leaf and 'action-value' list in the
         compression Rule, 'action-type' typedef and
         'action-base-type', 'action-forward' and
         'action-proxy-ping' identities.
       - Rule management: 'management' feature, 'nature-management'
         nature, 'context' container with the 'guard-period' timer,
         and 'rule-status' leaf with the 'status-type' typedef and
         its identities.
       - Constraints in 'entry-universal': 'field-length-value' is
         required by 'fl-length-bytes' and 'fl-length-bits', and the
         per-option CoAP Field IDs are rejected. The Target Value is
         no longer required for 'cda-compress-sent' and
         'cda-rev-compress-sent'.";
    reference
      "RFC XXXX A YANG Data Model for Static Context Header
                Compression (SCHC)";
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
      "Payload field in any protocol. This field contains the
       remaining
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

  identity fid-oscore-base-type {
    base fid-coap-option;
    status deprecated;
    description
      "OSCORE options (RFC8613) split in suboptions. Deprecated: this
       identity is not used as a base, the OSCORE suboption Field IDs
       derive from fid-coap-option.";
    reference
      "RFC 8824 Static Context Header Compression (SCHC) for the
                Constrained Application Protocol (CoAP)";
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

   identity fid-coap-option-oscore-flags-flagbits {
    base fid-coap-option-oscore-flags;
    description
      "First 5 bits of the OSCORE flags field forming flags
       subfield.";
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

  identity fid-coap-option-kudos-x-flagbits {
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
      "NOT YET DEFINED IN RFCs. This function is a generalization of
       fl-variable to support bit-level length definition.";
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
    by its index in bytes. This is a generalation of
    fl-token-length.";
  }

  identity fl-length-bits {
    base fl-base-type;

    description "This function return the length in bits of a field
    by its index in bits. ";
  }

  identity fl-remaining {
    base fl-base-type;

    description
      "Field length is not explicitly carried in the Compression
       Residue.
      This function is used for a field that extends to the end of
      the
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
      "Field ID type for regular protocol fields (IPv6, UDP, CoAP,
       etc.).
       Used in the regular-field case of compression-rule-entry.";
  }

  typedef space-id-type {
    type identityref {
      base space-id-base-type;
    }
    description
      "Space ID type for universal option spaces (CoAP options,
       etc.).
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

       Deprecated in favor of 'compression-rule-entry-universal',
       which
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
                derived-from-or-self(., 'cda-deviid') or
                derived-from-or-self(., 'cda-compress-sent') or
                derived-from-or-self(., 'cda-rev-compress-sent')" {
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
       - universal-value: the value associated to the space field ID
       to identify
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
        "Sequential position of the entry in the user-ordered Rule
         list,
         starting at 0.";
    }

    choice field-or-space {
      mandatory true;
      description
        "Distinguishes a regular protocol field from a universal
         option.
         Exactly one case must be present:
         - regular-field: a standard protocol field identified by a
         field ID
           (e.g., fid-ipv6-version, fid-udp-dev-port).
         - universal-option: a generic option identified by a space
         ID
           (e.g., space-id-coap) plus a numeric option value (e.g.,
           the
           CoAP option number).";

      case regular-field {
        description
          "The entry refers to a regular protocol field.
           No universal-value is needed.";
        leaf field-id {
          type schc:fid-type;
          must "not(
          derived-from-or-self(.,'fid-coap-option-if-match') or
          derived-from-or-self(.,'fid-coap-option-uri-host') or
          derived-from-or-self(.,'fid-coap-option-etag') or
          derived-from-or-self(.,'fid-coap-option-if-none-match') or
          derived-from-or-self(.,'fid-coap-option-observe') or
          derived-from-or-self(.,'fid-coap-option-uri-port') or
          derived-from-or-self(.,'fid-coap-option-location-path') or
          derived-from-or-self(.,'fid-coap-option-uri-path') or
          derived-from-or-self(.,'fid-coap-option-content-format') or
          derived-from-or-self(.,'fid-coap-option-max-age') or
          derived-from-or-self(.,'fid-coap-option-uri-query') or
          derived-from-or-self(.,'fid-coap-option-accept') or
          derived-from-or-self(.,'fid-coap-option-location-query') or
          derived-from-or-self(.,'fid-coap-option-block2') or
          derived-from-or-self(.,'fid-coap-option-block1') or
          derived-from-or-self(.,'fid-coap-option-size2') or
          derived-from-or-self(.,'fid-coap-option-proxy-uri') or
          derived-from-or-self(.,'fid-coap-option-proxy-scheme') or
          derived-from-or-self(.,'fid-coap-option-size1') or
          derived-from-or-self(.,'fid-coap-option-no-response'))" {
            error-message
              "Deprecated per-option CoAP Field IDs must not be used
               in entry-universal; use space-id-coap and the option
               number.";
          }
          mandatory true;
          description
            "Field ID of a regular protocol field.";
        }
      }

      case universal-option {
        description
          "The entry refers to a universal option within a given
           space.
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
            "Numeric value identifying the specific option within the
             space
             (e.g., the CoAP option number).";
        }
      } // case universal-option
    } // choice field-or-space

    leaf field-length {
      type union {
      type uint8;
      type schc:fl-type;
        }
      must "not(derived-from-or-self(., 'fl-length-bytes') or
                 derived-from-or-self(., 'fl-length-bits')) or
            ../field-length-value" {
        error-message
          "fl-length-bytes and fl-length-bits need
           field-length-value";
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
                derived-from-or-self(., 'cda-deviid') or
                derived-from-or-self(., 'cda-compress-sent') or
                derived-from-or-self(., 'cda-rev-compress-sent')" {
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
       compression Rule structure to operate on the SCHC Context.
       Management Rules are origin Rules: their status is always
       'status-origin'.";
  /* Reference to be added when the Rule Management specification
     is available. */
  }

  identity status-base-type {
    description
      "Base identity for the status of a Rule.";
  }

  identity status-valid {
    base status-base-type;
    description
      "The Rule belongs to the Set of Rules and can be used. Only
       the Rules with this status, or with a status derived from
       it, can be used for compression, decompression,
       fragmentation, and reassembly.";
  }

  identity status-origin {
    base status-valid;
    description
      "The Rule belongs to the initial Set of Rules, provisioned
       before the start of the SCHC instance. It is valid, and it
       cannot be modified or deleted by management. The management
       Rules are origin Rules. When the context is reset, only the
       origin Rules are kept. This status is only set by
       provisioning, never by management.";
  }

  identity status-candidate {
    base status-base-type;
    description
      "The Rule has been created by management and the other end
       has not validated it yet. It cannot be used for compression,
       decompression, fragmentation or reassembly, a SCHC packet
       MUST NOT be generated with its RuleID, and a received
       packet carrying its RuleID is dropped.";
  }

  identity status-invalid {
    base status-base-type;
    description
      "The Rule has been deleted by management and the guard period
       is running. It cannot be used, no Rule with the same RuleID
       can be created, and the SCHC packets carrying its RuleID are
       dropped. When the guard period has elapsed, the RuleID
       becomes free.";
  }

  identity status-free {
    base status-base-type;
    description
      "The RuleID can be used to create a new Rule. When the guard
       period of a deleted Rule has elapsed, the Rule is either kept
       in the Set of Rules with this status, or removed from it. A
       RuleID absent from the Set of Rules is also free. A Rule with
       this status is processed as if it did not exist in the Set
       of Rules.";
  }

  typedef status-type {
    type identityref {
      base status-base-type;
    }
    description
      "Defines the type to indicate the status of a Rule.";
  }

  identity action-base-type {
    description
      "Action applied to a packet matching a compression Rule, once
       compression has been performed (or before decompression is
       performed in the other direction). The default behavior of
       RFC 8724 is to hand the SCHC Packet over to the lower layer;
       other actions allow an intermediate entity, typically the
       SCHC gateway, to process the packet locally instead.";
    reference
      "draft-barthel-lpwan-oam-schc-05 OAM for LPWAN using Static
                Context Header Compression (SCHC) (see Section 5)";
  }

  identity action-forward {
    base action-base-type;
    description
      "The packet is handed over to the lower layer. This is the
       default behavior specified in RFC 8724.";
    reference
      "RFC 8724 SCHC: Generic Framework for Static Context Header
                Compression and Fragmentation";
  }

  identity action-proxy-ping {
    base action-base-type;
    description
      "The packet is not handed over to the lower layer but
       processed locally by a ping proxy, which answers the ICMPv6
       Echo Request on behalf of the Device.";
    reference
      "draft-barthel-lpwan-oam-schc-05 OAM for LPWAN using Static
                Context Header Compression (SCHC) (see Section 5)";
  }

  typedef action-type {
    type identityref {
      base action-base-type;
    }
    description
      "Defines the type to indicate the action associated with a
       compression Rule.";
  }

  grouping compression-content {
    status deprecated;
    list entry {
      must "derived-from-or-self(../rule-nature,
                                        'nature-compression')" {
        error-message "Rule nature must be compression";
      }
      key "field-id field-position direction-indicator";
      status deprecated;

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

    leaf action {
      type action-type;
      default "schc:action-forward";
      description
        "Action to be applied to a packet matching this Rule, after
         compression or before decompression. When absent, the
         packet is sent to the lower layer as specified in
         RFC 8724.";
    }

    list action-value {
      key "index";
      uses tv-struct;
      description
        "Parameters associated with the action, based on a TV
         structure, in order to allow for several arguments.
         'action-forward' requires no argument.";
    }

    description
      "Define a compression Rule composed of a list of entries,
       using the Universal-Option-aware
       'compression-rule-entry-universal'.";
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
      "This group contains parameters to control management
       procedure.";

    container guard-period {
      description
        "Duration of the Guard period, during which a deleted RuleID
         MUST NOT be reused. It applies to all the Rules.";
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
      leaf rule-status {
        if-feature "management";
        type status-type;
        default "status-origin";
        description
          "Status of the Rule. A Rule with the value
           'status-candidate' has been created by management and is
           not usable until the other end has validated it. A Rule
           with the value 'status-invalid' has been deleted and
           stays in this status during the guard period. A Rule
           without this leaf, such as a provisioned Rule, is
           considered 'status-origin'; a Rule created by management
           always carries its status.
           The value is a local view of the state of the management
           protocol machine of the node. It is managed locally by
           the node, and any value sent by the other end is
           overwritten.";
      }
      choice nature {
        case fragmentation {
          if-feature "fragmentation";
          uses fragmentation-content;
        }
        case compression {
          if-feature "compression";
          status deprecated;
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
}
~~~
{: #Fig-schc title="SCHC YANG Data Model" sourcecode-name="ietf-schc@2026-10-27.yang" sourcecode-markers="true"}

# IANA Considerations {#iana-considerations}

This document registers one URI and one YANG data model.

## URI Registration {#uri-registration}

IANA registered the following URI in the "IETF XML Registry" {{RFC3688}}:

URI:
: urn:ietf:params:xml:ns:yang:ietf-schc

Registrant Contact:
: The IESG.

XML:
: N/A; the requested URI is an XML namespace.

## YANG Module Name Registration {#yang-module-name-registration}

IANA has registered the following YANG data model in the "YANG Module Names" registry {{RFC6020}}.

name:
: ietf-schc

namespace:
: urn:ietf:params:xml:ns:yang:ietf-schc

prefix:
: schc

reference:
: RFC 9363

# Security Considerations {#security-considerations}

The YANG module specified in this document defines a schema for data that is designed to be accessed via network management protocols such as NETCONF {{RFC6241}} or RESTCONF {{RFC8040}}. The lowest NETCONF layer is the secure transport layer, and the mandatory-to-implement secure transport is Secure Shell (SSH) {{RFC6242}}. The lowest RESTCONF layer is HTTPS, and the mandatory-to-implement secure transport is TLS
{{RFC8446}}.

The Network Configuration Access Control Model (NACM) {{RFC8341}} provides the means to restrict access for particular NETCONF or RESTCONF users to a preconfigured subset of all available NETCONF or RESTCONF protocol operations and content.

There are a number of data nodes defined in this YANG module that are writable/creatable/deletable
(i.e., config true, which is the default). These data nodes may be considered sensitive or vulnerable
in some network environments. Write operations (e.g., edit-config) to these data nodes without proper
protection can have a negative effect on network operations. These are the subtrees and data nodes and
their sensitivity/vulnerability:

/schc:
: All the data nodes may be modified. The Rule contains sensitive information, such as the application IPv6 address where the device's data will be sent after decompression. An attacker may try to modify other devices' Rules by changing the application address and may block communication or allows traffic eavesdropping. Therefore, a device must be allowed to modify only its own rules on the remote SCHC instance. The identity of the requester must be validated. This can be done through certificates or access lists. Modification may be allowed regarding the Field Descriptor (i.e., IPv6 addresses field descriptors should not be modified, but UDP dev port could be changed).

Some of the readable data nodes in this YANG module may be considered sensitive or vulnerable in some network environments. It is thus important to control read access (e.g., via get, get-config, or notification) to these data nodes. These are the subtrees and data nodes and their sensitivity/vulnerability:

/schc:
: By reading a module, an attacker may learn the traffic generated by a device and can also learn about application addresses or REST API.

--- back

# Example {#annex-a-example}

The informal Rules given {{Fig-example-rules}} are represented in XML, as shown in {{Fig-XML-rules}}.

~~~ ascii-art
/-------------------------\
|Rule 6/3            110  |
|---------------+---+--+--+----------------+-------+----------------\
|IPV6.VER       |  4| 1|BI|               6|EQUAL  |NOT-SENT        |
|IPV6.TC        |  8| 1|BI|               0|EQUAL  |NOT-SENT        |
|IPV6.FL        | 20| 1|BI|               0|IGNORE |NOT-SENT        |
|IPV6.LEN       | 16| 1|BI|                |IGNORE |COMPUTE-LENGTH  |
|IPV6.NXT       |  8| 1|BI|              58|EQUAL  |NOT-SENT        |
|IPV6.HOP_LMT   |  8| 1|BI|             255|IGNORE |NOT-SENT        |
|IPV6.DEV_PREFIX| 64| 1|BI|200104701f2101d2|EQUAL  |NOT-SENT        |
|IPV6.DEV_IID   | 64| 1|BI|0000000000000003|EQUAL  |NOT-SENT        |
|IPV6.APP_PREFIX| 64| 1|BI|                |IGNORE |VALUE-SENT      |
|IPV6.APP_IID   | 64| 1|BI|                |IGNORE |VALUE-SENT      |
\---------------+---+--+--+----------------+-------+----------------/
/-------------------------\
|Rule 12/11     00001100  |
!=========================+=========================================\
!^ Fragmentation mode : NoAck   header dtag 2 Window  0 FCN  3  UP ^!
!^ No Tile size specified                                          ^!
!^ RCS Algorithm: RCS_CRC32                                        ^!
\===================================================================/
/-------------------------\
|Rule 100/8     01100100  |
| NO-COMPRESSION RULE     |
\-------------------------/
~~~
{: #Fig-example-rules title="Rules Example"}

~~~ xml
<?xml version='1.0' encoding='UTF-8'?>
  <schc xmlns="urn:ietf:params:xml:ns:yang:ietf-schc">
  <rule>
    <rule-id-value>6</rule-id-value>
    <rule-id-length>3</rule-id-length>
    <rule-nature>nature-compression</rule-nature>
    <entry-universal>
      <entry-index>0</entry-index>
      <field-id>fid-ipv6-version</field-id>
      <field-length>4</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-equal</matching-operator>
      <comp-decomp-action>cda-not-sent</comp-decomp-action>
      <target-value>
        <index>0</index>
        <value>AAY=</value>
      </target-value>
    </entry-universal>
    <entry-universal>
      <entry-index>1</entry-index>
      <field-id>fid-ipv6-trafficclass</field-id>
      <field-length>8</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-equal</matching-operator>
      <comp-decomp-action>cda-not-sent</comp-decomp-action>
      <target-value>
        <index>0</index>
        <value>AA==</value>
      </target-value>
    </entry-universal>
    <entry-universal>
      <entry-index>2</entry-index>
      <field-id>fid-ipv6-flowlabel</field-id>
      <field-length>20</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-ignore</matching-operator>
      <comp-decomp-action>cda-not-sent</comp-decomp-action>
      <target-value>
        <index>0</index>
        <value>AA==</value>
      </target-value>
    </entry-universal>
    <entry-universal>
      <entry-index>3</entry-index>
      <field-id>fid-ipv6-payload-length</field-id>
      <field-length>16</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-ignore</matching-operator>
      <comp-decomp-action>cda-compute</comp-decomp-action>
    </entry-universal>
    <entry-universal>
      <entry-index>4</entry-index>
      <field-id>fid-ipv6-nextheader</field-id>
      <field-length>8</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-equal</matching-operator>
      <comp-decomp-action>cda-not-sent</comp-decomp-action>
      <target-value>
        <index>0</index>
        <value>ADo=</value>
      </target-value>
    </entry-universal>
    <entry-universal>
      <entry-index>5</entry-index>
      <field-id>fid-ipv6-hoplimit</field-id>
      <field-length>8</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-ignore</matching-operator>
      <comp-decomp-action>cda-not-sent</comp-decomp-action>
      <target-value>
        <index>0</index>
        <value>AP8=</value>
      </target-value>
    </entry-universal>
    <entry-universal>
      <entry-index>6</entry-index>
      <field-id>fid-ipv6-devprefix</field-id>
      <field-length>64</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-equal</matching-operator>
      <comp-decomp-action>cda-not-sent</comp-decomp-action>
      <target-value>
        <index>0</index>
        <value>IAEEcB8hAdI=</value>
      </target-value>
    </entry-universal>
    <entry-universal>
      <entry-index>7</entry-index>
      <field-id>fid-ipv6-deviid</field-id>
      <field-length>64</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-equal</matching-operator>
      <comp-decomp-action>cda-not-sent</comp-decomp-action>
      <target-value>
        <index>0</index>
        <value>AAAAAAAAAAM=</value>
      </target-value>
    </entry-universal>
    <entry-universal>
      <entry-index>8</entry-index>
      <field-id>fid-ipv6-appprefix</field-id>
      <field-length>64</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-ignore</matching-operator>
      <comp-decomp-action>cda-value-sent</comp-decomp-action>
    </entry-universal>
    <entry-universal>
      <entry-index>9</entry-index>
      <field-id>fid-ipv6-appiid</field-id>
      <field-length>64</field-length>
      <field-position>1</field-position>
      <direction-indicator>di-bidirectional</direction-indicator>
      <matching-operator>mo-ignore</matching-operator>
      <comp-decomp-action>cda-value-sent</comp-decomp-action>
    </entry-universal>
  </rule>
  <rule>
    <rule-id-value>12</rule-id-value>
    <rule-id-length>11</rule-id-length>
    <rule-nature>nature-fragmentation</rule-nature>
    <direction>di-up</direction>
    <rcs-algorithm>rcs-crc32</rcs-algorithm>
    <dtag-size>2</dtag-size>
    <fcn-size>3</fcn-size>
    <fragmentation-mode
      >fragmentation-mode-no-ack</fragmentation-mode>
  </rule>
  <rule>
    <rule-id-value>100</rule-id-value>
    <rule-id-length>8</rule-id-length>
    <rule-nature>nature-no-compression</rule-nature>
  </rule>
 </schc>
~~~
{: #Fig-XML-rules title="XML Representation of the Rules"}

# Changes from RFC 9363 {#changes}
{:numbered="false"}

* Header: the document obsoletes RFC 9363, in the `schc` working group.
* Abstract: the document is presented as an update of the SCHC data model
  (protocol options, ICMPv6, Rule management).
* Introduction: RFC 9363 is presented as a data model dedicated to LPWAN
  environments, where the Rules were mostly installed directly in the SCHC
  end-points. The goal becomes to update the formal description of the Rules
  and to extend SCHC to new environments.
* Introduction: the data model also represents the management Rules.
* Introduction: third goal, the management of the Rules by the SCHC instances
  themselves (creation and deletion with CORECONF).
* Introduction: why this document is a revision and not an augmentation of
  RFC 9363 (new key of the compression entries).
* Terminology: new terms Action, Entry Index, Guard Period, Management Rule,
  Rule Status, Space ID, and Universal Option.
* SCHC Rules: fields are also specified by the Universal Option and ICMPv6
  compression documents.
* SCHC Rules: the management can also be selected with a feature statement.
* SCHC Rules: fields for the OSCORE Key Update (KUDOS).
* New section "Convention for Space Identifier": options referenced by a
  Space ID and their option number (`space-id-base-type`, `space-id-coap`,
  `space-id-type`, common ancestor `space-field-id-base-type`), and
  deprecation of the per-option CoAP Field IDs of RFC 9363.
* Convention for Field Length: new functions `fl-length-bytes`,
  `fl-length-bits`, `fl-variable-bits`, and `fl-remaining`; a Field Length
  function can take a numerical argument, carried in the `field-length-value`
  leaf.
* Convention for Matching Operator: new MOs `mo-rule-match` and
  `mo-rev-rule-match`.
* Convention for Compression Decompression Actions: new CDAs
  `cda-compress-sent` and `cda-rev-compress-sent`.
* New section "Convention for Action": `action-forward` is the default
  Action, carried in the `action` leaf, with its arguments in `action-value`.
* Conventions for Field Length, Matching Operator, CDA, and Action: the new
  identities are listed, and their behavior is described in the documents
  that define them.
* New section "Main Updates from RFC 9363": main updates of the YANG data
  model, with the sections describing them and references to the ICMPv6
  compression and CORECONF management documents.
* YANG module: `fid-oscore-base-type`, not used as a base, is deprecated.
* Compression Rule: the `compression-content` grouping is deprecated in favor
  of `compression-content-universal` (Universal Options, `entry-index` key,
  `field-length-value`).
* Compression Rule: the checks also cover the CDA Target Value, already
  checked by the module of RFC 9363, and the nature of the Rules using
  `entry-universal` (compression or management). In `entry-universal`, the
  `field-length-value` is checked for `fl-length-bytes` and `fl-length-bits`,
  and the per-option CoAP Field IDs are rejected. The CDA check no longer
  requires a Target Value for `cda-compress-sent` and
  `cda-rev-compress-sent`.
* Compression Rule: the identities defined by this revision MUST NOT be used
  in the deprecated `entry` list.
* Fragmentation Rule (Rule Definition): unchanged from RFC 9363, augmented by
  RFC 9441 to allow several bitmaps in a SCHC ACK message.
* New section "Management Rule": management Rules, defined in
  draft-toutain-schc-coreconf-management, modify the Set of Rules, except the
  management Rules themselves; the Guard period, global to all the Rules and
  coded as the fragmentation timers; the `rule-status` leaf, present with the
  `management` feature, with the values `status-valid`, `status-candidate`,
  `status-invalid`, and `status-free`; its value is a local view of the
  management protocol machine, and any value sent by the other end is
  overwritten. A Rule with `status-free` is processed as if it did not
  exist in the Set of Rules. Only the Rules with `status-valid` can be used for
  compression and fragmentation. Management Rules are always valid.
* YANG Data Model: the module of RFC 9363 is replaced by the revision
  `ietf-schc@2026-10-27`.
* YANG Data Model: the working revisions of the module are merged into a
  single revision, 2026-10-27, followed by the 2023-03-01 revision of
  RFC 9363. The module passes `pyang --ietf` (canonical order of the
  statements, description of `guard-period`).
* Rule status: new `status-origin`, derived from `status-valid`, for the Rules
  of the initial Set of Rules, which cannot be modified or deleted and are the
  only ones kept by a reset of the context. It is the default value of
  `rule-status`, and the status of the management Rules.
* YANG Data Model: the `duplicate-rule` RPC is removed; it is defined in
  draft-toutain-schc-coreconf-management.
* Example: the compression Rule uses the `entry-universal` list, with an
  `entry-index` per entry; the line breaks inside the `fragmentation-mode`
  value, which made the RFC 9363 example invalid, are moved inside the start
  tag.
* Compression Rule: the deprecation of the per-option CoAP Field IDs (OSCORE
  and KUDOS sub-fields remain current) and the validity of the deprecated
  definitions (RFC 7950, Section 7.21.2); the per-option CoAP Field IDs MUST
  NOT be used in `entry-universal`.
* Rule Definition: fourth nature, `nature-management`, for the management
  Rules.
* YANG Tree: updated to the new module (`context` container, `rule-status`,
  `compression-universal` case with `entry-universal`, and `action`).
* Fragmentation Rule: the `ietf-schc-compound-ack` module of RFC 9441
  augments this data model to acknowledge several windows in a single ACK,
  and remains valid for this revision.
* Convention for Field Identifier: deprecated CoAP Field IDs, new OSCORE and
  KUDOS sub-fields, ICMPv6 Field IDs, and the protocol-independent
  `fid-unused` and `fid-payload`.
* Editorial: the identifiers of the YANG data model are written in courier in
  the text.
* References: RFC 7950 (YANG 1.1) becomes normative, since the deprecation
  rules of its Section 11 are applied.

# Acknowledgments {#acknowledgments}
{:numbered="false"}

The authors would like to thank {{{Dominique Barthel}}}, {{{Carsten Bormann}}}, {{{Ivan Martinez}}}, and {{{Alexander Pelov}}} for their careful reading and valuable inputs. A special thanks for
{{{Joe Clarke}}}, {{{Carl Moberg}}}, {{{Tom Petch}}}, {{{Martin Thomson}}},
and {{{Éric Vyncke}}} for their explanations and wise advice when building the model.
