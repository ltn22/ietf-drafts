---
v: 3

title: CORECONF Rule management for SCHC
abbrev: SCHC for CoAP
docname: draft-toutain-schc-coreconf-management-01
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
      - name: Ana Minaburo
        org: Consultant
        street: Rue de Rennes
        city: Cesson-Sevigne
        code: 35510
        country: France
        email: anaminaburo@gmail.com
      - name: Laurent Toutain
        org: IMT Atlantique
        street: CS 17607, 2 rue de la Chataigneraie
        city: Cesson-Sevigne Cedex
        code: 35576
        country: France
        email: Laurent.Toutain@imt-atlantique.fr
      - name: Javier A. Fernandez
        org: IMT Atlantique
        street: CS 17607, 2 rue de la Chataigneraie
        city: Cesson-Sevigne Cedex
        code: 35510
        country: France
        email: javier-alejandro.fernandez@imt-atlantique.fr
      - name: Marion Dumay
        org: Orange
        email: marion.dumay@orange.com
      - name: Samar Sirohi
        org: Institut MINES TELECOM; IMT Atlantique
        street: 2 rue de la Chataigneraie CS 17607
        city: Cesson-Sevigne Cedex
        code: 35576
        country: France
        email: Samar.Sirohi@imt-atlantique.fr


normative:
  RFC8724:
  RFC9363:
  I-D.toutain-schc-rfc9363bis:
  RFC9254:
  I-D.ietf-core-comi:
  I-D.ietf-lpwan-architecture:
  I-D.toutain-schc-sid-allocation:
informative:

entity:
  SELF: "[RFC-XXXX]"

--- abstract

This document describes how CORECONF can be applied to SCHC for context and rule
set management between endpoints.


--- middle

# Introduction {#intro}

{{RFC8724}} defines three types of Rules (compression, fragmentation and
no-compression) that form a Static Context, also called Set of Rules (SoR). The
Context is called static because the Rules are not modified when a packet is
compressed, decompressed, fragmented or reassembled.

This document adds a fourth type of Rules, called management Rules (or M
Rules), which have the power to add or delete Rules in the SoR. The
nature of these Rules is defined in the SCHC YANG Data Model of
{{I-D.toutain-schc-rfc9363bis}} (module revision 2026-10-27).

This document focuses on the usage of the CoAP Management Interface (CORECONF)
to manage the SoR of a SCHC instance. CORECONF relies on {{RFC9254}}, which
defines how data modeled in YANG are serialized in CBOR, and on
{{I-D.ietf-core-comi}}, which defines the CoAP interface. The document also
specifies the compression Rules dedicated to the management traffic itself.
They make control and configuration exchanges more efficient and are distinct
from the compression applied to regular application data.

# Applicability statement

## Architecture

SCHC instance management allows the two endpoints to modify the common SoR, by:

* Adding a new rule.
* Removing an existing rule.
* Triggering Remote Procedural Calls (RPC) within the endpoints.

Because of propagation delays and of the possible loss of management messages,
both endpoints could temporarily hold different versions of a rule with the
same RuleID. Therefore, an existing rule MUST NOT be modified. To change a rule,
it MUST be deleted and a new rule MUST be created.

A new type of traffic is defined called management traffic, which deals
exclusively with message exchanges concerning context and rule management.

The rule management uses the CORECONF network management interface
{{I-D.ietf-core-comi}}, which is based on CoAP. In this context, management
traffic refers to the CORECONF messages exchanged between the endpoints to
configure or modify rule sets. The management traffic is transported as
SCHC-compressed packets, tagged with specific Rule IDs. These rules are
identified as Management Rules (or M Rules) in Figure {{Fig-SCHCManagement}}. M
Rules can be either Compression Rules or No-Compression Rules. Only M Rules are
permitted to modify the SoR.

The management traffic uses its own IPv6 stack, distinct from regular
application traffic. See Section {{sec-protocols}} for more details.

SCHC Packets using M Rules MUST be encrypted either by the underlying layer (for
instance in a QUIC stream dedicated to management inside a QUIC connection) or
directly using OSCORE or DTLS.

