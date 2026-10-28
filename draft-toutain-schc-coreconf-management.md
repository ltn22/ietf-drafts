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
  RFC7252:
  I-D.ietf-lpwan-architecture:
  I-D.toutain-schc-sid-allocation:
informative:
  RFC8613:
  RFC9175:

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

This document adds a fourth type of Rules, called management Rules (or M Rules),
which have the power to add or delete Rules in the SoR. The nature of these
Rules is defined in the SCHC YANG Data Model of {{I-D.toutain-schc-rfc9363bis}}
(module revision 2026-10-27).

This document focuses on the usage of the CoAP Management Interface (CORECONF)
to manage the SoR of a SCHC instance. CORECONF relies on {{RFC9254}}, which
defines how data modeled in YANG are serialized in CBOR, and on
{{I-D.ietf-core-comi}}, which defines the CoAP interface. The document also
specifies the compression Rules dedicated to the management traffic itself. They
make control and configuration exchanges more efficient and are distinct from
the compression applied to regular application data.

# Applicability statement

## Architecture

SCHC instance management allows the two endpoints to modify the common SoR, by:

* Adding a new rule.
* Removing an existing rule.
* Triggering Remote Procedural Calls (RPC) within the endpoints.

Because of propagation delays and of the possible loss of management messages,
both endpoints could temporarily hold different versions of a rule with the same
RuleID. Therefore, an existing rule MUST NOT be modified. To change a rule, it
MUST be deleted and a new rule MUST be created.

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


# Rule status

In {{RFC8724}} and its associated Data Model in {{RFC9363}}, rules are
established when the SCHC instance is configured and are not modified
afterwards. With management, rules can be created and deleted while the instance
is running. A new state is therefore introduced for the rules, the
"rule-status". It is used for the creation and the deletion of rules, to
preserve the atomicity of the SoR: both ends move together from one version of
the SoR to the next, and a rule is never used by one end while the other end
does not have it.

A rule can have several statuses:

* `valid`: the rule belongs to the SoR and can be used for C/D and F/R.
* `candidate`: the rule has been created, but the other end has not validated it
  yet. A SCHC PDU MUST NOT be generated with a `candidate` rule, neither for C/D
  nor for F/R. A `candidate` rule becomes `valid` either if the management
  message is positively acknowledged or the initiator receives a SCHC PDU with
  the newly created rule ID (the acknowledgment has been lost). This PDU is
  accepted and decoded as for a `valid` rule, since only an end that has
  validated the rule can have generated it.
* `invalid`: the rule has been deleted and the Guard period is running. The rule
  cannot be used, a rule with the same ID cannot be created, and the SCHC PDUs
  carrying its rule ID are dropped. The Guard period of the end that deletes the
  rule starts when it receives the acknowledgement; the Guard period of the
  other end starts when it receives the deletion. When the Guard period is over,
  the rule ID becomes `free`.
* `free`: the rule ID can be used to create a new rule. When the Guard period of
  a deleted rule is over, the rule is either marked `free` in the SoR, or
  removed from the SoR. A rule ID absent from the SoR is also `free`. A new rule
  ID MUST NOT be equal to, a prefix of, or prefixed by the rule ID of a `valid`,
  `candidate` or `invalid` rule, since the SCHC packets could not be parsed
  without ambiguity.



~~~ aasvg
              A                            B
  X valid     |                            | X valid
              |               /------------|  SCHC PDU (X)
  X invalid   |==============/============>| X invalid
              |<============/==============|---
           ---|            /      Ack      | |
            | |           /                | |
      Guard | |          /                 | |  Guard
            | |X========/                  | |
            v | dropped                    | v
           ---|                            |---
  X free      |                            | X free
              |        create Rule X       |
  X candidate |===========================>| X valid
              |                            | 
  X valid     |<===========================|
              |                   Ack      |
