---
title: "Static Context Header Compression (SCHC) for the Internet Control Message Protocol (ICMPv6)"
abbrev: "SCHC ICMPv6 Compression"
docname: draft-ietf-schc-icmpv6-compression-03
category: std
ipr: trust200902
submissiontype: IETF
v: 3
area: "Internet"
workgroup: "schc Working Group"
keyword:
 - template
date:

author:
 -
    ins: D. Barthel
    name: Dominique Barthel
    country: "France"
    email: dominique.barthel@orange.com
 -
    ins: L. Toutain
    name: Laurent Toutain
    org: "IMT Atlantique"
    street: "2 rue de la Chataigneraie CS 17607"
    city: "Cesson-Sevigne Cedex"
    code: "35576"
    country: "France"
    email: laurent.toutain@imt-atlantique.fr
 -
    ins: M. Dumay
    name: Marion Dumay
    org: "Orange"
    email: marion.dumay@orange.com

normative:
  I-D.ietf-schc-universal-option:
  I-D.toutain-schc-rfc9363bis:
  RFC2119:
  RFC4443:
  RFC4861:
  RFC4884:
  RFC8174:
  RFC8724:
  RFC9363:

--- abstract

This document describes how the ICMPv6 protocol can be integrated into
the SCHC architecture. It extends the YANG Data Model with new field IDs
specific to ICMPv6 headers.

To enhance the compression of ICMPv6 error messages, the document also
introduces two new Matching Operators and two new Compression
Decompression Actions to manipulate the ICMPv6 payload.

Finally, for constrained networks such as LPWAN, it introduces a proxy
behavior, where a SCHC Core end-point may anticipate the device reaction
to incorrect messages.

--- middle

# Introduction {#introduction}

ICMPv6 {{RFC4443}} is the companion protocol of IPv6. It carries
information about the state of the network, which both a SCHC Device and
a SCHC Application can benefit from. 

When applying SCHC compression to IPv6 networks, users expect to perform
control operations such as ping to ensure that devices are still active.
In the same way, ICMPv6 error messages can be helpful for the Device
which may adapt its behavior when the Application becomes unreachable.
The Application may also benefit from the ICMPv6 error message produced
by the SCHC entity when the compression is not possible.

The ICMPv6
messages covered by this document are those defined in ICMPv6 protocol
{{RFC4443}}. The compression described in this document does not cover
other ICMPv6 messages, such as an extended format of the same messages
{{RFC4884}} and other messages used by the Neighbor Discovery Protocol
{{RFC4861}}.

## ICMPv6 Messages

ICMPv6 defines a generic message format, which is used to inform the
source of an IPv6 packets about errors during the packet delivery. It
also specifies messages used by the ping command to test connectivity
with a remote node.

{{RFC4443}} instantiates four such error messages:

* Destination Unreachable (type = 1),
* Packet Too Big (type = 2),
* Time Exceeded (type = 3) and
* Parameter Problem (type = 4).

{{RFC4443}} also defines two informational messages, the Echo Request
(type=128) and Echo Reply messages (type = 129), which provide support
for the ping application.

This document describes recommended compression of ICMPv6/IPv6 messages
(including header fields and structured payload) and extends SCHC by
specifying new Field Identifiers for ICMPv6 and two MO and two CDA to
compress the ICMPv6 payload. This covers different scenarios:

* ICMPv6 messages initiated by SCHC Devices. They can be sent in
  their SCHC-compressed form, in ICMPv6 messages traffic. This includes
  error messages, as well as informational echo request/reply traffic
* ICMPv6 error messages returned from the Internet after Device
  transmission. The core SCHC forwards a compressed version of the error
  message to the Device, including if necessary a compressed payload.
* Traffic coming from the Internet that would generate an error on the
  Device: if it can detect the situation, the SCHC Core directly
  responds with an ICMPv6 error message, acting as a surrogate to the
  Device.

# Terminology {#terminology}

This draft re-uses the Terminology defined in {{RFC8724}} and the
achitecture document.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all
capitals, as shown here.

* SCHC Core: SCHC End-Point located at the boundary of a regular IP
  network and a network that applies SCHC compression and fragmentation
* SCHC Device: The other end of the SCHC instance formed with the SCHC
  core.
* Application: entity sending packets to the SCHC Device or receiving
  packets from the SCHC Device. The Application may be co-located with
  the SCHC Core, but is usually located on the regular Internet.
* Regular Internet: Network location carrying uncompressed IPv6 packets.

# Use cases {#use-cases}

In the following sections, we will describe at {{basicICMPv6}} ICMPv6
message compression for all ICMPv6 messages specified in {{RFC4443}}. We
will then extend this basic compression with a specific focus on the
following cases:

* The Device is the originator of an Echo Request message, and therefore
  the destination of the Echo Reply message. These messages are
  compressed/decompressed by the device and the SCHC Core using SCHC
  rules that match the ICMPv6 fields (see {{DevicePings}}).