~~~~ aasvg
+-----------------+                 +-----------------+
|       ^         |                 |       ^         |
|  C/D  !M    ___ |                 |       !M    ___ |
|       +-[]>[SoR]|       ...       |       +-[]>[SoR]|
|       !    [___]|                 |       !    [___]|
|       !         |                 |       !         |
|      F/R        |                 |      F/R        |
+------ins_id1----+-----ins_idi-----+------ins_idn----+         
.                   C/D  !    M         +---+    ___  .
.                        +------------->|Mng|<=>[SoR] .    
.                       F/R             +---+   [___] .
+.................. Discriminator ....................+
~~~~
{: #Fig-SCHCManagement title='Inband Management'}

## CoAP Profile

Management requests MUST be protected against packet loss. It is RECOMMENDED to
use CONfirmable requests and no Token. If the management request is too large
regarding the MTU, SCHC Fragmentation SHOULD be used instead of the Block
option. As shown in figure {{Fig-SCHCManagement}} fragmentation can be common to
Management rules and other rules.

{::comment}WARNING (modify): this whole section describes the modification of a
rule (candidate status, Guard period, Fig-Rule-mod). To remove or rewrite as a
deletion followed by a creation.{:/comment}

## Rule modification

SCHC imposes that both ends share exactly the same SoR, therefore, a new or
modified rule cannot be used while it remains in candidate status until the
other end has validated the modification. A candidate rule cannot be used,
either in C/D or F/R. A SCHC PDU MUST NOT be generated with a candidate rule ID
and received PDUs containing a candidate rule ID must be dropped.

~~~ aasvg
              A                         X  B
 X valid      |     modify Rule x    ------| X valid
 X candidate  |=====================/=====>| X candidate
              |        /------------       |
           ---|<======/====================|----
            | |      /                     |  |
     Guard  | |<-----                      |  | Guard
            v |                            |  v
           ---|                            |----   
 X valid      |                            | X valid

~~~
{: #Fig-Rule-mod title='Modifying a rule'}


{{Fig-Rule-mod}} illustrates a Rule modification. The left-hand side entity A
wants to make rule x evolve. It send an acknowledged CoAP message to the other
end. Host A change the status of the rule to "candidate", indicating that the
rule can no longer be used for SCHC procedures. The receiving entity B,
acknowledges the message and continues to maintain the "candidate" status for a
Guard period. At the reception of the acknowledgement, A set also a Guard period
before rule x becomes valid again.

The Guard period is used to avoid SCHC message with a rule ID to appear at the
other end after a rule modification. The Guard period appears only once during
the rule management and is depends on the out-of-sequence messages expected
between both ends.

## Rule creation 

Rule creation do not require a Guard period, and acknowledgement is RECOMMENDED.
Figure {{Fig-Rule-creation}} gives an example, where the Acknowledgment is lost.
Entity A sends a management message to create a new rule. Since its a new rule,
the Guard period is not set and the new rule becomes immediately valid on B. The
Acknowledgement does not reach A, so the rule stays in the candidate state, but
the reception of a SCHC PDU carrying the RulE ID X, informs that the message has
been correctly received by B. So X becomes valid in A.


~~~ aasvg
              A                            B
 X created    |       
 X candidate  |===========================>| X valid
              |         X==================|
              |                            |   
   X valid    |<---------------------------|   
              |               X            | 
              |                            | 

~~~
{: #Fig-Rule-creation title='Modifying a rule'}

## Rule deletion

After the rule deletion, a Guard period is established. During that period, a
rule with the same ID cannot be created, and SCHC PDU corrying the Rule ID are
dropped.

# Management messages

## YANG Data Model
CORECONF proposes an interface to manage data structured with a YANG Data Model.
{::comment}WARNING (modify): 'iPATCH to create, modify or delete a rule': remove
'modify'.{:/comment}

{{RFC9363}} defines a YANG Data Model for SCHC Rules, updated by
{{I-D.toutain-schc-rfc9363bis}} (module revision 2026-10-27). This document uses
the updated model. SCHC Instance Management MUST use FETCH to read a rule and
iPATCH to create, modify or delete a rule. The updated YANG Data Model brings
the following elements used for management:

### Management Rule Nature
M Rules have to be marked in a way that allows quickly identifying which rules
in a SoR are responsible for management. Therefore, a "nature-management" Rule
nature is defined, and the "management" feature enables it. A management Rule
uses the compression Rule structure, but only the "entry-universal" list
supports management Rules: the deprecated "entry" list does not. Compression
needs to be available and activated to do management.

{::comment}WARNING (modify): rule-status and Guard period were introduced for
rule modification. Check what remains useful for creation and deletion
only.{:/comment}

### Rule Status and Guard Period
To determine if a rule is considered available or not during the Guard period, a
rule has a status which determines if it can be used. The "rule-status" leaf is
defined only when the "management" feature is supported. Its type is an
identityref based on "status-base-type", with two identities: "status-active"
(the default), and "status-candidate". An available rule MUST associate the
"rule-status" with the value "status-active". Conversely, during the Guard
period, "rule-status" MUST be set to "status-candidate". A rule without
"rule-status" is considered as "status-active".

The duration of the Guard period is not a property of a single rule. It is
carried by the "guard-period" container of the top-level "context" container, a
sibling of "schc", present when the "management" feature is supported. It
contains the two leaves "ticks-duration" and "ticks-numbers", and applies to the
whole SCHC context.

### YANG tree representation
The YANG tree represents the Rule structure as defined in
{{I-D.toutain-schc-rfc9363bis}}. The deprecated "compression" case (entries
keyed by "field-id", "field-position" and "direction-indicator") is not used for
management and is not shown:

~~~
module: ietf-schc
  +--rw context {management}?
  |  +--rw guard-period
  |     +--rw ticks-duration?   uint8
  |     +--rw ticks-numbers?    uint16
  +--rw schc
     +--rw rule* [rule-id-value rule-id-length]
        +--rw rule-id-value                          uint32
        +--rw rule-id-length                         uint8
        +--rw rule-nature                            nature-type
        +--rw rule-status?                           status-type {management}?
        +--rw (nature)?
           +--:(fragmentation) {fragmentation}?
           |  +--rw fragmentation-mode          schc:fragmentation-mode-type
           |  +--rw l2-word-size?                    uint8
           |  +--rw direction                        schc:di-type
           |  +--rw dtag-size?                       uint8
           |  +--rw w-size?                          uint8
           |  +--rw fcn-size                         uint8
           |  +--rw rcs-algorithm?                   rcs-algorithm-type
           |  +--rw maximum-packet-size?             uint16
           |  +--rw window-size?                     uint16
           |  +--rw max-interleaved-frames?          uint8
           |  +--rw inactivity-timer
           |  |  +--rw ticks-duration?   uint8
           |  |  +--rw ticks-numbers?    uint16
           |  +--rw retransmission-timer
           |  |  +--rw ticks-duration?   uint8
           |  |  +--rw ticks-numbers?    uint16
           |  +--rw max-ack-requests?                uint8
           |  +--rw (mode)?
           |     +--:(no-ack)
           |     +--:(ack-always)
           |     +--:(ack-on-error)
           |        +--rw tile-size?                 uint8
           |        +--rw tile-in-all-1?             schc:all-1-data-type
           |        +--rw ack-behavior?              schc:ack-behavior-type
           |        +--rw bitmap-format?             bitmap-format-type
           |        +--rw last-bitmap-compression?   boolean
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
              +--rw action?                          action-type
              +--rw action-value* [index]
                 +--rw index    uint16
                 +--rw value?   binary
~~~
{: #Fig-tree title='YANG Data Model used for CORECONF'}


## Set of Rules Editing
For clarity reasons, this document will use YANG Identifiers in quotes instead
of SID values. Each entry in the YANG tree has a corresponding SID number. Each
level of the hierarchy is accessible through one or several keys. For example,
to access the hierarchy under "rule", "rule-id-value" and "rule-id-length" must
be specified. To access the hierarchy corresponding to a field entry in a
compression rule, "rule-id-value" and "rule-id-length" must be followed by the
"entry-index" of the "entry-universal" list. Since the TV, MO-value, and
CDA-value are stored as lists, "index" must be added to access a specific
element.

Therefore, to access a specific element in a hierarchy, the SID of this element
has to be specified, followed by the keys needed to access it.

The "entry-index" is the position of the entry in the Rule, and the field is
{::comment}WARNING (modify): 'the field it wants to modify': the entry-index is
also used to address fields for FETCH or deletion. Check the wording.{:/comment}

identified by a leaf ("field-id", or "space-id" and "universal-value") that is
not a key. A manager therefore needs to know the "entry-index" of the field it
wants to modify, for instance after a FETCH of the "field-id" of the Rule
entries.

For example, `["target-value/value", 5, 3, 0, 0]` is used to access the first
value (0) of TV of the first entry (0), for instance the IPv6 Version, of Rule
5/3.

## Management Errors

There are different levels of error detection:

* CORECONF Errors: these errors are directly generated at the CORECONF-managed
  context level. For instance, retrieving a value with a wrong key.
* YANG validation errors: the data model does not conform with constraints such
  as "must" or "mandatory". This check is optional, since it may require a lot
  of resources on a device.
* SCHC errors: errors on the Data Model that cannot be detected at the YANG
  level. For example, the rule numbering does not correspond to a binary tree.

## CoAP Methods

### FETCH

As mentioned in {{I-D.ietf-core-comi}}, FETCH requests are used to retrieve at
least one instance-value.

Example: Fetching the field-id, TV, MO and CDA of the entry 0 (fid-ipv6-version)
from Rule 6/3.

~~~
REQ: FETCH /c
     (Content-Format: application/yang-identifiers+cbor-seq)
["field-id",           6, 3, 0],
["target-value",       6, 3, 0],
["matching-operator",  6, 3, 0],
["comp-decomp-action", 6, 3, 0]

RES: 2.05 Content
     (Content-Format: application/yang-instances+cbor-seq)
{
  {"field-id"           : "fid-ipv6-version"},
  {"target-value"       : [{"index" : 0, "value" : h"06"}]},
  {"matching-operator"  : "mo-equal"},
  {"comp-decomp-action" : "cda-not-sent"}
}
~~~

### iPATCH

{::comment}WARNING (modify): this section (up to 'Adding an element') shows
iPATCH modifying TV, MO and CDA of an existing rule, which is not allowed
anymore.{:/comment}

Several payload formats can be used in a CoAP iPATCH request to modify SCHC rule
parameters. For example, when the entry 3 in Rule 7/8 is configured as
ignore/value-sent and no target value has been defined, the following iPATCH
request payload sets a new Target Value (TV) and updates the corresponding
Matching Operator (MO) and Compression/Decompression Action (CDA):

~~~
  iPATCH /c 
  {
    ["target-value", 7, 8, 3] : [{delta_TV: 0, delta_value: value}],
    ["matching-operator", 7, 8, 3] : "mo-equal",
    ["comp-decomp-action", 7, 8, 3] : "cda-not-sent"
  }
~~~

It is possible to represent each field update as a separate entry in the
payload, as shown above. However, when the modifications apply to elements of
the same subtree, it is RECOMMENDED to group them within a single structure
inside the iPATCH request payload, as shown below:

~~~
  iPATCH /c 
  {
    ["entry-universal", 7, 8, 3] : { 
        delta_target-value       : [{delta_index : 0, delta_value : value}],
        delta_matching-operator  : "mo-equal",
        delta_comp-decomp-action : "cda-not-sent"
    }
}
~~~

Both payload formats are valid encodings for a CoAP iPATCH request. The
interpretation and application of the modifications are implementation-specific.

{::comment}LT: I don't understand - Alejandro: clear now?{:/comment}
The same approach applies to rule updates and YANG leaf-list objects, where
multiple related modifications may be grouped within a single iPATCH request.

#### Adding an element

If the target object, field, or entry does not exist in the SCHC context, it is
added. It supports two main cases:

* Adding a new key-value pair to an existing object.
* Adding a new object to an existing list.

When adding a new element to a YANG leaf-list in the SCHC context, the model
requires that each list index be strictly incremental. CORECONF does not enforce
this automatically; it relies on the client or system to provide correctly
ordered indices.

Example: Add TV into the entry 3 (fid-ipv6-payload-length) in Rule 0/3

~~~
  REQ: iPATCH /c
      (Content-Format: application/yang-identifiers+cbor-seq)
  {
    ["target-value", 0, 3, 3] : [
        {delta_index : 0, delta_value : h"50"},
        {delta_index : 1, delta_value : h"55"}
    ]
  }
  
  RES: 2.04 Changed
~~~

{::comment}WARNING (modify): remove this subsection, or turn it into a
description of what is refused (modification of an existing rule).{:/comment}

#### Modify an element

If the target object, field, or entry does exist in the SCHC context, it is
updated.

Examples:

- The entry 0 (fid-ipv6-version) is in Rule 6/3.

~~~
  REQ: iPATCH /c
      (Content-Format: application/yang-identifiers+cbor-seq)
  {
    ["entry-universal", 6, 3, 0] : {
        {"delta_target-value": []},
        {"delta_matching-operator": "mo-ignore"},
        {"delta_comp-decomp-action": "cda-value-sent"}
    }
  }
  
  RES: 2.04 Changed
~~~


- The entry 0 is not in Rule 7/8 but Rule 7/8 exists. The entry is added, so its
  mandatory leaves (here the field-id, the field-length, the field-position and
  the direction-indicator) have to be provided.

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-identifiers+cbor-seq)
  {
    ["entry-universal", 7, 8, 0] : {
        {"delta_field-id"           : "fid-ipv6-version"},
        {"delta_field-length"       : 4},
        {"delta_field-position"     : 1},
        {"delta_direction-indicator": "di-bidirectional"},
        {"delta_target-value"       : []},
        {"delta_matching-operator"  : "mo-ignore"},
        {"delta_comp-decomp-action" : "cda-value-sent"}
    }
  }
  
  RES: 2.04 Changed
~~~

- The entry 0 is not in Rule 5/8, and Rule 5/8 does not exist. Therefore, Rule
  5/8 cannot be added in order to include the entry 0 because other fields,
  which are not keys, cannot be deducted at every depth of the context.

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-identifiers+cbor-seq)
  {
    ["entry-universal", 5, 8, 0] : {
        {"delta_target-value"       : []},
        {"delta_matching-operator"  : "mo-ignore"},
        {"delta_comp-decomp-action" : "cda-value-sent"}
    }
  }
  
  RES: 4.04 Not Found
~~~

#### Delete an element

If the specified value in the request is "null", it deletes an object, field, or
entry from the SCHC context

Example:

- Delete Rule 7/8

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-identifiers+cbor-seq)
  {
    ["rule", 7, 8]: null
  }
  
  RES: 2.04 Changed
~~~

When deleting objects in the SCHC context via iPATCH, the operation is
restricted to prevent removal of required structural elements. Deleting the
top-level object (`ietf-schc:schc`) does not remove it entirely; instead, the
object is reset to a minimal representation:

~~~
{"ietf-schc:schc": {"rule": []}}
~~~

{::comment}WARNING (modify): 'Updates to existing objects are generally
allowed': contradicts the prohibition of modification.{:/comment}

This ensures the SCHC context remains structurally valid. Updates to existing
objects are generally allowed, but deletion of protected keys is forbidden.

Example: Delete rule-id-value of Rule 0/3

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-identifiers+cbor-seq)
  {
    ["rule-id-value", 0, 3]: null
  }
  
  RES: 4.00 Bad Request
