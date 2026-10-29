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
        org: IMT Atlantique
        street: 2 rue de la Chataigneraie CS 17607
        city: Cesson-Sevigne Cedex
        code: 35576
        country: France
        email: Samar.Sirohi@imt-atlantique.fr


normative:
  RFC2119:
  RFC8174:
  RFC3688:
  RFC6020:
  RFC7950:
  RFC8724:
  RFC9363:
  I-D.toutain-schc-rfc9363bis:
  RFC9254:
  I-D.ietf-core-comi:
  RFC7252:
  RFC0768:
  RFC8200:
  RFC8613:
  I-D.ietf-lpwan-architecture:
  I-D.toutain-schc-sid-allocation:
informative:
  RFC8340:

entity:
  SELF: "[RFC-XXXX]"

--- abstract

This document describes how CORECONF can be applied to SCHC for context and rule
set management between endpoints. It also defines a YANG module with an RPC to
duplicate an existing Rule and apply some modifications, and an RPC to reset the
Set of Rules to its initial content.


--- middle

# Introduction {#intro}

{{RFC8724}} defines three types of Rules (compression, fragmentation and
no-compression) that form a Static Context, also called Set of Rules (SoR). The
Context is called static because the Rules are not modified when a packet is
compressed, decompressed, fragmented or reassembled.

This document adds a fourth type of Rules, called management rules, which have
the power to add or delete Rules in the SoR. The nature of these Rules is
defined in the SCHC YANG Data Model of {{I-D.toutain-schc-rfc9363bis}} (module
revision 2026-10-27).

This document focuses on the usage of the CoAP Management Interface (CORECONF)
to manage the SoR of a SCHC instance. CORECONF relies on {{RFC9254}}, which
defines how data modeled in YANG are serialized in CBOR, and on
{{I-D.ietf-core-comi}}, which defines the CoAP interface. The document also
specifies the compression Rules dedicated to the management traffic itself. They
make control and configuration exchanges more efficient and are distinct from
the compression applied to regular application data. Finally, it defines the
"ietf-schc-mgmt-basic" YANG module ({{sec-yang-module}}), which provides an RPC
to derive a Rule from an existing one and an RPC to reset the Set of Rules.

# Terminology

This document uses the following terms from {{RFC8724}}:

C/D:
: SCHC Compressor/Decompressor, or SCHC Compression/Decompression.

F/R:
: SCHC Fragmenter/Reassembler, or SCHC Fragmentation/Reassembly.

It also uses the following term, in addition to those defined in
{{RFC8724}}:

Overlapping rule IDs:
: Two different rule IDs overlap when one of them is a prefix of the other
  (e.g. 110 and 110001): a SCHC packet built with the longer rule ID also
  starts with the bits of the shorter one, so the two could not be parsed
  without ambiguity. A new rule ID MUST NOT equal, or overlap, the rule ID
  of an existing `origin`, `valid`, `candidate`, or `invalid` rule.

# Applicability statement

## Architecture

SCHC instance management allows the two endpoints to modify the common SoR, by:

* Adding a new rule.
* Removing an existing rule.
* Triggering Remote Procedural Calls (RPC) within the endpoints.

Because of propagation delays and of the possible loss of management messages,
both endpoints could temporarily hold different versions of a rule with the same
RuleID. Therefore, an existing rule MUST NOT be modified. To change a rule, a
new rule, with another RuleID, MUST be created, and the old one MAY then be
deleted, unless it is an `origin` rule (see {{sec-rule-status}}).

A new type of traffic is defined called management traffic, which deals
exclusively with message exchanges concerning context and rule management.

The rule management uses the CORECONF network management interface
{{I-D.ietf-core-comi}}, which is based on CoAP. In this context, management
traffic refers to the CORECONF messages exchanged between the endpoints to
configure or modify rule sets. The management traffic is transported as
SCHC-compressed packets, tagged with specific Rule IDs. These rules are the
management rules (M in Figure {{Fig-SCHCManagement}}). Management rules can be
either Compression Rules or No-Compression Rules. Only management rules are
permitted to modify the SoR.

The management traffic uses its own IPv6 stack, distinct from regular
application traffic. See Section {{sec-protocols}} for more details.

SCHC Packets using management rules MUST be encrypted either by the underlying
layer (for instance in a QUIC stream dedicated to management inside a QUIC
connection) or directly using OSCORE or DTLS.

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

## CoAP Profile {#sec-coap-profile}

Management requests MUST be protected against packet loss. It is RECOMMENDED to
use CONfirmable requests and no Token. If the management request is too large
regarding the MTU, SCHC Fragmentation SHOULD be used instead of the Block
option. As shown in {{Fig-SCHCManagement}} fragmentation can be common to
Management rules and other rules.


# Rule status {#sec-rule-status}

In {{RFC8724}} and its associated Data Model in {{RFC9363}}, rules are
established when the SCHC instance is configured and are not modified
afterwards. These rules form the initial SoR. With management, rules can be
created and deleted while the instance is running. A new state is therefore
introduced for the rules, the "rule-status". It is used for the creation and the
deletion of rules, to preserve the atomicity of the SoR: both ends move together
from one version of the SoR to the next, and a rule is never used by one end
while the other end does not have it. The status is local to each node: each
node sets the status of its rules according to its own protocol state, so that
both nodes can have different statuses for the same rule during a creation or a
deletion.

A rule can have several statuses (the state machine is summarized in
{{sec-fsm}}):

* `origin`: the rule belongs to the initial SoR, provisioned before the start
  of the SCHC instance, including the management rules. It is used for C/D
  and F/R as a `valid` rule, and behaves as a `valid` rule in the procedures
  below, except that it cannot be modified or deleted: a deletion whose
  target is an `origin` rule is refused with 4.03 Forbidden. It can be
  duplicated (see {{sec-duplicate-rule-rpc}}), except the management rules.
  The `origin` status is only set by provisioning, never by management, and
  the `origin` rules are the only ones kept when the SoR is reset (see
  {{sec-reset-rpc}}). Since they never change, they are always identical on
  both ends.
* `valid`: the rule belongs to the SoR and can be selected for compression or
  fragmentation, and the SCHC PDUs carrying its rule ID are decompressed or
  reassembled with it.
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
  ID MUST NOT be equal to, a prefix of, or prefixed by the rule ID of an
  `origin`, `valid`, `candidate` or `invalid` rule, since the SCHC packets could
  not be parsed without ambiguity.



~~~ aasvg
              A                            B
  X valid     |                            | X valid
              |  delete       /------------|  SCHC PDU (X)
  X invalid   |==============/============>| X invalid
              |<============/==============|---
           ---|            /      Ack      | |
            | |           /                | |
      Guard | |          /                 | |  Guard
            | |X========/                  | |
            v | dropped                    | v
           ---|                            |---
  X free      |                            | X free
              /                            /
              /                            /
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
            | |         X==================|
      Timer | |                            |   
   X valid  * |<---------------------------|   
              |               X            | 
              |                            | 