~~~
{: #Fig-Rule-del title='Deleting and re-creating a rule'}

{{Fig-Rule-del}} illustrates the deletion and the re-creation of rule X. Entity
A sends an acknowledged CoAP message to B to delete rule X. A marks the rule
`invalid` when it sends the request, but its Guard period starts only when it
receives the acknowledgement. B marks the rule `invalid` and starts its Guard
period when it receives the request. Meanwhile, B has sent a SCHC PDU built with
rule X before receiving the request. This PDU arrives at A during the Guard
period and is dropped, since rule X is `invalid`. When the Guard period is over,
rule X is `free` and can be created again, as shown by the last messages: A
creates X as `candidate`, B creates it as `valid`, and A validates it when it
receives the acknowledgement.

The Guard period is used to avoid a SCHC PDU carrying a rule ID to appear at the
other end after the rule has been deleted and, possibly, re-created with another
content. Its duration is computed in {{sec-guard-computation}}.

## Rule creation

Rule creation does not require a Guard period, and acknowledgement is
RECOMMENDED. Figure {{Fig-Rule-creation}} gives an example, where the
Acknowledgment is lost. Entity A sends a management message to create a new
rule. Since it is a new rule, the Guard period is not set and the new rule
becomes immediately `valid` on B. The Acknowledgement does not reach A, so the
rule stays in the `candidate` state, but the reception of a SCHC PDU carrying
the Rule ID X informs that the message has been correctly received by B. This
PDU is accepted and decoded, and X becomes `valid` in A.

~~~ aasvg
              A                            B
 X free       |
 X candidate  |===========================>| X valid
              |         X==================|
              |                            |   
   X valid    |<---------------------------|   
              |               X            | 
              |                            | 

~~~
{: #Fig-Rule-creation title='Creating a rule'}

### Idempotent creation {#sec-idempotent-creation}

The acknowledgement of a creation may be lost without a SCHC PDU using the new
rule being sent, or the CoAP exchange may be abandoned by A. Then A cannot know
if B has created the rule, and the rule stays `candidate` on A. To recover, A
repeats the creation. A creation is idempotent:

* if the rule does not exist on B (`free`), B creates it as `valid` and
  acknowledges it;
* if a `valid` rule with the same rule ID and exactly the same content already
  exists on B, B does not change anything and sends the same positive
  acknowledgement as for the first request;
* if a `valid` rule with the same rule ID but a different content exists on B,
  or a rule whose rule ID overlaps (see `free`), B refuses the request with 4.09
  Conflict. A discards its `candidate` rule (the rule becomes `free`), and may
  retry later with another rule ID or content;
* if the rule is `candidate` on B, the creations of both ends cross (see
  {{sec-simultaneous-creation}});
* if the rule ID is `invalid` on B, or overlaps an `invalid` rule ID (the Guard
  period is running), B answers 5.03 Service Unavailable, with a Max-Age option
  giving the remaining duration of its Guard period. A keeps the rule
  `candidate` and repeats the creation when this delay has elapsed.

A 4.09 Conflict must not be used during the Guard period: a copy of the creation
sent in a previous CoAP exchange may still be in flight, and be accepted by B
when its Guard period is over, while A would already have discarded the rule. An
answer is taken into account only if the rule is still the `candidate` rule
created by the request. Otherwise (the rule has been validated by a SCHC PDU,
replaced or deleted in the meantime), it is ignored.

The initiator keeps the rule `candidate` and repeats the creation, with an
increasing delay, until it receives an answer. The creation stops as soon as the
rule is no longer `candidate`: validated by a SCHC PDU, replaced (see
{{sec-simultaneous-creation}}) or deleted. Repeating it afterwards could create
the rule again on the other end after its deletion. To give up a `candidate`
rule, A deletes it as any other rule (see {{sec-rule-deletion}}): the deletion
is also idempotent, so B answers positively even if it never received the
creation.

The following example shows the creation of rule 9/4, where the acknowledgement
is lost and the rule is not used by B before A repeats the request:

~~~
A -> B  REQ: iPATCH /c
        (Content-Format: application/yang-identifiers+cbor-seq)
        ["rule", 9, 4]: { "rule-nature": "nature-compression", ... }
        A: rule 9/4 candidate
        B: rule 9/4 free -> valid
B -> A  RES: 2.04 Changed                                   (lost)
        A: CoAP exchange abandoned, rule 9/4 stays candidate

A -> B  REQ: iPATCH /c   (same content, new CoAP exchange)
        ["rule", 9, 4]: { "rule-nature": "nature-compression", ... }
        B: rule 9/4 already valid with the same content, no change
B -> A  RES: 2.04 Changed
        A: rule 9/4 candidate -> valid
~~~
{: #Fig-idempotent title='Idempotent creation after a lost acknowledgement'}

If the second request had a different content, B would answer 4.09 Conflict and
rule 9/4 would stay `valid` on B with its original content. A would discard its
`candidate` rule (it becomes `free`), and would have to delete rule 9/4 before
creating it with the new content.

### Simultaneous creation {#sec-simultaneous-creation}

Both ends can create rules. They could choose the same rule ID at the same time,
or two rule IDs where one is the prefix of the other. To avoid it, the rule ID
space SHOULD be shared between both ends:

* the rule IDs of the rules created by the Core start with a bit set to 0;
* the rule IDs of the rules created by the Device start with a bit set to 1.

With this convention, two rule IDs created by different ends are never equal,
nor the prefix of each other. It applies to the rules created with an iPATCH and
with the "duplicate-rule" RPC. The rules provisioned before the start of the
SCHC instance, such as the M Rules, are not concerned.

If both creations still cross, for instance when this convention is not used,
the Device has priority:

* the Device, receiving a creation for a rule ID that is `candidate` on the
  Device, or that overlaps with such a rule ID, refuses it with 4.09 Conflict;
* the Core, receiving a creation for a rule ID that is `candidate` on the Core,
  or that overlaps with such a rule ID, accepts it: the `candidate` rule of the
  Core is discarded, and the received rule becomes `valid`. The Core stops its
  own creation exchange, and the 4.09 Conflict sent by the Device for it is
  ignored, since the rule is no longer the `candidate` rule created by the
  request.

The Device has priority because retrying with another rule ID is less costly for
the Core than for a constrained Device.

~~~ aasvg
            Core                        Device
 X candidate  |==========\      /==========| X candidate
     (v1)     |           \    /           |     (v2)
              | create     \  /    create  |
              |   v1        \/       v2    |
              |             /\             |
              |            /  \            |
              |           /    \           |
 X valid      |<=========/      \=========>| rejects v1
     (v2)     |                            |
              |======== 2.04 Changed =====>| X valid (v2)
              |                            |
 ignored      |<===== 4.09 Conflict =======|
~~~
{: #Fig-simultaneous title='Crossing creations of the same rule ID'}

{{Fig-simultaneous}} shows two crossing creations of rule X. The Core and the
Device both create X as `candidate`, with different contents (v1 and v2). The
Device refuses the creation of the Core with 4.09 Conflict. The Core accepts the
creation of the Device: X becomes `valid` with the content v2 on the Core, which
acknowledges it. The Device validates X when it receives the acknowledgement,
and the 4.09 Conflict received by the Core is ignored. If the 4.09 Conflict
reaches the Core before the creation of the Device, the `candidate` rule of the
Core becomes `free` first, and the creation of the Device is then accepted as
any creation.

## Rule deletion {#sec-rule-deletion}

When a rule is deleted, it becomes `invalid` on both ends for a Guard period
(see {{Fig-Rule-del}}). During that period, a rule with the same ID cannot be
created, and the SCHC PDUs carrying the Rule ID are dropped.

The initiator marks the rule `invalid` when it sends the deletion, so that it
does not generate any more PDU with this rule. Its Guard period starts when it
receives the acknowledgement. If the deletion or its acknowledgement is lost and
the CoAP exchange is abandoned, the rule stays `invalid` on the initiator
without any running Guard period, and the initiator MUST repeat the deletion
until it receives the acknowledgement. In the meantime, the SCHC PDUs sent by
the other end with this rule ID are dropped.

The deletion is idempotent: if the rule is already `invalid` or `free` on the
receiving end, the request is acknowledged as the first one and does not restart
the Guard period. A deletion received for a `candidate` rule makes it `invalid`,
as for a `valid` rule, and the creation exchange in progress for this rule is
stopped. The other end may have validated the rule before deleting it, or the
deletion may be a late copy of an older one; in both cases, the rule can be
created again after the Guard period.

Management requests MUST be processed only once, as specified in {{RFC7252}},
Section 4.5. The relaxation allowed by this section for idempotent requests does
not apply: a creation or a deletion received again after the rule has been
deleted and created again is not idempotent.

## Guard period computation {#sec-guard-computation}

The Guard period must be longer than the time a SCHC PDU built with the deleted
rule can stay in the network. Let L be the maximum lifetime of a SCHC PDU: the
maximum time between its emission by an end and its delivery (or discard) by the
other end, including the queueing in the underlying network and, for a
fragmentation rule, the time to deliver the last fragment. L depends on the
network and is a configuration parameter. Consider a rule X deleted by A at time
tA (A sends the request) and received by B at time tB, with tB >= tA:

* A does not generate any PDU with X after tA. The last PDU of A reaches B at
  tA+L at the latest, that is at most L after the beginning of the Guard period
  of B (tB). B drops it. A Guard period of B longer than L is therefore enough.
* B generates PDUs with X until tB. The last PDU of B reaches A at tB+L at the
  latest. A receives the acknowledgement, sent after tB, at a time tA' >= tB,
  and starts its Guard period at tA'. A Guard period of A longer than L is
  therefore enough, since tA'+L >= tB+L.

Both ends then use the same Guard period

    G >= L

plus a margin for the clock drift. For example, in a network where a SCHC PDU is
delivered in less than 30 s, G = 30 s is enough, whatever the number of
retransmissions of the CoAP deletion, since they happen before the Guard period
of A starts.

Starting the Guard period of A when it sends the request would not be
sufficient: the request may be retransmitted for the CoAP transmission span (45
seconds with the default parameters of {{RFC7252}}) before B receives it, and B
keeps generating PDUs with X during all that time. A stale PDU would then arrive
after the Guard period of A, possibly when X has been created again with another
content: A in the `candidate` state would validate the new rule on this stale
PDU, and would decode the following PDUs with the wrong rule.

The Guard period of A ends at least as late as the one of B. A creation
initiated by A therefore reaches a B whose Guard period is over. A creation
initiated by B may arrive when the Guard period of A is still running: A answers
5.03 Service Unavailable, and B retries when the Guard period of A is over (see
{{sec-idempotent-creation}}).

## Restart of an end {#sec-restart}

Since management changes the SoR while the SCHC instance is running, the rules
and their status SHOULD be stored in a stable memory, which survives a restart.
Otherwise, an end that restarts loses the rules created or deleted by
management, while the other end still uses them, and an `invalid` rule would be
considered as `free` before the end of its Guard period. When the rules are
kept, after a restart:

* a `candidate` rule stays `candidate`, and the creation is repeated;
* an `invalid` rule whose deletion has not been acknowledged stays `invalid`,
  and the deletion is repeated;
* an `invalid` rule whose Guard period had started begins a new, complete Guard
  period, since the time spent during the restart is unknown. A longer Guard
  period is always safe: a creation received during this period is answered with
  5.03 Service Unavailable (see {{sec-idempotent-creation}}).

The deduplication state of CoAP (see {{sec-rule-deletion}}) is usually lost
during a restart, so that a copy of a management request sent before the restart
could be processed again. If this state is not kept in a stable memory, the end
MUST either check the freshness of the management requests with the Echo option
{{RFC9175}}, or answer them with 5.03 Service Unavailable during
EXCHANGE_LIFETIME ({{RFC7252}}) after the restart, so that the copies sent
before the restart are no longer in flight. When OSCORE is used, the recovery of
its replay window after a restart (Appendix B.1.2 of {{RFC8613}}) relies on the
same Echo mechanism.

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
The status of a rule determines if it can be used (see "Rule status"). The
"rule-status" leaf is defined only when the "management" feature is supported.
Its type is an identityref based on "status-base-type", with four identities
corresponding to the statuses `valid`, `candidate`, `invalid` and `free`:
"status-valid" (the default), "status-candidate", "status-invalid" and
"status-free". A rule without "rule-status" is considered as "status-valid".

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
        'rule-status': 'ietf-schc:status-valid', 
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
  "status-type", "status-valid", "status-candidate", "status-invalid" and
  "status-free") under the "management" feature, and the guard period duration
  is described by "context/guard-period".
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
* WARNING comments ("WARNING (modify)") mark the places that still describe the
  modification of an existing rule, which is no longer allowed.
* "Rule status": added the list of the statuses a rule can have (valid,
  candidate, invalid and free).
* "Rule status": the Guard period of the initiator of a deletion starts when it
  receives the acknowledgement (not when it sends the request), and
  {{Fig-Rule-del}} is updated. A PDU received with a `candidate` rule ID is now
  accepted and validates the rule, instead of being dropped.
* "Rule creation": added the idempotent creation ({{sec-idempotent-creation}})
  with an example, the typos were fixed, and the title of {{Fig-Rule-creation}}
  was corrected.
* "Rule deletion": the deletion is idempotent, and is repeated until it is
  acknowledged.
* "Guard period computation" ({{sec-guard-computation}}) added. {{RFC7252}}
  added to the references.
* "Rule status": new "Simultaneous creation" subsection
  ({{sec-simultaneous-creation}}): the rule ID space is shared between the Core
  (first bit 0) and the Device (first bit 1), and the Device has priority when
  two creations cross. A creation received during the Guard period is answered
  with 5.03 Service Unavailable instead of 4.09 Conflict. A deletion received
  for a `candidate` rule makes it `invalid`. A creation stops as soon as the
  rule is no longer `candidate`, and an answer is only taken into account for
  the `candidate` rule it was sent for. Management requests must be
  deduplicated. A new rule ID must not overlap an existing one. The alignment of
  the "X free" label in {{Fig-Rule-creation}} was fixed.
* "Restart of an end" ({{sec-restart}}) added: the rules and their status should
  be kept in a stable memory, the Guard period of an `invalid` rule restarts
  after a restart, and the freshness of the management requests is checked when
  the deduplication state is lost. {{RFC8613}} and {{RFC9175}} added to the
  informative references.
* Introduction: now starts with the three Rule types of {{RFC8724}} and the new
  management Rules, which {{I-D.toutain-schc-rfc9363bis}} introduces in the YANG
  Data Model.
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