~~~

### POST {#sec-post-method}

As described in {{I-D.ietf-core-comi}}, the POST CoAP method is used to trigger
Remote Procedure Calls (RPC) and other actions within a SCHC endpoint. Thus, a
POST message can be sent to invoke a specific RPC on the remote endpoint.
Details of the supported RPCs and their behavior are defined in Section
{{sec-rpcs}}.

RPCs and actions are defined in a YANG Data Model with optional associated
inputs and outputs, The request payload contains the RPC input map, if any. The
response payload contains the corresponding output map, if any.

{::comment}WARNING (modify): 'fine-grained field updates' and 'only a few fields
need to be updated' refer to modification. Check the whole Optimizations section
(derive-from-existing-rule, entry-index).{:/comment}

### Optimizations

Two optimizations are possible: first, deriving rules to avoid sending the full
object; second, using the entry-index for fine-grained field updates.

#### Derive-from-existing-rule optimization

When sending SCHC rules in iPATCH messages, the naive approach is to include the
full rule object in the payload, even if only a few fields need to be updated.
This can be inefficient, especially in constrained environments. To reduce the
amount of data transmitted, an optimization consists in deriving a new rule from
an existing one and specifying only the fields that are changing.

Therefore, for adding new rules, the RECOMMENDED method is to use the
`duplicate-rule` RPC, defined in Section {{sec-duplicate-rule-rpc}}, which
implements this derivation mechanism efficiently.