~~~
{: #Fig-Rule-creation title='Creating a rule'}

### Idempotent creation {#sec-idempotent-creation}

The acknowledgement of a creation may be lost without a SCHC PDU using the new
rule being sent, or the CoAP exchange may be abandoned by A. Then A cannot know
if B has created the rule, and the rule stays `candidate` on A. To recover, A
repeats the creation. If B does not immediately emit a SCHC PDU using the new
rule ID, A has no such indication either, and relies instead on the timer
shown in {{Fig-Rule-creation}} expiring before repeating the creation.

SCHC rule IDs do not have a fixed length. Two different rule IDs overlap when
one of them is a prefix of the other: for instance, the rule IDs 110 and 110001
overlap, since a SCHC packet built with rule 110001 also starts with the bits of
rule 110, and could not be parsed without ambiguity. The rule IDs 110 and 111 do
not overlap. A rule ID that overlaps the rule ID of an existing rule cannot be
used for a new rule (see `free`).

A creation is idempotent:

* if the rule does not exist on B (`free`), B creates it as `valid` and
  acknowledges it;
* if a `valid` rule with the same rule ID and exactly the same content already
  exists on B, B does not change anything and sends the same positive
  acknowledgement as for the first request;
* if a `valid` rule with the same rule ID but a different content exists on B,
  or a rule whose rule ID overlaps (see `free`), B refuses the request with 4.09
  Conflict;
* if the rule is `candidate` on B, the creations of both ends cross (see
  {{sec-simultaneous-creation}});
* if the rule ID is `invalid` on B, or overlaps an `invalid` rule ID (the Guard
  period is running), B also refuses the request with 4.09 Conflict.

When the creation is refused with 4.09 Conflict, A gives up its `candidate`
rule, as described below, and may retry with another rule ID. Discarding the
`candidate` rule without deleting it would not be safe: a copy of the creation
sent in a previous CoAP exchange may still be in flight, and be accepted by B
when its Guard period is over, while A would no longer have the rule. The
deletion makes the rule `invalid` on both ends: a late copy of the creation is
either refused during the Guard period of B, or accepted before the deletion,
which then removes it. An answer is taken into account only if the rule is
still the `candidate` rule created by the request. Otherwise (the rule has been
validated by a SCHC PDU, replaced or deleted in the meantime), it is ignored.
{{Fig-Rule-conflict}} illustrates this with a retransmitted copy of the
creation that arrives late, after A has already deleted its `candidate` rule.

~~~ aasvg
              A                             B
              |- - +                        |
              |    :                        |
 X candidate  |=========create(X)==========>|
              |    :                        |
              |    (a retransmitted copy    |
              |     is still in flight)     |
              |<=======4.09 Conflict========|
 X invalid    |=========delete(X)==========>| X invalid
            | |    :                        | |
      Guard | |    :                        | |
            | |    :   retransmitted        | |
            | |    :     create(X)          | |
            | |    + - - - - - - - - - - - >| |
            | |                             | | refused: X
            | |                             | | still guarded
            v |<============Ack=============|  v
 X free       |                             | X free
~~~
{: #Fig-Rule-conflict title='Late retransmitted creation refused after the candidate rule is deleted'}

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
        (Content-Format: application/yang-instances+cbor-seq)
        ["rule", 9, 4]: { "rule-nature": "nature-compression", ... }

A: rule 9/4 candidate.                    B: rule 9/4 free -> valid

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
rule 9/4 would stay `valid` on B with its original content. A would give up its
`candidate` rule by deleting it, which also deletes rule 9/4 on B, and would
create the rule with the new content with another rule ID.

### Simultaneous creation {#sec-simultaneous-creation}

Both ends can create rules. They could choose the same rule ID at the same
time, or two rule IDs where one overlaps the other. To avoid it, the rule ID
space SHOULD be shared between both ends:

* the rule IDs of the rules created by the Core start with a bit set to 0;
* the rule IDs of the rules created by the Device start with a bit set to 1.

With this convention, two rule IDs created by different ends are never equal,
nor the prefix of each other. It applies to the rules created with an iPATCH and
with the "duplicate-rule" RPC. The `origin` rules, provisioned before the start
of the SCHC instance, are not concerned: their rule IDs can be in both parts,
and are never `free`.

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
reaches the Core before the creation of the Device, the Core gives up its
`candidate` rule and deletes it (see {{sec-idempotent-creation}}). The creation
of the Device is then refused by the Core, or stopped by the deletion when it
reaches the Device: rule X is deleted on both ends, and each end may retry with
another rule ID.

## Rule deletion {#sec-rule-deletion}

When a rule is deleted, it becomes `invalid` on both ends for a Guard period
(see {{Fig-Rule-del}}). During that period, a rule with the same ID cannot be
created, and the SCHC PDUs carrying the Rule ID are dropped. An `origin` rule
cannot be deleted: the deletion is refused with 4.03 Forbidden, and the rule
stays `origin` on both ends.

The initiator marks the rule `invalid` when it sends the deletion, so that it
does not generate any more PDU with this rule. Its Guard period starts when it
receives the acknowledgement. If the deletion or its acknowledgement is lost and
the CoAP exchange is abandoned, the rule stays `invalid` on the initiator
without any running Guard period, and the initiator MUST repeat the deletion
until it receives the acknowledgement. In the meantime, the SCHC PDUs sent by
the other end with this rule ID are dropped.

The deletion is idempotent: if the rule is already `invalid` or `free` on the
receiving end, the request is acknowledged as the first one. It nevertheless
(re)starts the Guard period of the rule, which becomes `invalid` again if it was
`free`, unless the receiving end is itself waiting for the acknowledgement of
its own deletion of this rule. A Guard period that turns out to be useless only
delays the reuse of the rule ID, while a missing one can lead both ends to
disagree on the SoR (see {{sec-guard-computation}}). A deletion received for a
`candidate` rule makes it `invalid`, as for a `valid` rule, and the creation
exchange in progress for this rule is stopped. The other end may have validated
the rule before deleting it, or the deletion may be a late copy of an older one;
in both cases, the rule can be created again after the Guard period.

Management requests MUST be processed only once, as specified in {{RFC7252}},
Section 4.5. That section lets a server redo an idempotent request instead of
just resending the same answer when it sees a duplicate; this does not apply
here, since redoing a creation or a deletion after the rule has since been
deleted and created again would not give the same result.

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

A MUST start its Guard period when it receives the acknowledgement of the
deletion, not when it sends the deletion request: the request may be
retransmitted during MAX_TRANSMIT_SPAN ({{RFC7252}}) before B receives it and
stops using the rule, so starting the Guard period any earlier could let a
stale PDU from B arrive after it has already ended.

{{Fig-Guard-timing}} shows these two arguments on a timeline (not to scale).

~~~ aasvg
---+--------+-----------+-------------------------------------------------
   tA       tB          tA'

            <-------------->
              Guard(B), >= L, starts when B gets delete(X)
                  ^
    A's last PDU(X)

                        <-------------->
                          Guard(A), >= L, starts when A gets the Ack
                           ^
             B's last PDU(X)
~~~
{: #Fig-Guard-timing title='Why a Guard period of L suffices on each end'}

Both ends then use the same Guard period

    G >= L

plus a margin for the clock drift. G does not depend on the number of
retransmissions of the CoAP deletion, since they happen before the Guard period
of A starts.

This document gives no value for L, G and the CoAP transmission parameters: they
depend on the network, which can be a terrestrial LPWAN as well as an
interplanetary link. The CoAP transmission parameters are configured accordingly
({{RFC7252}}, Section 4.8.1), with a MAX_LATENCY of at least L, and the values
derived from them, such as MAX_TRANSMIT_SPAN and EXCHANGE_LIFETIME, are the ones
used in this document.

When both ends delete the same rule at the same time, each end keeps sending its
deletion request until it is acknowledged, possibly after the Guard period of
the other end has started. These requests are not covered by the computation
above. A late copy could reach an end that has already created the rule again,
and make it `invalid` there, while the creation has already reached the other
end. This is why every received deletion restarts the Guard period (see
{{sec-rule-deletion}}): the copies of an earlier exchange are then sent before
the restart, and the copies of the acknowledged exchange are removed by the
deduplication.

Without crossing deletions, the Guard period of A ends at least as late as the
one of B. A creation initiated by A therefore reaches a B whose Guard period is
over. A creation initiated by B may arrive when the Guard period of A is still
running, and be refused with 4.09 Conflict (see {{sec-idempotent-creation}}).
The Guard period of A ends at most MAX_TRANSMIT_SPAN + 2L after the one of B:
the acknowledgement of the deletion may be lost, and the deletion repeated
during MAX_TRANSMIT_SPAN ({{RFC7252}}), while the deduplication prevents these
copies from restarting the Guard period of B. To avoid such a refusal, a rule ID
SHOULD NOT be used for a new rule before MAX_TRANSMIT_SPAN + 2L after the end of
its Guard period: the Guard period of the other end is then over too.

This document recommends not reusing a deleted rule ID immediately: the Guard
period then becomes transparent to management, since no new rule is ever
confused with a stale PDU of the deleted one.

## Restart of a node {#sec-restart}

Since management changes the SoR while the SCHC instance is running, the rules
and their status SHOULD be stored in a stable memory, which survives a restart.
Otherwise, an end that restarts loses the rules created or deleted by
management, while the other end still uses them, and an `invalid` rule would be
considered as `free` before the end of its Guard period. The `origin` rules are
not concerned, since they are part of the initial SoR. An end that restarts
without its rules has only its `origin` rules: it SHOULD then reset the SoR with
the "reset-context" RPC (see {{sec-reset-rpc}}), before any other management
request, so that both ends come back to the same SoR. 

When the rules are kept, after a restart:

* a `candidate` rule stays `candidate`, and the creation is repeated;
* an `invalid` rule whose deletion has not been acknowledged stays `invalid`,
  and the deletion is repeated;
* an `invalid` rule whose Guard period had started begins a new, complete Guard
  period, since the time spent during the restart is unknown. A longer Guard
  period is always safe: a creation received during this period is refused with
  4.09 Conflict, and the other end deletes its `candidate` rule (see
  {{sec-idempotent-creation}}).

The deduplication state of CoAP (see {{sec-rule-deletion}}) is usually lost
during a restart, so that a copy of a management request sent before the restart
could be processed again. If this state is not kept in a stable memory, the end
MUST silently discard the management requests during EXCHANGE_LIFETIME
({{RFC7252}}) after the restart, as if it had not restarted yet, so that the
copies sent before the restart are no longer in flight when it processes
requests again. For the other end, this cannot be distinguished from a loss:
the CoAP exchange is abandoned and the request is repeated later, a rule being
created stays `candidate`, and a rule being deleted stays `invalid`. When OSCORE
is used, its replay window is recovered after a restart as described in
Appendix B.1.2 of {{RFC8613}}.

# Management messages

## Management Protocol Stack {#sec-mgmt-stack}

The management messages are carried by a protocol stack dedicated to management,
independent of the stack used by the application traffic ({{Fig-mgmt-stack}}).
No other traffic uses this stack.

~~~ aasvg
     Device (fe80::1)                     Core/App (fe80::2)
  +-------------------+                 +-------------------+
  |     CORECONF      |                 |     CORECONF      |
  +-------------------+                 +-------------------+
  |   CoAP (inner)    |                 |   CoAP (inner)    |
  |  - - OSCORE - -   |    optional     |  - - OSCORE - -   |
  |   CoAP (outer)    |                 |   CoAP (outer)    |
  +-------------------+                 +-------------------+
  |        UDP        |                 |        UDP        |
  +-------------------+                 +-------------------+
  |       IPv6        |                 |       IPv6        |
  +-------------------+                 +-------------------+
  |       SCHC        |                 |       SCHC        |
  | management rules  |                 | management rules  |
  +-------------------+                 +-------------------+
  |        L2         |<--------------->|        L2         |
  +-------------------+                 +-------------------+
~~~
{: #Fig-mgmt-stack title='Protocol stack of the management messages'}

* IPv6 {{RFC8200}}: the management traffic does not use the same interface as
  the application. Each node of the SCHC instance includes a second IPv6
  stack, used only for management. Only two link-local addresses are allowed on this stack:
  `fe80::1` for the Device and `fe80::2` for the Core, on the application side (see
  {{sec-protocols}}). These IPv6 packets are only exchanged between the two
  nodes of the SCHC instance and are never forwarded outside of it. They are
  compressed with the management rules and, when they are larger than the MTU of
  the underlying network, fragmented with SCHC fragmentation.
* UDP {{RFC0768}}: the CoAP server uses port 5683 and the CoAP client port 3865
  (see {{sec-protocols}}).
* CoAP {{RFC7252}}: both nodes are CoAP clients and servers, so that each node
  can initiate a management operation. The requests are Confirmable, and their
  deduplication is required by the rule status procedures (see
  {{sec-coap-profile}} and {{sec-rule-deletion}}). A node MUST NOT send a
  management request on a rule before it has received the response to its
  previous request on the same rule ID, or on an overlapping rule ID, or
  abandoned this request. The rule status procedures rely on this order.
  Requests on different rules can be sent without waiting for each other, within
  the limit of NSTART ({{RFC7252}}, Section 4.7), the maximum number of requests
  sent to a node and still waiting for a response. Its default value is 1, and
  it can be increased by configuration when the underlying network allows it.
* OSCORE {{RFC8613}}: when the underlying network does not protect the
  management traffic, the CoAP messages MAY be protected with OSCORE between
  both nodes. OSCORE encrypts the inner CoAP message (code, most options and
  payload) and carries it in an outer CoAP message. The outer code is POST for
  the requests and 2.04 Changed for the responses, and the outer message
  contains an OSCORE option. The management rules then compress the outer CoAP
  header. The replay protection of OSCORE complements the CoAP deduplication,
  including after a restart (see {{sec-restart}}).
* CORECONF {{I-D.ietf-core-comi}}: the SoR is a datastore structured by the SCHC
  YANG Data Model ({{I-D.toutain-schc-rfc9363bis}}), whose data nodes are
  identified by SIDs and encoded in CBOR {{RFC9254}}. The rules are read with
  FETCH, iPATCH, and the operations on the SoR are RPCs invoked with POST (see
  {{sec-rpcs}}).


## CORECONF Interface

{{RFC9363}} defines a YANG Data Model for SCHC Rules, updated by
{{I-D.toutain-schc-rfc9363bis}} (module revision 2026-10-27). This document uses
the updated model. SCHC Instance Management MUST use FETCH to read a rule and
iPATCH to create a full rule or delete a rule, and RPCs to generate specific rules depending
on RPC parameters, such as the RPCs of the "`ietf-schc-mgmt-basic`" module
({{sec-yang-module}}).

Management rules have to be marked in a way that allows quickly identifying
which rules in a SoR are responsible for management. Therefore, a
"`nature-management`" Rule nature is defined, and the "management" feature enables
it. A management Rule uses the compression Rule structure, but only the
"`entry-universal`" list supports management Rules: the deprecated "entry" list
does not. Compression needs to be available and activated to do management.

The technical choices of the management protocol stack ({{sec-mgmt-stack}})
allow the management rules to compress the management traffic very efficiently
(see {{sec-protocols}}), since most of the header fields are known in advance:

* the IPv6 addresses are fixed (fe80::1 for the Device and fe80::2 for the
  Core), as are the other fields of the IPv6 header (version, traffic class,
  flow label, next header and hop limit). They are not sent, and the payload
  length is computed;
* the UDP ports are fixed (5683 for the CoAP server and 3865 for the CoAP
  client). They are not sent, and the UDP length and checksum are computed;
* the CoAP requests only use three methods (FETCH, iPATCH and POST), so that
  their code is sent as a short index with a mapping. The type (Confirmable for
  the requests, Acknowledgement for the responses), the Token length (no Token),
  and the Uri-Path and Content-Format options have fixed values and are not
  sent. Only the least significant bits of the Message ID are sent;
* when OSCORE is used, the outer code is always POST for the requests and 2.04
  Changed for the responses, and is not sent either.

A management message is therefore reduced to the Rule ID, a few bits of CoAP
header and the CORECONF payload.

### Rule Status and Guard Period


The "`rule-status`" leaf is managed locally by each node, which rewrites it
according to the protocol state of the rule (see {{sec-rule-status}} and
{{sec-fsm}}). It is never set by the other node: a value received in the
creation of a rule is ignored, and a request whose target is "rule-status" is
refused as any modification of a rule (see {{sec-ipatch}}). In particular, a
node never gets an `origin` rule from the other node. It can be read with FETCH,
for instance to check the state of the rules after a restart.

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
        +--rw rule-id-value     uint32
        +--rw rule-id-length    uint8
        +--rw rule-nature       nature-type
        +--rw rule-status?      status-type {management}?
        +--rw (nature)?
           +--:(fragmentation) {fragmentation}?
           | ...
           +--:(compression-universal) {compression or management}?
              +--rw entry-universal* [entry-index]
              |  ...
~~~
{: #Fig-tree title='YANG Data Model used for CORECONF'}


## Set of Rules Editing
For clarity reasons, this document uses YANG identifiers, written in quotes and
in a fixed-width font, instead of SID values. Each entry in the YANG tree has a
corresponding SID number. Each level of the hierarchy is accessible through one
or several keys. For example, to access the hierarchy under "`rule`",
"`rule-id-value`" and "`rule-id-length`" must be specified.


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

### iPATCH {#sec-ipatch}

iPATCH can only be used to create a complete rule or to delete a rule. It is
forbidden to modify, add or delete an element of an existing rule (an entry, a
Target Value, a Matching Operator, etc.), since a rule cannot be changed without
changing its rule ID (see {{sec-rule-status}}). The creation and the deletion
follow the rule status procedures ({{sec-rule-status}}).

#### Creating a rule

The target of the iPATCH request is the rule, identified by its keys
"`rule-id-value`" and "`rule-id-length`", and the value is the complete rule:
its nature and all its entries. The model requires that the indices of the lists
of the rule ("`entry-index`" and "`index`") be strictly incremental. CORECONF
does not enforce it; it relies on the node that builds the rule. If the rule
already exists, the request follows the idempotent creation (see
{{sec-idempotent-creation}}). The "`rule-status`" of the rule is not part of the
request: if it is present, it is ignored, since each node sets the status of its
rules according to its own protocol state.

Example: creation of Rule 9/4, whose first entry is the IPv6 Version.

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-instances+cbor-seq)
  {
    ["rule", 9, 4] : {
        "rule-nature" : "nature-compression",
        "entry-universal" : [
          { "entry-index"         : 0,
            "field-id"            : "fid-ipv6-version",
            "field-length"        : 4,
            "field-position"      : 1,
            "direction-indicator" : "di-bidirectional",
            "target-value"        : [{"index" : 0, "value" : h"06"}],
            "matching-operator"   : "mo-equal",
            "comp-decomp-action"  : "cda-not-sent" },
          ...
        ]
    }
  }

  RES: 2.04 Changed
~~~

#### Deleting a rule

The target of the iPATCH request is the rule, and the value is null.

Example: deletion of Rule 7/8.

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-instances+cbor-seq)
  {
    ["rule", 7, 8]: null
  }

  RES: 2.04 Changed
~~~

An `origin` rule, such as a management rule, cannot be deleted: a deletion whose
target is an `origin` rule is refused with 4.03 Forbidden.

Deleting the top-level object (`ietf-schc:schc`) is refused with 4.03 Forbidden.
To return to the initial SoR, a node uses the "reset-context" RPC (see
{{sec-reset-rpc}}), which keeps the `origin` rules, so that the management of
the SCHC instance remains possible.

#### Forbidden requests

A node receiving an iPATCH request whose target is an element of a rule, such as
an entry, a leaf or a list element, MUST refuse it with 4.03 Forbidden, whether
this element exists or not.

Examples:

- Change of the Matching Operator and the CDA of the entry 0 of Rule 6/3:

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-instances+cbor-seq)
  {
    ["entry-universal", 6, 3, 0] : {
        {"delta_matching-operator": "mo-ignore"},
        {"delta_comp-decomp-action": "cda-value-sent"}
    }
  }

  RES: 4.03 Forbidden
~~~

- Addition of a Target Value to the entry 3 of Rule 0/3:

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-instances+cbor-seq)
  {
    ["target-value", 0, 3, 3] : [
        {delta_index : 0, delta_value : h"50"}
    ]
  }

  RES: 4.03 Forbidden
~~~

- Deletion of the "`rule-id-value`" key of Rule 0/3:

~~~
  REQ: iPATCH /c
       (Content-Format: application/yang-instances+cbor-seq)
  {
    ["rule-id-value", 0, 3]: null
  }

  RES: 4.03 Forbidden
~~~

### RPC invocation (POST) {#sec-post-method}

As described in {{I-D.ietf-core-comi}}, the POST CoAP method is used to trigger
Remote Procedure Calls (RPC) and other actions within a SCHC endpoint. Thus, a
POST message can be sent to invoke a specific RPC on the remote endpoint.
Details of the supported RPCs and their behavior are defined in Section
{{sec-rpcs}}.

This document defines the "`duplicate-rule`" and "`reset-context`" RPCs, in the
"`ietf-schc-mgmt-basic`" YANG module ({{sec-yang-module}}). Other RPCs, adapted to
the traffic in order to optimize the rule management for its patterns, may be
defined by other documents, in other YANG modules.

The input parameters of an RPC acting on some rules MUST provide the rule IDs of
the rules on which the RPC is applied, so that both nodes know the rules
concerned by the rule status procedures ({{sec-rule-status}}). An RPC acting on
all the rules that are not `origin`, such as "reset-context", has no rule ID in
its input. The output of the RPC MUST provide a status summarizing the execution
of the RPC.

RPCs and actions are defined in a YANG Data Model with optional associated
inputs and outputs. The request payload contains the RPC input map, if any. The
response payload contains the corresponding output map, if any.

## Management Errors

There are different levels of error detection:

* CORECONF Errors: these errors are directly generated at the CORECONF-managed
  context level. For instance, retrieving a value with a wrong key.
* YANG validation errors: the data model does not conform with constraints such
  as "must" or "mandatory". This check is optional, since it may require a lot
  of resources on a device.
* SCHC errors: errors on the Data Model that cannot be detected at the YANG
  level. For example, overlapping rule IDs.

# RPCs {#sec-rpcs}

A YANG "RPC" is an operation that may be invoked within a SCHC endpoint and
triggers a specified behavior. Within the context of rule management, RPCs are
used to perform actions on the set of rules and may also be used for other
operations, such as rebooting the remote device endpoint.

Each RPC resource has specific inputs and outputs, and may be invoked remotely
via a POST CoAP message, as described in Section {{sec-post-method}}.



## Duplicate Rule {#sec-duplicate-rule-rpc}

To add a new rule, instead of using the iPATCH method with a full rule
definition (especially when the new rule is similar to an existing one), the
RECOMMENDED approach is to use the `duplicate-rule` RPC. This operation copies
an existing rule ("from") into a new rule ("to"), and can optionally include an
iPATCH sequence specifying modifications to apply to the duplicated rule. The
output returns a status, an enumeration conveying the result of the operation:
"success-with-validation" (0), "success-not-validated" (1),
"to-rule-already-exists" (-1), "from-rule-not-found" (-2) or "invalid-ipatch"
(-3).

The "from" rule can be an `origin` rule, except a management rule: this is the
usual way to specialize a generic rule of the initial SoR. The new rule is
created as any rule created by management (see {{sec-rule-status}}), and is
never an `origin` rule.

Represented as a tree:

~~~
module: ietf-schc-mgmt-basic

  rpcs:
    +---x duplicate-rule {schc:management}?
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
  real request. In the `ipatch-sequence`, 14 is the `entry-index` of the
  target entry in the new Rule, and 0 is the `index` of its target value.

~~~
REQ:  POST </c>
      (Content-Format: application/yang-instances+cbor-seq)

{
  "/ietf-schc-mgmt-basic:duplicate-rule":
  {
    "input/from/rule-id-value": 8,
    "input/from/rule-id-length": 4,
    "input/to/rule-id-value": 9,
    "input/to/rule-id-length": 4,
    "input/ipatch-sequence":
      [
        "/ietf-schc:schc/rule/entry-universal/target-value/value", 14, 0,
      ]: b"\x14\x00"
  }
}

RES:  2.04 Changed
      (Content-Format: application/yang-instances+cbor-seq)

{
  "/ietf-schc-mgmt-basic:duplicate-rule":
  {
    "output/status": "success-with-validation",
  }
}
~~~

### Error Handling

The `duplicate-rule` operation SHALL be atomic. If an error occurs during either
stage of the process (rule duplication or subsequent modification through the
iPATCH sequence) the SCHC endpoint MUST revert any partial changes to restore
the previous state. The RPC output MUST indicate the failure with one of the
negative status values ("to-rule-already-exists", "from-rule-not-found" or
"invalid-ipatch"), to signal that the duplication did not take place as
requested. The precise error code and diagnostic message are
implementation-dependent but SHOULD provide enough context for the management
entity to identify the cause of failure.

## Reset Context {#sec-reset-rpc}

The "reset-context" RPC brings the SoR of both nodes back to its initial
content: only the `origin` rules are kept, and all the rules created by
management are removed. It has no input, and its output is a status.

Represented as a tree:

~~~
module: ietf-schc-mgmt-basic

  rpcs:
    +---x reset-context {schc:management}?
       +--ro output
          +--ro status?   enumeration
~~~

The reset applies to every rule that is not `origin`, whatever its status, and
to every rule ID that is not the rule ID of an `origin` rule, even if it is
absent from the SoR. It follows the deletion procedure ({{sec-rule-deletion}}):

* the node that invokes the RPC applies the reset when it sends the request:
  these rules and rule IDs become `invalid`, its creations and deletions in
  progress are stopped, and its Guard period starts when it receives the
  response;
* the node that receives the request applies the reset at the reception: these
  rules and rule IDs become `invalid`, its creations in progress are stopped,
  its Guard period starts, and it answers 2.04 Changed with the "success"
  status. If it is itself waiting for the acknowledgement of its own deletion or
  reset, its Guard period starts when it receives this acknowledgement;
* during the Guard period, a creation is refused with 4.09 Conflict (see
  {{sec-idempotent-creation}}), and the SCHC PDUs carrying a rule ID that is not
  the rule ID of an `origin` rule are dropped. When the Guard period is over,
  these rule IDs become `free`: the SoR only contains the `origin` rules;
* the `origin` rules are not affected and remain usable during the whole
  procedure, since they are identical on both nodes. In particular, the
  management traffic, compressed with the management rules, is not
  interrupted.

A reset concerns all the rule IDs: a node does not send it while another
management request of this node is in progress, and does not send another
management request before the reset is acknowledged. As a deletion, the reset is
idempotent: the node repeats the request until it receives the response, a copy
of the request is removed by the deduplication ({{sec-rule-deletion}}), and two
crossing resets lead to the same SoR.

Applying the reset on the invoking node only when it receives the response would
not be safe. The other node, which has already applied the reset, could create a
rule whose creation reaches the invoking node before the response: the invoking
node would accept it, then remove it when the response arrives, while the rule
stays `valid` on the other node. Applying the reset when the request is sent
makes the invoking node refuse such a creation with 4.09 Conflict, and the other
node then deletes its `candidate` rule (see {{sec-idempotent-creation}}).

Example:

~~~
REQ:  POST </c>
      (Content-Format: application/yang-instances+cbor-seq)

{
  "/ietf-schc-mgmt-basic:reset-context": null
}

RES:  2.04 Changed
      (Content-Format: application/yang-instances+cbor-seq)

{
  "/ietf-schc-mgmt-basic:reset-context":
  {
    "output/status": "success"
  }
}
~~~

A node that is not allowed to reset the SoR of the other node refuses the
request with 4.03 Forbidden.

# Protocol Stack {#sec-protocols}

The management inside the instance has its own IPv6 stack, independent of the
application traffic. IPv6/UDP/CoAP is used to allow the implementation of the
CORECONF interface. No other kind of traffic is allowed.

The end-point acting as a Device has the IPv6 address fe80::1/64 and the other
end, the Core, is assigned the address fe80::2/64.

Both endpoints implement CoAP client and server capabilities, that is, both
endpoints are capable of sending requests and processing responses. The server
uses port 5683 and the client 3865.

## Management Compression Rules {#sec-mgmt-rules}

To enable CORECONF-based context and rule management over SCHC, a set of
dedicated management rules is defined. These rules are used exclusively for
management traffic, that is, packets exchanged between SCHC endpoints for the
purpose of managing rules, not for application data transfer.

This specification introduces four rules, allowing bidirectional operation and
fine control over management capabilities. Each rule defines the compression
behavior for management messages in its direction, distinguishing between
requests and responses.

* M1: the requests sent by the Device (Uplink), and their responses carrying a
  payload (Downlink).
* M2: the responses without payload to the requests sent by the Device
  (Downlink).
* M3: mirrors M1 for the requests sent by the Core (Downlink), and their
  responses carrying a payload (Uplink).
* M4: mirrors M2 for the responses without payload to the requests sent by the
  Core (Uplink).

Implementations MAY choose to support only a subset of these rules, depending on
their operational or security requirements. For instance, an implementation may
include only M3 and M4 to permit management operations exclusively from one
endpoint, effectively preventing unsolicited management requests in the other
direction. In this sense, the absence of some management rules in the SoR
implicitly acts as a policy mechanism or safeguard for rule management
operations.

Management rules are protected elements within the SoR. They define the
operation of the management channel itself: they are `origin` rules (see
{{sec-rule-status}}), which cannot be modified or deleted, and in addition they
MUST NOT be duplicated. Any attempt to apply a modification, duplication or
deletion request to a management rule MUST result in a 4.03 Forbidden error
response. This restriction ensures the integrity and stability of the SCHC
management process, which is not interrupted by a reset of the SoR
({{sec-reset-rpc}}).

The four management rules are given in {{sec-mgmt-rules-tables}}.

# OSCORE

## Compression Rules

# DTLS

## Compression Rules

# YANG Module {#sec-yang-module}

The "ietf-schc" module, including the "management" feature, the "rule-status"
leaf and the "context" container, is defined in {{I-D.toutain-schc-rfc9363bis}}
(revision 2026-10-27). This document defines the "ietf-schc-mgmt-basic" module,
which imports it and defines the "duplicate-rule" RPC
({{sec-duplicate-rule-rpc}}) and the "reset-context" RPC ({{sec-reset-rpc}}).
The RPCs are only available when the "management" feature of "ietf-schc" is
supported.

The tree of the module, following the notation of {{RFC8340}}, is:

~~~
module: ietf-schc-mgmt-basic

  rpcs:
    +---x duplicate-rule {schc:management}?
    |  +---w input
    |  |  +---w from
    |  |  |  +---w rule-id-value?    uint32
    |  |  |  +---w rule-id-length?   uint8
    |  |  +---w to
    |  |  |  +---w rule-id-value?    uint32
    |  |  |  +---w rule-id-length?   uint8
    |  |  +---w ipatch-sequence?   binary
    |  +--ro output
    |     +--ro status?   enumeration
    +---x reset-context {schc:management}?
       +--ro output
          +--ro status?   enumeration
~~~
{: #Fig-tree-rpc title='Tree of the ietf-schc-mgmt-basic YANG module'}

{::comment}
TODO: allocate a SID range and a SID file for this module, needed by CORECONF.
{:/comment}

~~~ yang
module ietf-schc-mgmt-basic {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-schc-mgmt-basic";
  prefix schc-bmgmt;

  import ietf-schc {
    prefix schc;
    reference
      "draft-toutain-schc-rfc9363bis A YANG Data Model for Static
       Context Header Compression (SCHC)";
  }

  organization
    "IETF Static Context Header Compression (schc) Working Group";
  contact
    "WG Web:   <https://datatracker.ietf.org/wg/schc/about/>
     WG List:  <mailto:schc@ietf.org>
     Editor:   Laurent Toutain
       <mailto:laurent.toutain@imt-atlantique.fr>";
  description
    "Copyright (c) 2026 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject to
     the license terms contained in, the Revised BSD License set
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

     This module defines the basic management operations of the
     SCHC Rules with CORECONF: the 'duplicate-rule' and
     'reset-context' RPCs.";

  revision 2026-10-29 {
    description
      "Initial version, with the 'duplicate-rule' RPC moved from the
       'ietf-schc' module, and the 'reset-context' RPC.";
    reference
      "draft-toutain-schc-coreconf-management CORECONF Rule
       management for SCHC";
  }

  rpc duplicate-rule {
    if-feature "schc:management";
    description
      "Duplicate an existing Rule, given by 'from', into a new Rule,
       given by 'to', which does not exist yet. The new Rule may be
       updated with the CORECONF iPATCH payload carried in
       'ipatch-sequence' before it is added to the Set of Rules.
       The 'from' Rule can be an origin Rule, except a management
       Rule. The new Rule is created as any Rule created by
       management, and is never an origin Rule.";
    input {
      container from {
        description
          "Rule ID of the existing Rule to duplicate.";
        uses schc:rule-id-type;
      }
      container to {
        description
          "Rule ID of the new Rule.";
        uses schc:rule-id-type;
      }
      leaf ipatch-sequence {
        type binary;
        description
          "CBOR sequence of a CORECONF iPATCH payload, applied to the
           new Rule before it is added. This parameter is optional;
           when it is absent, the new Rule is an exact copy.";
      }
    }
    output {
      leaf status {
        type enumeration {
          enum success-with-validation {
            value 0;
            description
              "The new Rule has been created and validated.";
          }
          enum success-not-validated {
            value 1;
            description
              "The new Rule has been created, but not validated
               yet.";
          }
          enum to-rule-already-exists {
            value -1;
            description
              "The Rule ID given by 'to' already exists.";
          }
          enum from-rule-not-found {
            value -2;
            description
              "The Rule ID given by 'from' does not exist.";
          }
          enum invalid-ipatch {
            value -3;
            description
              "The 'ipatch-sequence' cannot be applied to the new
               Rule.";
          }
        }
        description
          "Result of the duplication. A negative value indicates that
           the duplication did not take place.";
      }
    }
  }

  rpc reset-context {
    if-feature "schc:management";
    description
      "Reset the Set of Rules to its initial content: only the
       origin Rules are kept. Every other Rule, whatever its status,
       becomes invalid, and its RuleID becomes free when the guard
       period is over. The node that invokes the RPC applies the
       reset when it sends the request, the node that receives it
       applies the reset when it receives the request.";
    output {
      leaf status {
        type enumeration {
          enum success {
            value 0;
            description
              "The reset has been applied.";
          }
        }
        description
          "Result of the reset. A reset that cannot be applied is
           refused with a CoAP error code.";
      }
    }
  }
}
~~~
{: #Fig-yang-mgmt title="ietf-schc-mgmt-basic YANG Module" sourcecode-name="ietf-schc-mgmt-basic@2026-10-29.yang" sourcecode-markers="true"}

# Security Considerations {#sec-security}

The management of the SoR is restricted to the management rules, and the SCHC
Packets using them MUST be protected by the underlying layer, OSCORE, or DTLS
({{sec-mgmt-stack}}). The `origin` rules, including the management rules,
cannot be modified or deleted, and the management rules cannot be duplicated
({{sec-rule-status}}, {{sec-mgmt-rules}}). The `origin` status is only set by
provisioning: a node cannot make a rule permanent on the other node.

The "ietf-schc-mgmt-basic" YANG module defines two RPCs that change the SoR:

* "duplicate-rule" creates a Rule. Creating a Rule changes the way the packets
  are compressed, decompressed, fragmented, and reassembled, and an
  unauthorized invocation could disturb or redirect the traffic of the Device;
* "reset-context" removes all the Rules created by management. An unauthorized
  or repeated invocation would prevent the use of these Rules, and degrade the
  compression to the `origin` rules, but cannot remove the management rules.

These RPCs are only invoked through the management rules, and their access
SHOULD be restricted to the authorized management nodes, for instance using the
NETCONF Access Control Model {{?RFC8341}} or an equivalent mechanism.

# IANA Considerations {#sec-iana}

## URI Registration

IANA is requested to register the following URI in the "ns" subregistry of the
"IETF XML Registry" {{RFC3688}}:

URI:
: urn:ietf:params:xml:ns:yang:ietf-schc-mgmt-basic

Registrant Contact:
: The IESG.

XML:
: N/A; the requested URI is an XML namespace.

## YANG Module Name Registration

IANA is requested to register the following YANG module in the "YANG Module
Names" registry {{RFC6020}}:

Name:
: ietf-schc-mgmt-basic

Namespace:
: urn:ietf:params:xml:ns:yang:ietf-schc-mgmt-basic

Prefix:
: schc-bmgmt

Reference:
: RFC XXXX

--- back


# Management Rules {#sec-mgmt-rules-tables}

This appendix gives the four management rules described in {{sec-mgmt-rules}}.
They follow the management protocol stack ({{sec-mgmt-stack}}):

* the Device has the IPv6 address fe80::1 and the Core the address fe80::2;
* the CoAP server uses the UDP port 5683 and the CoAP client the port 3865. In
  the exchanges initiated by the Device (M1 and M2), its port is 3865; in those
  initiated by the Core (M3 and M4), the port of the Device is 5683;
* the requests are Confirmable and have no Token. They use the methods FETCH,
  iPATCH and POST on the datastore resource "c". The responses are piggybacked
  in the Acknowledgement, and the error responses have no payload;
* OSCORE is not used.

In the tables, "ids" stands for the Content-Format
application/yang-identifiers+cbor-seq (FETCH requests), and "inst" for
application/yang-instances+cbor-seq (iPATCH and POST requests, and responses
carrying a payload). The CoAP codes are given as numbers: 2 (POST), 5 (FETCH), 7
(iPATCH), 68 (2.04 Changed), 69 (2.05 Content), 128 (4.00 Bad Request), 129
(4.01 Unauthorized), 131 (4.03 Forbidden), 132 (4.04 Not Found), 137 (4.09
Conflict) and 160 (5.00 Internal Server Error).

~~~
 +---------------------------------------------------------------------+
 |RuleID M1                                                            |
 +-------------------+--+--+--+-------------+-------------+------------+
 |        FID        |FL|FP|DI|     TV      |     MO      |    CDA     |
 +-------------------+--+--+--+-------------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6            |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1            |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470       |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|             |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17           |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64           |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::1          |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::2          |equal        |not-sent    |
 +===================+==+==+==+=============+=============+============+
 |UDP DevPort        |16|1 |Bi|3865         |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|5683         |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|             |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|             |ignore       |compute-*   |
 +===================+==+==+==+=============+=============+============+
 |CoAP Version       |2 |1 |Bi|1            |equal        |not-sent    |
 |CoAP Type          |2 |1 |Up|0            |equal        |not-sent    |
 |CoAP Type          |2 |1 |Dw|2            |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0            |equal        |not-sent    |
 |CoAP Code          |8 |1 |Up|[2, 5, 7]    |match-mapping|mapping-sent|
 |CoAP Code          |8 |1 |Dw|[68, 69]     |match-mapping|mapping-sent|
 |CoAP MID           |16|1 |Bi|0            |MSB(9)       |LSB         |
 |CoAP Uri-Path      |8 |1 |Up|c            |equal        |not-sent    |
 |CoAP Content-Format|8 |1 |Up|[ids, inst]  |match-mapping|mapping-sent|
 |CoAP Content-Format|8 |1 |Dw|inst         |equal        |not-sent    |
 +===================+==+==+==+=============+=============+============+
~~~
{: #Fig-M1 title='Management Rule 1'}

~~~
 +---------------------------------------------------------------------+
 |RuleID M2                                                            |
 +-------------------+--+--+--+-------------+-------------+------------+
 |        FID        |FL|FP|DI|     TV      |     MO      |    CDA     |
 +-------------------+--+--+--+-------------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6            |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1            |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470       |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|             |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17           |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64           |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::1          |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::2          |equal        |not-sent    |
 +===================+==+==+==+=============+=============+============+
 |UDP DevPort        |16|1 |Bi|3865         |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|5683         |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|             |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|             |ignore       |compute-*   |
 +===================+==+==+==+=============+=============+============+
 |CoAP Version       |2 |1 |Bi|1            |equal        |not-sent    |
 |CoAP Type          |2 |1 |Dw|2            |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0            |equal        |not-sent    |
 |CoAP Code          |8 |1 |Dw|[68, 128,    |match-mapping|mapping-sent|
 |                   |  |  |  | 129, 131,   |             |            |
 |                   |  |  |  | 132, 137,   |             |            |
 |                   |  |  |  | 160]        |             |            |
 |CoAP MID           |16|1 |Bi|0            |MSB(9)       |LSB         |
 +===================+==+==+==+=============+=============+============+
~~~
{: #Fig-M2 title='Management Rule 2'}

~~~
 +---------------------------------------------------------------------+
 |RuleID M3                                                            |
 +-------------------+--+--+--+-------------+-------------+------------+
 |        FID        |FL|FP|DI|     TV      |     MO      |    CDA     |
 +-------------------+--+--+--+-------------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6            |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1            |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470       |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|             |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17           |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64           |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::1          |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::2          |equal        |not-sent    |
 +===================+==+==+==+=============+=============+============+
 |UDP DevPort        |16|1 |Bi|5683         |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|3865         |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|             |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|             |ignore       |compute-*   |
 +===================+==+==+==+=============+=============+============+
 |CoAP Version       |2 |1 |Bi|1            |equal        |not-sent    |
 |CoAP Type          |2 |1 |Dw|0            |equal        |not-sent    |
 |CoAP Type          |2 |1 |Up|2            |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0            |equal        |not-sent    |
 |CoAP Code          |8 |1 |Dw|[2, 5, 7]    |match-mapping|mapping-sent|
 |CoAP Code          |8 |1 |Up|[68, 69]     |match-mapping|mapping-sent|
 |CoAP MID           |16|1 |Bi|0            |MSB(9)       |LSB         |
 |CoAP Uri-Path      |8 |1 |Dw|c            |equal        |not-sent    |
 |CoAP Content-Format|8 |1 |Dw|[ids, inst]  |match-mapping|mapping-sent|
 |CoAP Content-Format|8 |1 |Up|inst         |equal        |not-sent    |
 +===================+==+==+==+=============+=============+============+
~~~
{: #Fig-M3 title='Management Rule 3'}

~~~
 +---------------------------------------------------------------------+
 |RuleID M4                                                            |
 +-------------------+--+--+--+-------------+-------------+------------+
 |        FID        |FL|FP|DI|     TV      |     MO      |    CDA     |
 +-------------------+--+--+--+-------------+-------------+------------+
 |IPv6 Version       |4 |1 |Bi|6            |equal        |not-sent    |
 |IPv6 Traffic Class |8 |1 |Bi|1            |equal        |not-sent    |
 |IPv6 Flow Label    |20|1 |Bi|144470       |equal        |not-sent    |
 |IPv6 Length        |16|1 |Bi|             |ignore       |compute-*   |
 |IPv6 Next Header   |8 |1 |Bi|17           |equal        |not-sent    |
 |IPv6 Hop Limit     |8 |1 |Bi|64           |equal        |not-sent    |
 |IPv6 DevPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 DevIID        |64|1 |Bi|::1          |equal        |not-sent    |
 |IPv6 AppPrefix     |64|1 |Bi|fe80::/64    |equal        |not-sent    |
 |IPv6 AppIID        |64|1 |Bi|::2          |equal        |not-sent    |
 +===================+==+==+==+=============+=============+============+
 |UDP DevPort        |16|1 |Bi|5683         |equal        |not-sent    |
 |UDP AppPort        |16|1 |Bi|3865         |equal        |not-sent    |
 |UDP Length         |16|1 |Bi|             |ignore       |compute-*   |
 |UDP Checksum       |16|1 |Bi|             |ignore       |compute-*   |
 +===================+==+==+==+=============+=============+============+
 |CoAP Version       |2 |1 |Bi|1            |equal        |not-sent    |
 |CoAP Type          |2 |1 |Up|2            |equal        |not-sent    |
 |CoAP TKL           |4 |1 |Bi|0            |equal        |not-sent    |
 |CoAP Code          |8 |1 |Up|[68, 128,    |match-mapping|mapping-sent|
 |                   |  |  |  | 129, 131,   |             |            |
 |                   |  |  |  | 132, 137,   |             |            |
 |                   |  |  |  | 160]        |             |            |
 |CoAP MID           |16|1 |Bi|0            |MSB(9)       |LSB         |
 +===================+==+==+==+=============+=============+============+
~~~
{: #Fig-M4 title='Management Rule 4'}

# Rule Status State Machine {#sec-fsm}

{{Fig-fsm}} summarizes the statuses of a rule on one node, and the transitions
described in {{sec-rule-status}}. The `invalid` status has two phases: the Guard
period is either waiting for the acknowledgement of the deletion sent by this
node, or running. The `origin` status is not shown: an `origin` rule is set by
provisioning and never changes status.

~~~ aasvg
              +-------------------- T2 -----------------+
              |                                         v
          +--------+    T1    +-----------+   T4    +--------+
          |        |--------->|           |-------->|        |
          |  free  |          | candidate |         | valid  |
          |        |          |           |         |        |
          +--------+          +-----------+         +--------+
            |   ^               |    ^  |               |
         T7 |   | T8         T5 |    +--+ T3            | T6
            v   |               v                       v
        +-----------------------------------------------------+
        | guard waiting for ack                               |
        |                      invalid                        |
        | guard running                                       |
        +-----------------------------------------------------+
                              |   ^
                              +---+ T9
~~~
{: #Fig-fsm title='Rule status state machine on one node'}

* T1, `free` to `candidate`: the node decides to create a rule, with a rule ID in
  its part of the rule ID space (see {{sec-simultaneous-creation}}) that does
  not overlap an existing rule ID. A rule ID whose Guard period has just ended
  SHOULD NOT be chosen (see {{sec-guard-computation}}). It sends the creation.
* T2, `free` to `valid`: a creation is received. The node answers 2.04 Changed.
* T3, `candidate` to `candidate`: the CoAP exchange is abandoned, in particular
  when the other end restarts (see {{sec-restart}}), or the node restarts. The
  creation is repeated later, with an increasing delay. A Device that receives a
  creation for this rule ID answers 4.09 Conflict.
* T4, `candidate` to `valid`: 2.04 Changed is received for the creation, a SCHC
  PDU carrying the rule ID is received and decoded, or, on the Core only, a
  creation of the same rule ID is received from the Device. In the last case,
  the received rule replaces the `candidate` one, and the Core answers 2.04
  Changed. In all cases, the creation stops.
* T5, `candidate` to `invalid`: a deletion is received (the Guard period starts,
  the creation stops, the node answers 2.04 Changed), or the node gives up the
  rule, in particular when 4.09 Conflict is received for the creation, and
  sends a deletion (the Guard period waits for the acknowledgement).
* T6, `valid` to `invalid`: the node sends a deletion (the Guard period waits for
  the acknowledgement), or receives one (the Guard period starts, the node
  answers 2.04 Changed).
* T7, `free` to `invalid`: a deletion is received for a free rule ID. The Guard
  period starts, and the node answers 2.04 Changed.
* T8, `invalid` to `free`: the Guard period is over. The rule is marked `free`
  or removed from the SoR.
* T9, `invalid` to `invalid`:
  * 2.04 Changed is received for the deletion sent by the node: the Guard period
    starts;
  * the CoAP exchange of the deletion is abandoned, or the node restarts while
    waiting for the acknowledgement: the deletion is repeated;
  * a deletion is received: the node answers 2.04 Changed, and the Guard period
    restarts, unless it is waiting for the acknowledgement of the deletion sent
    by the node;
  * a creation is received: the node answers 4.09 Conflict;
  * the node restarts while the Guard period is running: a new, complete Guard
    period starts.
* T10, any status except `origin` to `invalid` (not shown in the figure): the
  node sends a reset (the Guard period waits for the acknowledgement), or
  receives one (the Guard period starts, the node answers 2.04 Changed), see
  {{sec-reset-rpc}}. This also applies to a rule ID absent from the SoR.

The following events do not change the status:

* `origin`: a SCHC PDU carrying the rule ID is decoded with the rule. A deletion
  is answered 4.03 Forbidden. A creation is answered as for a `valid` rule. A
  reset does not change it.
* `valid`: a SCHC PDU carrying the rule ID is decoded with the rule. A creation
  with the same content is answered 2.04 Changed, a creation with a different
  content or an overlapping rule ID is answered 4.09 Conflict.
* `candidate`: no SCHC PDU is generated with the rule. An answer that does not
  correspond to the creation of the current `candidate` rule is ignored.
* `free` and `invalid`: a SCHC PDU carrying the rule ID is dropped.

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
* "YANG Module": the module is moved from the appendix to the body of the
  document, after the RPCs section. The RPCs section no longer says that the
  definition of the RPCs is out of scope: this document defines
  "duplicate-rule", other documents may define other RPCs. The abstract, the
  introduction and "YANG Data Model" mention the module.
* "Restart of a node": the Echo option is removed; without a stable
  deduplication state, the requests are answered with 5.03 during
  EXCHANGE_LIFETIME.
* "Applicability statement": to change a rule, a new rule with another RuleID
  is created, and the old one may then be deleted (instead of deleting the
  rule and creating a new one).
* New "Security Considerations" and "IANA Considerations" sections: protection
  of the management traffic and of the "duplicate-rule" RPC, registration of
  the namespace URI and of the module name. References to RFC 2119, RFC 8174,
  RFC 3688, RFC 6020, RFC 7950 and RFC 8340 are added.
* "Idempotent creation": definition of overlapping rule IDs, which do not have a
  fixed length: one is a prefix of the other, as 110 and 110001.
* "Rule status": a `valid` rule can be selected for compression or
  fragmentation, and the SCHC PDUs carrying its rule ID are decompressed or
  reassembled with it.
* New `origin` status ({{sec-rule-status}}): the rules of the initial SoR,
  including the management rules, are `origin` rules. They are used as `valid`
  rules, cannot be modified or deleted (4.03 Forbidden), can be duplicated
  except the management rules, and are the only rules kept by a reset. The
  status is only set by provisioning, and is the default of "rule-status"
  ("status-origin", derived from "status-valid"). "Rule Status and Guard
  Period", "Simultaneous creation", "Rule deletion", "Restart of a node" and the
  state machine ({{sec-fsm}}) are updated accordingly.
* New "reset-context" RPC ({{sec-reset-rpc}}): it replaces the factory reset by
  the deletion of the top-level object, which is now refused with 4.03
  Forbidden. The invoking node applies the reset when it sends the request, the
  other node when it receives it; all the rule IDs that are not `origin` become
  `invalid` for a Guard period. Applying it only at the reception of the
  response leads both nodes to disagree on the SoR. An end that restarts without
  its rules should reset the SoR.
* The YANG module is named "ietf-schc-mgmt-basic" (prefix "schc-bmgmt"), for
  the basic management operations, and contains both RPCs; IANA and Security
  Considerations are updated.
* "Idempotent creation": a creation received during the Guard period is
  refused with 4.09 Conflict again, instead of 5.03 Service Unavailable. On
  4.09 Conflict, whatever its cause, the creator gives up its `candidate` rule
  by deleting it, and may retry with another rule ID: the deletion makes the
  rule `invalid` on both ends, so that a late copy of the creation cannot
  create it again. "Guard period computation": the Guard periods of both ends
  do not end at the same time, and a rule ID SHOULD NOT be used for a new rule
  before MAX_TRANSMIT_SPAN + 2L after the end of its Guard period.
  "Simultaneous creation", "Restart of a node", "reset-context" and the state
  machine ({{sec-fsm}}) are updated: the transition from `candidate` to `free`
  is removed, and the transitions are renumbered.
* "Restart of a node": without a stable deduplication state, the management
  requests are silently discarded during EXCHANGE_LIFETIME after the restart,
  instead of being answered with 5.03 Service Unavailable; the other end repeats
  them as after a loss. 5.03 Service Unavailable is no longer used, and is
  removed from the codes of the management rules M2 and M4.
* "Guard period computation": the numerical examples (G of 30 s, CoAP
  transmission span of 45 s) are removed. L, G and the CoAP transmission
  parameters depend on the network, possibly an interplanetary link; the CoAP
  transmission parameters are configured accordingly, with a MAX_LATENCY of at
  least L.
* The refusal of a request on a management rule is a 4.03 Forbidden response
  instead of an Unauthorized one.
* "YANG Module" appendix (was "YANG DM"): the "duplicate-rule" RPC is no longer
  in the "ietf-schc" module of {{I-D.toutain-schc-rfc9363bis}}; it is defined in
  the new "ietf-schc-mgmt-basic" module of this document, which imports
  "ietf-schc" and makes the RPC depend on its "management" feature. The
  descriptions of the RPC and of its status values are completed. The CORECONF
  examples use the "/ietf-schc-mgmt-basic:duplicate-rule" path.
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
* New appendix "Rule Status State Machine" ({{sec-fsm}}), summarizing the
  statuses and transitions of a rule.
* "Rule deletion": every received deletion (re)starts the Guard period, even if
  the rule is already `invalid` or `free`, to cover crossing deletions.
* "Management Protocol Stack" ({{sec-mgmt-stack}}) added: IPv6, UDP, CoAP with
  optional OSCORE, and CORECONF. {{RFC0768}}, {{RFC8200}} and {{RFC8613}} are
  normative references.
* "M Rules" replaced by "management rules" in the text and in
  {{Fig-mgmt-stack}}; the four management rules keep their names M1 to M4.
* "YANG Data Model": added a paragraph explaining how the fixed elements of the
  management protocol stack (addresses, ports, CoAP methods and options) make
  the management traffic very compressible.
* "Set of Rules Editing": the YANG identifiers are written in quotes and in a
  fixed-width font.
* "Management Errors" moved to the end of the "Management messages" section.
* "iPATCH": iPATCH can only create a complete rule or delete a rule; the
  modification, addition or deletion of an element of an existing rule is
  refused with 4.03 Forbidden. The examples of modification were replaced by
  examples of creation, deletion and refused requests, and the Content-Format of
  the iPATCH requests was corrected (application/yang-instances+cbor-seq).
* "rule-status" is managed locally by each node according to its protocol state:
  it is never set by the other node, a value received in a creation is ignored,
  and it can be read with FETCH.
* "Deleting a rule": the management rules cannot be deleted; deleting the
  top-level object deletes all the other rules, each one following the deletion
  procedure, and keeps the management rules. The "Management Compression Rules"
  section now also refuses the deletion of a management rule.
* "Deleting a rule": deleting the top-level object is now a factory reset, which
  restores the set of rules present at the start of the SCHC instance.
* The "POST" subsection of "CoAP Methods" is renamed "RPC invocation (POST)".
* "RPC invocation (POST)": the definition of the RPCs is out of the scope of the
  document, they may be defined in other YANG Data Models, and "duplicate-rule"
  is given as an illustration. The input of an RPC acting on rules must provide
  their rule IDs, and its output a status.
* "RPC statements": "duplicate-rule" illustrates the use of RPCs, other RPCs may
  be defined in other documents for specific traffic patterns.
* The RPCs are now a section of their own ({{sec-rpcs}}), after "Management
  messages": its introduction gathers the general rules on the RPCs, and "RPC
  invocation (POST)" only describes the use of CoAP.
* The tables of the management rules M1 to M4 are moved to an appendix
  ({{sec-mgmt-rules-tables}}) and adapted to the management protocol stack:
  DevIID ::1 and AppIID ::2, ports of the Device swapped in M3 and M4, Uri-Path
  only in the requests, two Content-Formats for the requests, 2.04 Changed added
  to the responses with a payload, and 4.03, 4.09 and 5.03 added to the
  responses without payload. The description of M1 to M4 is clarified.
* Acknowledgments: Claude was used to validate the protocol used to modify the
  SoR.
* The 5.03 Service Unavailable responses carry no Max-Age option, so that they
  can be compressed by the management rules M2 and M4.
* "Restart of a node" ({{sec-restart}}) added: the rules and their status should
  be kept in a stable memory, the Guard period of an `invalid` rule restarts
  after a restart, and the freshness of the management requests is checked when
  the deduplication state is lost. {{RFC8613}} and RFC 9175 added to the
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

This work has been supported by the SCHC Chair from IMT Atlantique and Afnic.

The authors also thank Claude (Anthropic) for rewriting parts of this document
in more correct English, for checking its consistency with the YANG data model,
and for validating the protocol used to modify the SoR (see {{sec-rule-status}})
with a systematic exploration of a bounded model of this protocol.
