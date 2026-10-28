# Changes to Apply to RFC 9363 for rfc9363bis

This document lists, section by section, the changes to apply to RFC 9363
([rfc9363.md](rfc9363.md)) to include what is defined in
the previous version of draft-toutain-schc-rfc9363bis, now
[draft-toutain-schc-rfc9363bis-old.md](draft-toutain-schc-rfc9363bis-old.md). The
reference for the YANG module is ietf-schc@2026-10-27
([ietf-schc@2026-10-27.yang](ietf-schc@2026-10-27.yang)).

For each change, "bis:" gives the section of draft-toutain-schc-rfc9363bis-old
where the text and the YANG fragments can be taken from.

## ~~1. Front Matter~~

* ~~docname: draft-ietf-schc-rfc9363bis-00 (or the name chosen by the WG),
  with "obsoletes: 9363". The workgroup becomes "schc" (lpwan is closed).~~
* ~~Abstract: add a sentence listing the extensions (universal entries, new
  Field IDs and functions, Action, compound ACK, rule management).~~
* ~~Add a "Changes from RFC 9363" section (or appendix), summarizing the items
  of this document.~~

## 2. References

To add to the normative references:

* ~~RFC 7950 (YANG 1.1): already cited by RFC 9363 as informative; it becomes
  normative, since the deprecation rules of Section 11 are applied.~~
* ~~RFC 9595 (YANG SID files), for the SID allocation.~~
* ~~RFC 9441 (SCHC Compound ACK), whose YANG module is merged.~~
* ~~draft-ietf-schc-universal-option~~, ~~draft-ietf-schc-8824-update~~ and
  ~~draft-ietf-schc-icmpv6-compression~~, which define the new Field IDs,
  functions, MOs and CDAs.

To add to the informative references: ~~RFC 4443 (ICMPv6)~~, RFC 7641
(Observe), RFC 7959 (Block), RFC 8407 (YANG guidelines),
~~draft-ietf-core-oscore-key-update (KUDOS)~~, draft-barthel-lpwan-oam-schc
(Action: not needed, the Action is defined in
draft-ietf-schc-icmpv6-compression), draft-toutain-schc-sid-allocation,
draft-toutain-core-private-sid-translation and
~~draft-toutain-schc-coreconf-management (rule management)~~.

## 3. Introduction (RFC 9363 Section 1)

* ~~Keep the two goals of RFC 9363, and add the third one: the management of
  the Rules by the SCHC instances themselves (creation and deletion of Rules
  with CORECONF, see draft-toutain-schc-coreconf-management).~~
* ~~Explain why the module is revised rather than augmented: the key of the
  "entry" list cannot be changed in place (RFC 7950, Section 11), hence the
  new "entry-universal" structure and the deprecation of the old one (bis:
  Universal Options).~~

## ~~4. Terminology (RFC 9363 Section 3)~~

Add:

* ~~Entry index: position of an entry in a compression Rule, key of the
  "entry-universal" list.~~
* ~~Space ID: identifier of a protocol option space (e.g. the CoAP options).~~
* ~~Universal Option: option referenced by a Space ID and its option number,
  instead of a dedicated Field ID.~~
* ~~Action: processing applied to a packet matching a compression Rule
  ("forward" by default).~~
* ~~Management Rule, Guard period, Rule status: as defined in
  draft-toutain-schc-coreconf-management.~~

## 5. SCHC Rules (RFC 9363 Section 4)

### 5.1 Compression Rules (Section 4.1)

* Update Figure 1 ("Compression Decompression Context") with a first column
  numbering the entries (entry index). bis: Universal Options, Example 1.
* State that the entries are ordered as the fields in the header (RFC 8724),
  and that "entry-universal" is "ordered-by user", the entry index being
  assigned sequentially.

### 5.2 Identifier Generation (Section 4.2)

* ~~No change of principle. Mention the new common ancestor
  "space-field-id-base-type" of "fid-base-type" and "space-id-base-type",
  used by the "field-or-space" choice.~~ (done in the Space ID section)

### 5.3 Convention for Field Identifier (Section 4.3)

Add the following subsections:

* ~~Universal Options and Space ID (bis: Universal Options, Space ID):
  "space-id-base-type", "space-id-coap", "space-id-type"; an entry is either a
  regular field ("field-id") or an option ("space-id" and "universal-value").~~
* Deprecation of the twenty per-option CoAP Field IDs (bis: Deprecating the
  Per-Option CoAP FIDs): "fid-coap-option-if-match" to
  "fid-coap-option-no-response" are marked "status deprecated", with the table
  giving the replacing option number. They MUST NOT be used in
  "entry-universal", and no SID is allocated to them.
* OSCORE and KUDOS suboptions (bis: OSCORE and KUDOS Suboptions):
  "fid-coap-option-oscore-flags-flagbits", "fid-coap-option-oscore-flags-n",
  "fid-coap-option-kudos-x", "fid-coap-option-kudos-x-flagbits",
  "fid-coap-option-kudos-x-m", "fid-coap-option-kudos-nonce".
  "fid-coap-option" stays current, as the base of these suboptions.
* ICMPv6 Field IDs (bis: ICMPv6, New FID): "fid-icmpv6-base-type",
  "fid-icmpv6-type", "-code", "-checksum", "-mtu", "-pointer", "-identifier",
  "-sequence", "-payload".
* Generic Field IDs (bis: ICMPv6, New FID): "fid-unused" (to skip a part of
  a header) and "fid-payload" (last entry of a Rule, with "fl-remaining").
* Remove "fid-oscore-base-type": it was never used as a base. Warning: this
  identity is published in RFC 9363; RFC 7950, Section 11, does not allow to
  remove it, so it should be marked "status deprecated" (or "obsolete")
  instead of being removed.

### 5.4 Convention for Field Length (Section 4.4)

bis: Field Length Functions, Variable Length in Bits.

* ~~"fl-length-bytes" and "fl-length-bits": the length is given by the value
  of another entry, whose entry index is carried in the new
  "field-length-value" leaf. "fl-token-length" is equivalent to
  "fl-length-bytes" with the index of the TKL entry.~~
* ~~"fl-remaining": length of the rest of the packet, without length prefix,
  for the last entry of a Rule.~~
* ~~"fl-variable-bits": as "fl-variable", with a length prefix counted in
  bits, same escape structure (RFC 8724, Section 7.4.2).~~
* Add the OSCORE outer header example (bis: Example 1) and the non-byte-aligned
  MSB example (bis: Example, Non-Byte-Aligned MSB Residue).

### 5.5 Convention for Matching Operator (Section 4.8)

* ~~Add "mo-rule-match" and "mo-rev-rule-match" (bis: ICMPv6, New Matching
  Operators): the Target Value matches another Rule, keeping or reversing the
  direction.~~

### 5.6 Convention for Compression Decompression Actions (Section 4.9)

* ~~Add "cda-compress-sent" and "cda-rev-compress-sent" (bis: ICMPv6): the
  field is sent compressed with the matched Rule, or with its
  direction-reversed counterpart.~~
* Section 4.9.1 says that no CDA requires arguments: update it, since these
  new CDAs refer to a Rule.
* Optionally, the ICMPv6 alternative with the invoking IPv6 header described
  in the Rule (bis: Alternative: Include IPv6 in Header Format), if the WG
  keeps it.

### ~~5.7 New Section: Action~~

~~bis: Action. New leaves "action" (identityref "action-type", default
"action-forward") and "action-value" (list based on the Target Value
structure), at the level of the compression Rule, not of an entry. Identities
"action-base-type", "action-forward", "action-proxy-ping", typedef
"action-type". Include the ping Rule example.~~ The identities stay in
"ietf-schc", but their behavior and the example are described in
draft-ietf-schc-icmpv6-compression.

### 5.8 Fragmentation Rule (Section 4.10)

* Section 4.10.4 (Acknowledgment Behavior): add the compound ACK of RFC 9441
  (bis: Compound ACK): leaves "bitmap-format" (default "bitmap-RFC8724") and
  "last-bitmap-compression" (default true) in the "ack-on-error" case;
  identities "bitmap-format-base-type", "bitmap-RFC8724",
  "bitmap-compound-ack"; typedef "bitmap-format-type". State that the
  "ietf-schc-compound-ack" module of RFC 9441 is merged into "ietf-schc".