#### Entry-index optimization

In {{I-D.toutain-schc-rfc9363bis}}, a field entry of a compression Rule is a
member of the "entry-universal" list, keyed by the single "entry-index" leaf.
The field is identified by a "field-id", or, for a protocol option, by a
"space-id" (the protocol containing the option, e.g., CoAP, QUIC, TCP) and a
"universal-value" (the option itself). These leaves are not keys.

{::comment}WARNING (modify): 'selective field-level updates without touching the
rest of the rule': refers to modification.{:/comment}

An entry is therefore reached with three keys: "rule-id-value", "rule-id-length"
and "entry-index", whatever the protocol and the option. This allows efficient
selective field-level updates without touching the rest of the rule, and keeps
the request short, since the field position and direction do not have to be
repeated in the path.

~~~
REQ: FETCH </c>
        (Content-Format: application/yang-identifiers+cbor-seq)
   ["space-id", 8, 3, 7],
   ["universal-value", 8, 3, 7],
   ["matching-operator", 8, 3, 7]
~~~

The example fetches the protocol and the option, and the Matching Operator, of
the entry 7 of Rule 8/3.

## RPC statements {#sec-rpcs}

A YANG "RPC" is an operation that may be invoked within a SCHC endpoint and
triggers a specified behavior. Within the context of rule management, RPCs are
used to perform actions on the set of rules and may also be used for other
operations, such as rebooting the remote device endpoint.