* The Device should have sent an ICMP error message, mainly in response
  to an incorrect incoming IPv6 message. In this case, as much as
  possible, the SCHC Core should act on behalf of the Device and
  originate the Unreachable ICMP Destination message, so that the Device
  and the network are protected from this unwanted traffic (see
  {{ProxyErrMsg}}).
* The Device is the destination of the ICMPv6 message, mainly in
  response to a packet sent by the device to the network that generates
  an error. In this case, we want the ICMPv6 message to reach the
  Device, and this document describes in {{ErrMsgCompr}} what SCHC
  compression should be applied. Since ICMPv6 error messages contain in
  the payload the original message which has triggered the error, SCHC
  can compress it using the rules in the reverse direction (see
  {{device-is-the-destination}}).


# ICMPv6 compression {#basicICMPv6}

This section defines ICMPv6 fields that can be compressed by SCHC.
{{RFC4443}} defines several formats with respect to the type of the
ICMPv6 message.

From them, several fields can be extracted (the Field ID identities are
specified in the SCHC YANG data model of
{{I-D.toutain-schc-rfc9363bis}}):

These fields are present in all the messages:

* ICMPv6 Type indicates the fields present in the message.
* ICMPv6 Code is related to the ICMPv6 type and does not have an impact
  on the message format.
* ICMPv6 Checksum covers the ICMPv6 message and part of the IPv6 header
  to protect against errors. It is generally
  reconstructed with the `compute-*` CDA and therefore is not
  transmitted.
* ICMPv6 Payload is part of the ICMPv6 protocol and is not directly
  originated from upper layers protocols, so this field may be
  compressed by SCHC at the ICMPv6 level. In the ICMPv6 error message,
  the payload can be compressed by SCHC compression rules, as it
  contains the IPv6 message header responsible for the error. For Echo
  Request and Echo Reply it contains a specific pattern to set the
  message length.

The other fields depends of the message type:

* ICMPv6 MTU is used by Packet Too Big message (type = 2) to carry the
  MTU expected by a node rejecting the packet forwarding. This 32-bit
  field carries an MTU that is at least the minimum IPv6 MTU (1280
  bytes), so its upper bits are almost always constant. It can be
  compressed with the MSB CDA, or with the LSB CDA if the Device is
  known to send smaller packets, sending e.g. only 11 bits.
* ICMPv6 Pointer is used by Parameter Problem message to indicate the
  position of a detected error in the original message. This 32-bit
  field is the offset, in bytes from the start of the invoking
  packet, of the octet where the error was detected. When the error is
  in the fixed IPv6 header (Code 0, erroneous header field, or Code 1,
  unrecognized Next Header, which points to octet 6), the value is
  lower than 40 and 6 bits are enough with the LSB CDA. When the error
  is in an extension header or an option (Code 2, unrecognized IPv6
  option, always, or Codes 0 and 1 beyond octet 39), the value is
  bounded by the size of the invoking packet, which an ICMPv6 error
  message limits to the minimum IPv6 MTU, so 11 bits are enough. A
  rule can use a mapping or the LSB CDA to cover the first case, and a
  second rule the other one.
* ICMPv6 Identifier and ICMPv6 Sequence Number are used by ping echo
  (type 128) and reply (type 129) messages.

Since the fields present in an ICMP message differ from one type to
another, it is not possible to use a single rule to compress all ICMP
messages. Examples of ICMP message compression Rules are given in
{{parsing-payload}}, {{DevicePings}} and {{device-is-the-destination}}.

## New FID {#new-fid}

This document introduces eight new Field IDs for ICMPv6
{{RFC4443}}:

* `fid-icmpv6-type`, `fid-icmpv6-code`, `fid-icmpv6-checksum`:
  present in every ICMPv6 message.
* `fid-icmpv6-mtu`: present in the Packet Too Big message.
* `fid-icmpv6-pointer`: present in the Parameter Problem message.
* `fid-icmpv6-identifier`, `fid-icmpv6-sequence`: present in the
  Echo Request/Reply message.
* `fid-icmpv6-payload`: the data following the ICMPv6 header.

Two more generic Field IDs, not specific to ICMPv6, are defined
alongside them in the `ietf-schc` module
({{I-D.toutain-schc-rfc9363bis}}). Being generic, they can be used
as-is when compressing other protocols:

* `fid-unused`: as its name suggests, used to skip an unused part of
  a header. This matters when parsing and compression happen on the
  fly, and reinforces the constraint that fields appear in the same
  order as in the header. Setting its Target Value to 0, its
  Matching Operator to `ignore`, and its Compression/Decompression
  Action to `not-sent` is RECOMMENDED.
* `fid-payload`: MUST be the last entry of the Rule, matching its
  position as the packet's trailing payload, and uses the new
  `fl-remaining` of {{I-D.ietf-schc-universal-option}}, which accepts any
  length without prefixing the residue with one, since the last entry's
  residue already runs to the end of the SCHC packet. With Matching
  Operator `ignore` and Compression/Decompression Action
  `value-sent`, it behaves exactly as the usual, implicit SCHC
  behavior, where the payload simply follows the compression
  residue. But it can also be used to intercept the payload:
  Compression/Decompression Action `not-sent` then elides it, or
  another CDA can apply a special treatment to compress it, instead
  of sending it verbatim.