## 6. Rule Definition (RFC 9363 Section 5)

* Add the fourth nature, "nature-management" (bis: Rule Management): a
  management Rule uses the compression structure, and only the
  "entry-universal" list.
* Section 5.1 (Compression Rule): describe "entry-universal", keyed by
  "entry-index", the "field-or-space" choice and "field-length-value". The
  "entry" list and the "compression" case are kept, "status deprecated". The
  checks on the values (MO ignore without Target Value, MSB with argument) are
  unchanged.
* Section 5.3 (YANG Tree): replace the tree with the tree of
  ietf-schc@2026-10-27, including the "context" container and the
  "duplicate-rule" RPC.

## 7. New Section: Rule Management

bis: Rule Management. Content to add:

* "management" feature.
* "nature-management" nature; the "entry-universal" list accepts
  "nature-compression" and "nature-management".
* "context" container (if-feature management) with the "guard-period" timer
  ("management-content" grouping), which applies to the whole context.
* "rule-status" leaf (if-feature management, default "status-valid"), of type
  "status-type", with the identities "status-valid", "status-candidate",
  "status-invalid" and "status-free". The procedures are defined in
  draft-toutain-schc-coreconf-management: refer to it rather than copying
  them.
* "duplicate-rule" RPC: "from" and "to" Rule IDs, optional
  "ipatch-sequence", "status" enumeration as output.

## 8. YANG Data Model (RFC 9363 Section 6)

* Replace the module with ietf-schc@2026-10-27.
* Replace the intermediate revision statements of the working copy
  (2025-11-24 to 2026-10-27, never published) with a single revision
  statement for the new RFC, describing all the changes, followed by the
  2023-03-01 revision of RFC 9363.
* Fix "fid-oscore-base-type" (see 5.3).
* Update the module description (RFC number, new features).

## 9. IANA Considerations (RFC 9363 Section 7)

* Update the "YANG Module Names" registration of "ietf-schc" to reference the
  new RFC; the namespace URI is unchanged.
* SID file: register the SID file of the new revision in the YANG-SID Modules
  registry, in the range 2550-2949 allocated to "ietf-schc" (bis: Deprecating
  the Per-Option CoAP FIDs, Manual SID Allocation). No SID for the deprecated
  per-option FIDs.

## 10. Security Considerations (RFC 9363 Section 8)

* Keep the text of RFC 9363 and add:
  * the management of the Rules: only management Rules can access the SoR, the
    management traffic is protected (OSCORE, DTLS or the underlying layer),
    and the management Rules cannot be deleted (see
    draft-toutain-schc-coreconf-management);
  * the "action-proxy-ping" Action: the core answers on behalf of the Device,
    which hides its real state.
* Update the sentence "Modification may be allowed regarding the Field
  Descriptor ...": with management, an existing Rule can no longer be
  modified, only created or deleted.

## 11. Appendix A: Example (RFC 9363 Appendix A)

* Update the XML example to the "entry-universal" structure ("entry-index",
  "field-or-space") and, if possible, add the same Rules in JSON (RFC 7951)
  and in CBOR with SIDs.

## 12. New Appendix: Manual SID Allocation

bis: Manual SID Allocation. Delta encoding of the SIDs used together, separate
ranges for data and identity nodes, room for later augmentations, private SID
translation (draft-toutain-core-private-sid-translation).

## 13. Open Points of the bis to Solve Before

* OSCORE example: presence and encoding of the KID Context entry ("TO BE
  DISCUSSED" in bis: Example 1).
* KUDOS: the Nonce length is "m + 1", which "fl-length-bytes" cannot express
  (bis: OSCORE and KUDOS Suboptions).
* Manual SID allocation: the mechanism is still a TODO.
* "duplicate-rule": the description of the "status" output is "TO BE DEFINED
  MORE PRECISELY", and its "ipatch-sequence" would modify the copied rule
  before its creation.
* "rule-status": declare it "config false", since it is managed locally by
  each node (draft-toutain-schc-coreconf-management)?
* "fid-oscore-base-type": deprecate instead of remove (see 5.3).
* Security and IANA sections of the bis are still TODO.