Each RPC resource has specific inputs and outputs, and may be invoked remotely
via a POST CoAP message, as described in Section {{sec-post-method}}.


### Duplicate Rule {#sec-duplicate-rule-rpc}

To add a new rule, instead of using the iPATCH method with a full rule
{::comment}WARNING (modify): the optional ipatch-sequence modifies the copy
before it is added. Check that it is still consistent with the prohibition (new
rule only).{:/comment}

definition (especially when the new rule is similar to an existing one), the
RECOMMENDED approach is to use the `duplicate-rule` RPC. This operation copies
an existing rule ("from") into a new rule ("to"), and can optionally include an
iPATCH sequence specifying modifications to apply to the duplicated rule. The
output returns a status, an enumeration conveying the result of the operation:
"success-with-validation" (0), "success-not-validated" (1),
"to-rule-already-exists" (-1), "from-rule-not-found" (-2) or "invalid-ipatch"
(-3).

Represented as a tree:

~~~
  rpcs:
    +---x duplicate-rule
       +---w input
       |  +---w from
       |  |  +---w rule-id-value?    uint32
       |  |  +---w rule-id-length?   uint8
       |  +---w to
       |  |  +---w rule-id-value?    uint32
       |  |  +---w rule-id-length?   uint8
       |  +---w ipatch-sequence?   binary
       +--ro output
          +--ro status?   enumeration
~~~~

This mechanism reduces management overhead and addresses the issue of adapting
to variable application traffic. For example, a SCHC instance may begin with a
generic rule with low compression rate, but progressively make rule duplications
to make more specialized rules that better match the observed traffic patterns,
acheiving higher compression rates and thus adapting the rule set dynamically to
the session characteristics.

Example:

- Representation with identifiers for clarity. Delta-encoded SIDs are used in a
  real request.

~~~
REQ:  POST </c>
      (Content-Format: application/yang-instances+cbor-seq)

{
  "/ietf-schc:duplicate-rule":
  {
    "input/from/rule-id-value": 8,
    "input/from/rule-id-length": 4,
    "input/to/rule-id-value": 9,
    "input/to/rule-id-length": 4,
    "input/ipatch-sequence":
      [
        "/ietf-schc:schc/rule/entry-universal/target-value/value", 9, 4,
        14, 0,
      ]: b"\x14\x00"
  }
}

RES:  2.04 Changed
      (Content-Format: application/yang-instances+cbor-seq)

{
  "/ietf-schc:duplicate-rule":
  {
    "output/status": "success-with-validation",
  }
}
~~~

#### Error Handling