# Parsing the payload {#parsing-payload}

## Invoking Packet Description in the Same Rule

A first possibility is to
continue the compression process inside the ICMPv6 payload, instead
of matching it against a separate Rule. {{fig-alt-icmpv6}} shows the
resulting Rule. This solution requires creating a Rule for each
original packet format.

The invoking header's Field Descriptors are distinct from the outer
header's, and are designed from the ICMPv6 message's point of view.

Fields designed with an application or device role remain unchanged
(e.g. `ipv6-deviid` or `udp-app-port`), but the direction is
reversed during parsing. Field Position is also incremented if a field is repeated,
as for the IPv6 fields in the example.

The compression mechanism MUST also include the port numbers: an
ICMPv6 error message exists to inform the source that a given flow
failed to reach its destination, and the port numbers are part of
that flow's identification. It is RECOMMENDED to be able to fully
reconstruct the Layer 4 header this way, not just the ports.

| # | Field | FL | FP | DI | TV | MO | CDA |
|---|---|---|---|---|---|---|---|
|  | *Outer IPv6 header* |  |  |  |  |  |  |
| 0 | ipv6-version | 4 | 1 | Bi | 6 | equal | not-sent |
| 1 | ipv6-trafficclass | 8 | 1 | Bi | 0 | ignore | not-sent |
| 2 | ipv6-flowlabel | 20 | 1 | Bi | 0 | ignore | not-sent |
| 3 | ipv6-payload-length | 16 | 1 | Bi | - | ignore | compute |
| 4 | ipv6-nextheader | 8 | 1 | Bi | 58 | equal | not-sent |
| 5 | ipv6-hoplimit | 8 | 1 | Up | - | ignore | value-sent |
| 6 | ipv6-hoplimit | 8 | 1 | Dw | 1 | equal | not-sent |
| 7 | ipv6-devprefix | 64 | 1 | Bi | aaaa:: | equal | not-sent |
| 8 | ipv6-deviid | 64 | 1 | Bi | ::zzzz | equal | not-sent |
| 9 | ipv6-appprefix | 64 | 1 | Bi | - | ignore | value-sent |
| 10 | ipv6-appiid | 64 | 1 | Bi | - | ignore | value-sent |
|  | *ICMPv6 header* |  |  |  |  |  |  |
| 11 | icmpv6-type | 8 | 1 | Bi | 1 | equal | not-sent |
| 12 | icmpv6-code | 8 | 1 | Bi | 4 | equal | not-sent |
| 13 | icmpv6-checksum | 16 | 1 | Bi | 0 | ignore | compute |
| 14 | unused | 32 | 1 | Bi | 0 | ignore | not-sent |
|  | *Invoking (embedded) IPv6 header* |  |  |  |  |  |  |
| 15 | ipv6-version | 4 | 2 | Bi | 6 | equal | not-sent |
| 16 | ipv6-trafficclass | 8 | 2 | Bi | 0 | ignore | not-sent |
| 17 | ipv6-flowlabel | 20 | 2 | Bi | 0 | ignore | not-sent |
| 18 | ipv6-payload-length | 16 | 2 | Bi | - | ignore | compute |
| 19 | ipv6-nextheader | 8 | 2 | Bi | 17 | equal | not-sent |
| 20 | ipv6-hoplimit | 8 | 2 | Dw | 1 | equal | not-sent |
| 21 | ipv6-hoplimit | 8 | 2 | Up | - | ignore | value-sent |
| 22 | ipv6-devprefix | 64 | 2 | Bi | aaaa:: | equal | not-sent |
| 23 | ipv6-deviid | 64 | 2 | Bi | ::zzzz | equal | not-sent |
| 24 | ipv6-appprefix | 64 | 2 | Bi | - | ignore | value-sent |
| 25 | ipv6-appiid | 64 | 2 | Bi | - | ignore | value-sent |
|  | *Invoking (embedded) UDP header* |  |  |  |  |  |  |
| 26 | udp-dev-port | 16 | 1 | Bi | 5683 | equal | not-sent |
| 27 | udp-app-port | 16 | 1 | Bi | - | ignore | value-sent |
| 28 | udp-length | 16 | 1 | Bi | 0 | ignore | compute |
| 29 | udp-checksum | 16 | 1 | Bi | 0 | ignore | compute |
| 30 | payload | remaining | 1 | Bi | - | ignore | not-sent |
{: #fig-alt-icmpv6 title="ICMPv6 Rule Embedding the Invoking IPv6/UDP Header In Place"}

The `unused` entry skips the 32 bits following the Checksum.

Note that the IPv6 fields of the invoking header are not sent as residue, but
are used to check that the ICMPv6 packet is not rogue: an error
message whose invoking header does not match the Rule is not
decompressed.


The result is compact: the payload is elided and the fields of the
invoking IPv6 header are not sent. They are nevertheless matched
against the Rule, which helps to validate that the ICMPv6 message
refers to a packet that really was exchanged with the Device.

Compared to the solution compressing the invoking packet with its own
Rule ({{newMO}}), this one has the following advantages:

* the residue contains a single RuleID, whereas the other solution
  sends a second RuleID for the invoking packet;
* no length has to be sent to delimit the compressed invoking packet,
  and no padding is needed between the two parts.

These advantages come at a cost: the Rule is longer, and since it
describes the complete stack, a Rule is needed for each original
packet format, where the other solution needs a single ICMPv6 Rule
for all of them.

~~~ aasvg
+----------+------------------------------------------+
|  RuleID  |       IPv6/ICMPv6/IPv6/UDP residue       |
+----------+------------------------------------------+
~~~
{: #fig-alt-icmpv6-residue title="Residue Layout for that ICMPv6 Rule"}

## New Matching Operators and Compression/Decompression Actions {#newMO}

Instead of describing the invoking packet in the same Rule, it can be
compressed with its own Rule. This document introduces two Matching
Operators and two Compression/Decompression Actions:

* `mo-rule-match`: returns true if the Target Value matches another
  Rule, keeping the Up/Down direction.
* `mo-rev-rule-match`: same as `mo-rule-match`, but reversing the
  Up/Down direction.
* `cda-compress-sent`: sends a compressed version of the Target
  Value, using the matched Rule.
* `cda-rev-compress-sent`: same as `cda-compress-sent`, using the
  direction-reversed counterpart of the matched Rule.

The reversed forms exist because, per RFC 4443 {{RFC4443}}, an
ICMPv6 error message carries back as much as possible of the IPv6
packet that triggered it -- a packet that was sent in the opposite
direction from the error message itself. Compressing that embedded
copy therefore means matching it against, and generating its
residue from, a Rule for the reverse direction.

### Example, ICMPv6 Error with Reverse Compression

{{fig-example-icmpv6}} gives a Rule for the "Time Exceeded" message,
prefixed with the outer IPv6
header carrying the ICMPv6 message itself (as in {{fig-alt-icmpv6}}
above): the Type and Code identify the error, the Checksum is
recomputed on decompression, the 32-bit "Unused" field mandated by
RFC 4443 {{RFC4443}} for this message is elided with `fid-unused`
({{new-fid}}), and the Payload -- the offending IPv6 packet -- is
matched and compressed against its own (Up) Rule with
`rev-rule-match` and `rev-compress-sent`, instead of being sent in
full.

| # | Field | FL | FP | DI | TV | MO | CDA |
|---|---|---|---|---|---|---|---|
|  | *Outer IPv6 header* |  |  |  |  |  |  |
| 0 | ipv6-version | 4 | 1 | Dw | 6 | equal | not-sent |
| 1 | ipv6-trafficclass | 8 | 1 | Dw | 0 | ignore | not-sent |
| 2 | ipv6-flowlabel | 20 | 1 | Dw | 0 | ignore | not-sent |
| 3 | ipv6-payload-length | 16 | 1 | Dw | - | ignore | compute |
| 4 | ipv6-nextheader | 8 | 1 | Dw | 58 | equal | not-sent |
| 5 | ipv6-hoplimit | 8 | 1 | Dw | 1 | equal | not-sent |
| 6 | ipv6-devprefix | 64 | 1 | Dw | aaaa:: | equal | not-sent |
| 7 | ipv6-deviid | 64 | 1 | Dw | ::zzzz | equal | not-sent |
| 8 | ipv6-appprefix | 64 | 1 | Dw | - | ignore | value-sent |
| 9 | ipv6-appiid | 64 | 1 | Dw | - | ignore | value-sent |
|  | *ICMPv6 header* |  |  |  |  |  |  |
| 10 | icmpv6-type | 8 | 1 | Dw | 3 | equal | not-sent |
| 11 | icmpv6-code | 8 | 1 | Dw | \[0,1\] | match-mapping | mapping-sent |
| 12 | icmpv6-checksum | 16 | 1 | Dw | - | ignore | compute |
| 13 | unused | 32 | 1 | Dw | 0 | ignore | not-sent |
| 14 | icmpv6-payload | variable-bits | 1 | Dw | 0 | rev-rule-match | rev-compress-sent |
{: #fig-example-icmpv6 title="ICMPv6 Error Compressed Against a Reverse-Direction Rule"}

The bytes following the `unused` entry are assigned to `icmpv6-payload` and
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
({{I-D.ietf-schc-universal-option}}), so counting it in bytes keeps that
length prefix itself compact.

~~~ aasvg
                             |----------------Length-----------------|
+---------+---------+--------+---------+---------+---------+---------+
| RuleID  | IPv6/   | Length | RuleID  | IPv6/   | Payload | Padding |
| (outer) | ICMPv6  |        | (inner) | UDP     |         |         |
|         | residue |        |         | residue |         |         |
+---------+---------+--------+---------+---------+---------+---------+
~~~
{: #fig-icmpv6-residue title="Residue Layout for an ICMPv6 Error Compressing an Embedded, Compressed Header"}

# Device does a ping {#DevicePings}

A Device may send an Echo Request message to check the availability of
the network and the host running the Application.

If a ping Echo Request is generated by a Device, then SCHC compression
applies.

The format of an ICMPv6 Echo Request message is described in
{{Fig-ICMPv6-Echo-Request}}, with Type=128 and Code=0.

~~~
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |     Type      |     Code      |          Checksum             |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |           Identifier          |        Sequence Number        |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |     Data ...
      +-+-+-+-+-
~~~
{: #Fig-ICMPv6-Echo-Request title="ICMPv6 Echo Request message format"}

If we assume that one rule will be devoted to compressing Echo Request
messages, then the Type and Code are known in the rule to be 128 and 0
and can therefore be elided with the not-sent CDA.

{{RFC4443}} states that the identifier and sequence number are meant to
"help in matching Echo Responses to this Echo Request" and that they
"may be zero". Data are "zero or more bytes of arbitrary data".

For constrained devices or networks, we recommend that the Identifier be
zero, the Sequence Number be a counter on 3 bits, and the Data be zero
bytes (absent). Therefore, Identifier is elided with the not-sent CDA,
Sequence Number is transmitted on 3 bits with the LSB CDA and no Data is
transmitted.

The data part is defined in the rule through the ICMPv6 Payload field.
The payload can be sent as a residue with a value-sent. It is also
possible to elide the data by setting them in the Target Value and use a
not-sent CDA.

When the destination receives the Echo Request message, it will respond
with an Echo Reply message. This message bears the same format as the
Echo Request message but with Type = 129 (see
{{Fig-ICMPv6-Echo-Request}}).

{{RFC4443}} states that the Identifier, Sequence Number, and Data fields
of the Echo Reply message shall contain the same values as the invoking
Echo Request message. Therefore, a rule shall be used similar to that
used for compressing the Echo Request message.

## Rule example {#rule-example}

The following rule gives an example of a SCHC compression. The type can
be elided if the direction is taken into account. Identifier is ignored
and generated as 0 at decompression. This implies that only one single
ping can be launched at any given time on a device. Finally, only the
least significant 8 bits of the sequence number are sent on the LPWAN,
allowing a serie of 255 consecutive pings.

| # | Field | FL | FP | DI | TV | MO | CDA | Sent bits |
|---|---|---|---|---|---|---|---|---|
| 0-9 | *IPv6 headers description* |  |  |  |  |  |  |  |
| 10 | icmpv6-type | 8 | 1 | Up | 128 | equal | not-sent |  |
| 11 | icmpv6-type | 8 | 1 | Dw | 129 | equal | not-sent |  |
| 12 | icmpv6-code | 8 | 1 | Bi | 0 | equal | not-sent |  |
| 13 | icmpv6-checksum | 16 | 1 | Dw |  | ignore | compute-* |  |
| 14 | icmpv6-identifier | 16 | 1 | Bi | 0 | ignore | not-sent |  |
| 15 | icmpv6-sequence | 16 | 1 | Bi | 0 | MSB(13) | LSB | 3 |
| 16 | icmpv6-payload | remaining | 1 | Bi | 0 | ignore | value-sent | data*8 |
{: #Fig-ping-up title="Example of compression rule for a ping from the device"}

The transmission cost of the Echo Request message is therefore the size
of the Rule Id + 3 bits and the data size increased of the Payload
residue size. The rule ID Length can be chosen to avoid adding padding.

# Device is pinged {#action}

If a Device is pinged, a Rule similar to the one described in the
previous section can be used, but it may be more efficient to delegate
the answer to the SCHC Core.

Selecting a compression Rule triggers compression and sends the
resulting SCHC Packet to the other end.

We introduce a new element called `action` for the Rule. An Action
can change that behavior: the Rule is still selected through its MO
results, but the incoming packet is forwarded to a procedure defined in
the `action`. Since the Rule is then used only for the selection, the
Compression/Decompression Actions of its entries do not matter and are
set to `not-sent`.

The motivating example is an ICMPv6 {{RFC4443}} Echo Request sent to
a Device. The Device receives the compressed message and answers
with an Echo Reply, but propagating the Echo Request over the LPWAN
costs two messages on a constrained link, and the Identifier and
Sequence Number fields compress poorly when the sender is unaware of
the compression scheme. The Action `action-proxy-ping` avoids that
traffic: the core SCHC C/D answers the Echo Request on behalf of the
Device.

The Action takes a parameter, in seconds, giving the interval during
which the Device is considered active. During this interval, the
proxy echoes the ping requests; after it, they are discarded. The
parameter is carried in `action-value`, a list reusing the Target
Value structure, so an Action may take several arguments.

{{fig-action-rule}} shows a compression Rule for pinging a Device,
with the identity short names used in the Field and CDA columns. The
Action applies to the Rule as a whole, and is therefore shown above its
entries.

~~~ aasvg
+--------------------------+
| Action: proxy-ping(300)  |
+----+----------------+----+----+----+----+---------+----------+
| #  | Field          | FL | FP | DI | TV | MO      | CDA      |
+----+----------------+----+----+----+----+---------+----------+
| IPv6 Headers description (omitted)                           |
+----+----------------+----+----+----+----+---------+----------+
| 10 | icmpv6-type    | 8  | 1  | Dw | 128| equal   | not-sent |
| 11 | icmpv6-code    | 8  | 1  | Bi | 0  | equal   | not-sent |
| 12 | icmpv6-        | 16 | 1  | Bi | -  | ignore  | not-sent |
|    | identifier     |    |    |    |    |         |          |
| 13 | icmpv6-        | 16 | 1  | Bi | 0  | MSB(24) | not-sent |
|    | sequence       |    |    |    |    |         |          |
+----+----------------+----+----+----+----+---------+----------+
~~~
{: #fig-action-rule title="Example Compression Rule for a Ping to a Device"}

In the data model, the Action is added to the compression Rule
itself, as a sibling of its entries, and not to a single entry:

* `action`: an identityref to `action-type`, defaulting to
  `action-forward`. Being optional and defaulted, a Rule that does
  not carry it behaves exactly as in RFC 8724 {{RFC8724}}.
* `action-value`: the Action's arguments, a list based on the Target
  Value structure, as for Matching Operator and
  Compression/Decompression Action arguments.

The corresponding new identities are:

* `action-base-type`: base identity for the Action applied to a
  packet matching a compression Rule.
* `action-forward`: the default, where the packet is handed over to
  the lower layer, as specified in RFC 8724 {{RFC8724}}.
* `action-proxy-ping`: the packet is processed locally by a ping
  proxy, which answers the Echo Request on behalf of the Device.

# Device is the source of an ICMPv6 error message {#ProxyErrMsg}

As stated in {{RFC4443}}, a node should generate an ICMPv6 message in
response to an IPv6 packet that is malformed or which cannot be
processed due to some incorrect field value.

The general intent of this document is to spare both the Device and the
LPWAN network this un-necessary traffic. The incorrect packets should be
caught at the SCHC Core and the ICMPv6 notification should be sent back
from there.

~~~ aasvg
     Device       NGW     SCHC Core                    Internet Host

       |           |            |    Destination Port=XXX    |
       |           |            |<---------------------------|
       |           |            |                            |
       |           |            |--------------------------->|
       |           |            | ICMPv6 Port Unreachable    |
       |           |            |                            |
       |           |            |                            |
~~~
{: #Fig-ICMPv6-up title="Example of ICMPv6 error message sent back to the Internet"}

{{Fig-ICMPv6-up}} shows an example of an IPv6 packet trying to reach a
Device.

Let's assume that no rule matches the incoming packet (i.e. there is no
co-compression rule)

Instead of sending the packet over the LPWAN and having this packet
rejected by the Device, the SCHC Core issues an ICMPv6 error message
"Destination Unreachable" (Type 1) with Code 1 ("Port Unreachable") on
behalf of the Device.

In that case the SCHC C/D MAY act as a router (i.e. it MUST have a
routable IPv6 address to generate an ICMPv6 message). When compressing a
packet containing an IPv6 header, no compression rules are found and:

* if the incoming packet contains some extension headers and no SCHC
  Rule covers this case, a parameter problem may be generated
  (type 4),
* no rule contains the IPv6 device address found in the incoming packet,
  a no route to destination ICMPv6 message (type 0, code 3) may be
  generated,
* a device IPv6 address is found, but no port matches, a port
  unreachable ICMPv6 message (type 0, code 4) may be generated,
* if the incoming packet is too large for any of the fragmentation
  rules, an ICMPv6 Message Too big MAY be generated with the largest
  size allowed by the fragmentation rules.

# Device is the destination of an ICMPv6 error message {#device-is-the-destination}

In this situation, a Device has been configured to send information to a
server on the Internet. If this server becomes no longer accessible, an
ICMPv6 message will be generated back towards the Device by either an
intermediate router or the destination. This information can be useful
to the Device, for example, for reducing the reporting rate in case of
periodic reporting of data.

Therefore, the ICMPv6 error message should reach the Device. The data
inside this error message includes the packet at the origin of the
error. It should be compressed by the SCHC Core, but in the reverse
direction. The MOs and CDAs defined in {{newMO}} perform this
operation.

~~~ aasvg
     Device       NGW     SCHC Core                    Internet Server

       |           |            |                            |
       | SCHC compressed IPv6   |                            |
       |~~~~~~~~~~~|----------->|----------------------X     |
       |           |            |<---------------------      |
       |<~~~~~~~~~~|------------| ICMPv6 Host unreachable    |
       |SCHC compressed ICMPv6  | payload: IPv6 packet       |
       |payload: compressed IPv6|                            |
       |           |            |                            |
~~~
{: #Fig-ICMPv6-down title="Example of ICMPv6 error message sent back to the Device"}

{{Fig-ICMPv6-down}} illustrates this behavior. The ICMPv6 error message
is compressed as described in {{ErrMsgCompr}} and forwarded over the
LPWAN to the Device.

The SCHC returning message contains the SCHC residue of the ICMPv6
message and MAY contain the compressed original message contained in the
ICMP message. The compression can be done by the SCHC Core by reversing
the direction as if this message was issued by the device.

## Example of ICMPv6 error message compression. {#ErrMsgCompr}

The ICMPv6 error messages defined in {{RFC4443}} contain the fields
shown in {{Fig-ICMP-error}}.

~~~
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |     Type      |     Code      |          Checksum             |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                           Value/Unused                        |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                    As much of invoking packet                 |
      +                as possible without the ICMPv6 packet          +
      |                exceeding the minimum IPv6 MTU                 |
~~~
{: #Fig-ICMP-error title="ICMPv6 Error Message format"}

{{RFC4443}} states that Type can take the values 1 to 4, and Code can be
set to values between 0 and 6. Value is unused for the Destination
Unreachable and Time Exceeded messages. It contains the MTU for the
Packet Too Big message and a pointer to the byte causing the error for
the Parameter Error message.

The payload is viewed as a field. An unsued field MUST not appear in the
compressoin rules.

The source address of the message SHOULD be `ignore`, since it can be
initiated by any router on the path.

The following generic rule can therefore be used to compress all ICMPv6
error messages as defined today. More specific rules can also be defined
to achieve better compression of some error messages.

The Type field can be associated with a matching list \[1, 2, 3, 4\] and
is therefore compressed to 2 bits. Code can be reduced to 3 bits using
the LSB CDA. Compression of the Value field for the Packet Too Big
message is discussed in {{basicICMPv6}}.

The first rule example {{Fig-icmp-error1}} just sends the ICMP type and
code as residue to the device.

| # | Field | FL | FP | DI | TV | MO | CDA | Sent bits |
|---|---|---|---|---|---|---|---|---|
| 0-9 | *IPv6 headers description* |  |  |  |  |  |  |  |
| 10 | icmpv6-type | 8 | 1 | Dw | 3 | equal | not-sent |  |
| 11 | icmpv6-code | 8 | 1 | Dw | \[0, 1\] | match-mapping | mapping-sent | 1 |
| 12 | icmpv6-checksum | 16 | 1 | Dw |  | ignore | compute-* |  |
| 13 | unused | 32 | 1 | Dw | 0 | ignore | not-sent |  |
| 14 | icmpv6-payload | remaining | 1 | Dw | 0 | ignore | not-sent | 0 |
{: #Fig-icmp-error1 title="Example of compression rule for a ICMP error to a device"}

The compression of the offending packet in the reverse direction relies
on the `rev-rule-match` MO and the `rev-compress-sent` CDA defined in
{{newMO}}. Used with the `not-sent` CDA, `rev-rule-match` only checks
that a Rule matches the payload, which is then elided, so the residue
still contains only the ICMPv6 type and code. Used with
`rev-compress-sent`, the offending packet is compressed and sent as
residue, as detailed in {{fig-example-icmpv6}} and its residue layout
({{fig-icmpv6-residue}}).

# YANG Data Model Extensions {#yang-tree}

This document extends the SCHC YANG Data Model of {{RFC9363}}. The
complete YANG module, including all the identities and the leaves
listed below, is found in {{I-D.toutain-schc-rfc9363bis}}. This
section only lists the elements related to this document.

The new Field IDs, all based on `fid-icmpv6-base-type` for the ICMPv6
ones ({{new-fid}}):

* `fid-icmpv6-type`, `fid-icmpv6-code`, `fid-icmpv6-checksum`,
  `fid-icmpv6-mtu`, `fid-icmpv6-pointer`, `fid-icmpv6-identifier`,
  `fid-icmpv6-sequence`: the fields of the ICMPv6 header.
* `fid-icmpv6-payload`: the ICMPv6 payload, parsed as a specific field.
* `fid-unused`: skips an unused part of a header.
* `fid-payload`: the trailing payload of the packet, which uses the
  `fl-remaining` Field Length function of
  {{I-D.ietf-schc-universal-option}}.

The new Matching Operators ({{newMO}}):

* `mo-rev-rule-match`: the value contained in the Field Value matches a
  Rule. The direction used for matching is the opposite of the incoming
  message: UP becomes DOWN and DOWN becomes UP. This MO can be used to
  test if the Payload contained in the ICMPv6 message matches a Rule.
  This means that the original packet, at the origin of the ICMPv6
  message, may have been generated from the SCHC decompression.
* `mo-rule-match`: the value contained in the Target Value matches a
  Rule. The direction is the one of the incoming message. This MO is
  not used for ICMPv6 messages, but since it can be used in other
  situations, it has been included in the Data Model.

The new Compression/Decompression Actions ({{newMO}}). The Field Value
may be compressed by a Rule. The result SHOULD be included in the SCHC
message as a variable length residue. It contains the Rule ID used by
the compression, the residue, the payload and some padding bits, since
the variable length in it is in bytes.

* `cda-rev-compress-sent`: the direction used for compression is the
  opposite of the incoming message: UP becomes DOWN and DOWN becomes UP.
* `cda-compress-sent`: the direction used for compression is the same
  as for the incoming message.

The new elements of the Rule and the Action identities ({{action}}):

* `action` and `action-value`: leaves of the compression Rule, as
  siblings of its entries, giving the Action and its arguments.
* `action-base-type`, `action-forward`, `action-proxy-ping`: the
  identities of the Actions.

# Security considerations {#security-considerations}

flood the return path with ICMP error messages.

When the invoking packet is described in the same Rule
({{parsing-payload}}), the IPv6 fields of the invoking header are not
sent, but are used to check that the ICMPv6 packet is not rogue: an
error message whose invoking header does not match the Rule is not
decompressed.

# IANA Considerations {#iana-considerations}

TODO

# Contributors {#contributors}

The following people have been co-authors of precursor versions of this
draft. Their contribution is deeply appreciated and acknowledged.

* Arunprabhu Kandasamy (Acklio)
* Diego Dujovne (Universidad Diego Portales)
* Juan Carlos Zuniga (Cisco)

# Changes {#changes}

* ICMPv6 MTU ({{basicICMPv6}}): added a remark on compressing the MTU
  with the MSB/LSB CDA (the LSB example of the Packet Too Big Value
  field had disappeared from {{device-is-the-destination}}).
* {{device-is-the-destination}}: the sentence on sending the Value
  field on 11 bits was replaced by a reference to {{basicICMPv6}}.
* {{parsing-payload}}: the subsection describing the invoking packet in
  the same Rule now comes before the new MOs and CDAs.
  The subsection title was changed from "Describing the Invoking
  Packet in the Same Rule" to "Invoking Packet Description in the
  Same Rule".
* {{parsing-payload}}: added a note after {{fig-alt-icmpv6}} stating that
  the invoking IPv6 fields are not sent but are used to detect rogue
  ICMPv6 packets.
* Throughout: YANG identifiers in the text are now typeset as code (backticks),
  so they appear in a monospaced font in the PDF/HTML versions, instead of being
  written between double quotes. The typo "rev-rule match" is corrected to
  `rev-rule-match`.
* ICMPv6 Pointer ({{basicICMPv6}}): added a remark on compressing the
  Pointer: 6 bits with the LSB CDA for errors in the fixed IPv6 header
  (value lower than 40), up to 11 bits for errors in extension headers
  or options.
* {{security-considerations}}: added the same note on the invoking IPv6 fields
  used to detect rogue ICMPv6 packets.
* {{parsing-payload}}: the sentence stating that the in-place description of the
  invoking packet requires a Rule for each original packet format was moved
  from the end of the subsection to its introduction.
* {{parsing-payload}}: rewrote the paragraph following {{fig-alt-icmpv6}} (it
  said the result "should be better", which no longer follows the other
  solution) and added objective arguments comparing the two solutions.
* {{parsing-payload}}: the note on the `unused` entry was moved from the second
  example ({{fig-example-icmpv6}}) to the first one ({{fig-alt-icmpv6}}).
* {{newMO}}: the list of the new MOs and CDAs was merged with their
  descriptions, with one item per identifier.
* {{basicICMPv6}}, {{DevicePings}}: the remark that the Checksum is
  reconstructed with the `compute-*` CDA was moved from the ping section to the
  description of the ICMPv6 Checksum field.
* {{ProxyErrMsg}}: the Parameter Problem case now states that the incoming
  packet contains extension headers that no SCHC Rule covers.
* {{device-is-the-destination}}: removed the duplicate explanation of
  `rev-rule-match` and the second and third rule examples
  (`Fig-icmp-error2`, `Fig-icmp-error3`), now covered by {{newMO}} and
  {{fig-example-icmpv6}}, which are referenced instead.
* {{action}} ("Device is pinged"): new section copied from {{I-D.toutain-schc-rfc9363bis}}, describing the
  Action of a Rule (`action`, `action-value`, `action-forward`,
  `action-proxy-ping`) with the ping proxy example; the reference to the
  entry-index column and short names of the other draft was reworded, and the
  reference to the draft-barthel-lpwan-oam-schc draft was removed.
* {{action}}: the section now starts by stating that a Rule similar to the one of
  {{DevicePings}} can be used, but that it may be more efficient to delegate
  the answer to the SCHC Core.
* {{action}}: added the sentence introducing the new `action` element of the Rule.
* {{action}}: in {{fig-action-rule}}, the Echo Request Type is written 128 (it was 80,
  its hexadecimal value) and the Code and Sequence Number target values 0, like the
  other tables; the paragraph layout was cleaned up.
* {{yang-tree}}: the section was renamed "YANG Data Model Extensions" and now
  lists every YANG element related to this document (Field IDs, MOs, CDAs,
  Action), stating that the complete module is in {{I-D.toutain-schc-rfc9363bis}}.
* Authors: added Marion Dumay as the last co-author.