The `duplicate-rule` operation SHALL be atomic. If an error occurs during either
stage of the process (rule duplication or subsequent modification through the
iPATCH sequence) the SCHC endpoint MUST revert any partial changes to restore
the previous state. The RPC output MUST indicate the failure with one of the
negative status values ("to-rule-already-exists", "from-rule-not-found" or
"invalid-ipatch"), to signal that the duplication did not take place as
requested. The precise error code and diagnostic message are
implementation-dependent but SHOULD provide enough context for the management
entity to identify the cause of failure.

# Protocol Stack {#sec-protocols}

The management inside the instance has its own IPv6 stack, independent of the
application traffic. IPv6/UDP/CoAP is used to allow the implementation of the
CORECONF interface. No other kind of traffic is allowed.

The end-point acting as a Device has the IPv6 address fe80::1/64 and the other
end, the Core, is assigned the address fe80::2/64.

Both endpoints implement CoAP client and server capabilities, that is, both
endpoints are capable of sending requests and processing responses. The server
uses port 5683 and the client 3865.

## Management Compression Rules (M Rules)

To enable CORECONF-based context and rule management over SCHC, a set of
dedicated management rules, identified as M rules, is defined. These rules are
used exclusively for management traffic, that is, packets exchanged between SCHC
endpoints for the purpose of managing rules, not for application data transfer.

This specification introduces four rules, allowing bidirectional operation and
fine control over management capabilities. Each rule defines the compression
behavior for management messages in its direction, distinguishing between
requests and responses.

* M1: Handles packets containing a payload (e.g., CoAP requests or Content
  responses) in one direction (Uplink).
* M2: Handles packets without a payload (e.g., CoAP responses) in the same
  direction (Uplink).
* M3: Mirrors M1 in the opposite direction (Downlink), for payload-bearing
  management messages.
* M4: Mirrors M2 in the opposite direction (Downlink), for payloadless messages.

Implementations MAY choose to support only a subset of these rules, depending on
their operational or security requirements. For instance, an implementation may
include only M3 and M4 to permit management operations exclusively from one
endpoint, effectively preventing unsolicited management requests in the other
direction. In this sense, the absence of certain M rules in the SoR implicitly
acts as a policy mechanism or safeguard for rule management operations.

M rules are protected elements within the SoR. They define the operation of the
management channel itself and therefore MUST NOT be modified, duplicated, or
deleted through CORECONF operations. Any attempt to apply a modification or
duplication request to an M rule MUST result in an Unauthorized error response.
This restriction ensures the integrity and stability of the SCHC management
process.

~~~
 +-------------------------------------------------------------------+
 |RuleID M1                                                          |
 +-------------------+--+--+--+-----------+-------------+------------+
 |        FID        |FL|FP|DI|  TV       |     MO      |    CDA     |
 +-------------------+--+--+--+-----------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6          |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1          |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470     |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|           |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17         |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64         |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64  |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::2        |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64  |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::1        |equal        |not-sent    |
 +===================+==+==+==+===========+=============+============+
 |UDP DevPort        |16|1 |Bi|3865       |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|5683       |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|           |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|           |ignore       |compute-*   |
 +===================+==+==+==+===========+=============+============+
 |CoAP Version       |2 |1 |Bi|1          |equal        |not-sent    |
 |CoAP Type          |2 |1 |Dw|2          |equal        |not-sent    |
 |CoAP Type          |2 |1 |Up|0          |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0          |equal        |not-sent    |
 |CoAP Code          |8 |1 |Up|[2, 5, 7]  |match-mapping|mapping-sent|
 |CoAP Code          |8 |1 |Dw|69         |equal        |not-sent    |
 |CoAP MID           |16|1 |Bi|0          |MSB(9)       |LSB         |
 |CoAP Uri-Path      |8 |1 |Bi|c          |equal        |not-sent    |
 |CoAP Content-Format|8 |1 |Bi|application|equal        |not-sent    |
 |                   |  |  |  |/yang-ident|             |            |
 |                   |  |  |  |fiers+cbor-|             |            |
 |                   |  |  |  |seq        |             |            |
 +===================+==+==+==+===========+=============+============+
~~~
{: #Fig-M1 title='Management Rule 1'}

~~~
 +----------------------------------------------------------------------+
 |RuleID M2                                                             |
 +-------------------+--+--+--+--------------+-------------+------------+
 |        FID        |FL|FP|DI|      TV      |     MO      |    CDA     |
 +-------------------+--+--+--+--------------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6             |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1             |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470        |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|              |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17            |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64            |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64     |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::2           |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64     |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::1           |equal        |not-sent    |
 +===================+==+==+==+==============+=============+============+
 |UDP DevPort        |16|1 |Bi|3865          |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|5683          |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|              |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|              |ignore       |compute-*   |
 +===================+==+==+==+==============+=============+============+
 |CoAP Version       |2 |1 |Bi|1             |equal        |not-sent    |
 |CoAP Type          |2 |1 |Dw|2             |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0             |equal        |not-sent    |
 |CoAP Code          |8 |1 |Dw|[68, 128, 129,|match-mapping|mapping-sent|
 |                   |  |  |  | 132, 160]    |             |            |
 |CoAP MID           |16|1 |Bi|0             |MSB(9)       |LSB         |
 +===================+==+==+==+==============+=============+============+
~~~
{: #Fig-M2 title='Management Rule 2'}

~~~
 +-------------------------------------------------------------------+
 |RuleID M3                                                          |
 +-------------------+--+--+--+-----------+-------------+------------+
 |        FID        |FL|FP|DI|  TV       |     MO      |    CDA     |
 +-------------------+--+--+--+-----------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6          |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1          |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470     |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|           |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17         |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64         |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64  |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::2        |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64  |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::1        |equal        |not-sent    |
 +===================+==+==+==+===========+=============+============+
 |UDP DevPort        |16|1 |Bi|3865       |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|5683       |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|           |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|           |ignore       |compute-*   |
 +===================+==+==+==+===========+=============+============+
 |CoAP Version       |2 |1 |Bi|1          |equal        |not-sent    |
 |CoAP Type          |2 |1 |Up|2          |equal        |not-sent    |
 |CoAP Type          |2 |1 |Dw|0          |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0          |equal        |not-sent    |
 |CoAP Code          |8 |1 |Dw|[2, 5, 7]  |match-mapping|mapping-sent|
 |CoAP Code          |8 |1 |Up|69         |equal        |not-sent    |
 |CoAP MID           |16|1 |Bi|0          |MSB(9)       |LSB         |
 |CoAP Uri-Path      |8 |1 |Bi|c          |equal        |not-sent    |
 |CoAP Content-Format|8 |1 |Bi|application|equal        |not-sent    |
 |                   |  |  |  |/yang-ident|             |            |
 |                   |  |  |  |fiers+cbor-|             |            |
 |                   |  |  |  |seq        |             |            |
 +===================+==+==+==+===========+=============+============+
~~~
{: #Fig-M3 title='Management Rule 3'}

~~~
 +----------------------------------------------------------------------+
 |RuleID M4                                                             |
 +-------------------+--+--+--+--------------+-------------+------------+
 |        FID        |FL|FP|DI|      TV      |     MO      |    CDA     |
 +-------------------+--+--+--+--------------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6             |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1             |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470        |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|              |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17            |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64            |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64     |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::2           |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64     |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::1           |equal        |not-sent    |
 +===================+==+==+==+==============+=============+============+
 |UDP DevPort        |16|1 |Bi|3865          |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|5683          |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|              |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|              |ignore       |compute-*   |
 +===================+==+==+==+==============+=============+============+
 |CoAP Version       |2 |1 |Bi|1             |equal        |not-sent    |
 |CoAP Type          |2 |1 |Up|2             |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0             |equal        |not-sent    |
 |CoAP Code          |8 |1 |Up|[68, 128, 129,|match-mapping|mapping-sent|
 |                   |  |  |  | 132, 160]    |             |            |
 |CoAP MID           |16|1 |Bi|0             |MSB(9)       |LSB         |
 +===================+==+==+==+==============+=============+============+
~~~
{: #Fig-M4 title='Management Rule 4'}

# OSCORE

## Compression Rules

# DTLS

## Compression Rules


# Example CORECONF usage in Python

In these examples, the entries of the Rules are numbered in the order of the
IPv6 header (entry 0 is the IPv6 Version, 1 the Traffic Class, 2 the Flow Label,
3 the Payload Length). The SID values and the CBOR encodings depend on the SID
file of the module revision 2026-10-27 and are not shown.

## Deletion cases

* Delete root element:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc'): None
  } 
  
  RES: 2.04 Changed
  ~~~

* Delete a specific rule:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule', 0, 3): None
  } 
  
  RES: 2.04 Changed
  ~~~
  
* Delete a specific protocol field entry:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal', 0, 3, 0): None
  } 
  
  RES: 2.04 Changed
  ~~~
  
* Delete a specific key:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/rule-status', 0, 3): None
  } 
  
  RES: 2.04 Changed
  ~~~
  
* Delete a list element:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal/target-value/value',
     0, 3, 0, 0): None
  } 
  
  RES: 2.04 Changed
  ~~~
  
* Delete a multiple list elements:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal/target-value/value',
     0, 3, 1, 1): None
  } 
  
  RES: 2.04 Changed
  ~~~
  
* Delete an unknown entry:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal', 2, 3, 0): None
  } 
  
  RES: 4.00 Bad Request
  ~~~
  
* Delete a protected key:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/rule-id-value', 0, 3): None
  } 
  
  RES: 4.00 Bad Request
  ~~~

{::comment}WARNING (modify): remove: modification of an existing rule is not
allowed (rule-id-value, rule-status updates).{:/comment}

## Update cases

* Update protected key:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/rule-id-value', 0, 3): 5
  } 
  
  RES: 2.04 Changed
  ~~~

* Update a specific key:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/rule-status', 0, 3): 'status-candidate'
  } 
  
  RES: 2.04 Changed
  ~~~

## Addition cases

* Add a new entry:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal', 0, 3, 10): {
        'field-id': 'ietf-schc:fid-ipv6-appprefix',
        'field-length': 64, 
        'field-position': 1,
        'direction-indicator': 'ietf-schc:di-bidirectional',
        'target-value': [{'index': 0, 'value': '/oAAAAAAAAA='}], 
        'matching-operator': 'ietf-schc:mo-equal', 
        'comp-decomp-action': 'ietf-schc:cda-not-sent'
    }
  } 
  
  RES: 2.04 Changed
  ~~~

* Add a list element (auto-indexed)
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal/target-value', 0, 3, 2): {
        'index': 4, 'value': 'vLw='
    }
  } 
  
  RES: 2.04 Changed
  ~~~

* Add a list element (explicit index):
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal/target-value', 0, 3, 2): {
        'index': 7, 'value': 'vLw='
    }
  } 
  
  RES: 2.04 Changed
  ~~~

* Add a new key–value pair element:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal/target-value', 0, 3, 3): [
        {'index': 0, 'value': 'UA=='}, 
        {'index': 1, 'value': 'VQ=='}
    ]
  } 
  
  RES: 2.04 Changed
  ~~~

* Add a new rule:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule', 5, 3): {
        'rule-status': 'ietf-schc:status-active', 
        'rule-id-value': 10, 
        'rule-id-length': 5, 
        'rule-nature': 'ietf-schc:nature-compression'
    }
  } 
  
  RES: 2.04 Changed
  ~~~

* Add an entry into an unknown rule:
  
  ~~~
  YANG REQ: iPATCH /c
  {
    ('/ietf-schc:schc/rule/entry-universal', 250, 8, 3): {
        'field-id': 'ietf-schc:fid-ipv6-payload-length',
        'field-length': 16, 
        'field-position': 1,
        'direction-indicator': 'ietf-schc:di-bidirectional',
        'matching-operator': 'ietf-schc:mo-ignore', 
        'comp-decomp-action': 'ietf-schc:cda-value-sent'
    }
  } 
  
  RES: 4.00 Bad Request
  ~~~

--- back


# YANG DM

The complete "ietf-schc" module, including the "management" feature, the
"rule-status" leaf, the "context" container and the "duplicate-rule" RPC, is
defined in {{I-D.toutain-schc-rfc9363bis}} (revision 2026-10-27). For reference,
the RPC is:

~~~ yang
rpc duplicate-rule {
  input {
    container from {
      uses schc:rule-id-type;
    }
    container to {
      uses schc:rule-id-type;
    }
    leaf ipatch-sequence {
      type binary;
    }
  }
  output {
    leaf status {
      type enumeration {
        enum success-with-validation { value 0; }
        enum success-not-validated   { value 1; }
        enum to-rule-already-exists  { value -1; }
        enum from-rule-not-found     { value -2; }
        enum invalid-ipatch          { value -3; }
      }
    }
  }
}
~~~

# Changes {#changes}

* {{Fig-tree}}: the YANG tree is the one of {{I-D.toutain-schc-rfc9363bis}}
  (module revision 2026-10-27). The "entry" list keyed by "field-id",
  "field-position" and "direction-indicator" is replaced by "entry-universal",
  keyed by "entry-index"; "action" and "action-value" are added; the "context"
  container is added.
* "Rule Status and Guard Period": "rule-status" is now defined (identityref
  "status-type", "status-active" and "status-candidate") under the "management"
  feature, and the guard period duration is described by "context/guard-period".
* "Management Rule Nature": the "nature-management" nature uses the
  "entry-universal" list only.
* "Set of Rules Editing", FETCH and iPATCH examples: the keys "field-id",
  "field-position" and "direction-indicator" are replaced by the "entry-index";
  entries created by iPATCH carry their mandatory leaves.
* "Entry-index optimization" replaces "Universal-options optimization", since
  the universal options are part of the "entry-universal" list.
* "Duplicate Rule": the status is an enumeration, the example uses an
  "entry-index", and the YANG fragment of the annex is aligned with the module.
* "Example CORECONF usage in Python": the YANG requests use the
  "entry-universal" paths; the SID values and CBOR encodings, which depend on
  the SID file of the new module revision, have been removed and have to be
  regenerated.
* Samar Sirohi added as co-author.
* WARNING comments ("WARNING (modify)") mark the places that still describe
  the modification of an existing rule, which is no longer allowed.
* Introduction: now starts with the three Rule types of {{RFC8724}} and the
  new management Rules, which {{I-D.toutain-schc-rfc9363bis}} introduces in the
  YANG Data Model.
* Corentin Banier moved from the authors to the acknowledgments, and the SCHC
  Chair and Claude were added to the acknowledgments.
* Reference to {{I-D.toutain-schc-rfc9363bis}} added, and to the
  universal-option draft removed.

# Acknowledgments # {#acknowledgments}
{:unnumbered}

The authors sincerely thank Corentin Banier for his contributions to this work.

This work was supported by the Sweden's Innovation Agency VINNOVA within the
EUREKA CELTIC-NEXT project CYPRESS.

This work has been supported by the SCHC Chair from IMT Atlantique and Afnic.

The authors also thank Claude (Anthropic) for rewriting parts of this document
in more correct English and for checking its consistency with the YANG data
model.
