---
title: "Extending Normalized Forms to String-Derived Types"
abbrev: "YANG String Normalized Form"
category: std

docname: draft-fedyk-netmod-yang-normal-form-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Network Modeling"
keyword:
 - YANG
 - NETMOD
 - normalized form
 - MAC address

author:
 -
    fullname: "Don Fedyk"
    initials: "D."
    surname: "Fedyk"
    organization: LabN Consulting, L.L.C.
    email: "dfedyk@labn.net"
 -
    fullname: "Scott Mansfield"
    initials: "S."
    surname: "Mansfield"
    organization: Ericsson
    email: "scott.mansfield@ericsson.com"

normative:
  RFC6991:
  RFC7950:
  RFC9911:

informative:
  IEEE-802-1Qcw:
    target: https://doi.org/10.1109/IEEESTD.2023.10317806
    title: "IEEE Standard for Local and Metropolitan Area Networks--Bridges and Bridged Networks Amendment 36: YANG Data Models for Scheduled Traffic, Frame Preemption, and Per-Stream Filtering and Policing"
    author:
      - org: IEEE
    date: 2023-11-17
    seriesinfo:
      IEEE: "802.1Qcw-2023"


--- abstract

YANG models frequently define identifiers using string or string-derived
types whose lexical space permits multiple representations of the same
underlying value.  This can lead to incorrect behavior when semantically
equivalent values are compared lexically.

This document adds an optional extension to the existing YANG concept of
normalized form to string-derived types whose lexical space permits
multiple representations of the same underlying value.


--- middle

# Introduction

YANG {{RFC7950}} treats values of type `string` as lexically distinct;
equality and uniqueness are therefore determined by exact string
comparison.

YANG defines normalized representations for many built-in data types.
Canonical or normalized forms provide a unique representation of a value
independent of how it may have been entered or encoded.

For string-derived types, YANG currently provides no mechanism to define a
normalized form distinct from the lexical representation.  As a result,
semantically equivalent values that have multiple valid lexical
representations may not compare equal and may not be detected as
duplicates.

This issue has been observed in both IETF and IEEE YANG modules, leading to
interoperability problems and incorrect duplicate detection.  Existing YANG
typedefs such as `mac-address` ({{RFC6991}}, {{RFC9911}}) define syntax but
do not define normalized comparison semantics.

A prominent example is the representation of MAC addresses in YANG.  IETF
and IEEE modules define different lexical forms for MAC addresses, and
equivalent values may not compare equal when represented using different
valid formats.  This problem is described in {{mac-address-background}}.

While MAC addresses provide a clear motivating example, the underlying
issue is more general.  YANG lacks a mechanism for schema authors to define
normalized forms for string-derived types whose lexical space permits
multiple representations of the same underlying value.

The resulting normalized form is used for equality-based operations.  The
normalized form is an abstract comparison value and is not a new lexical
representation.  This extension does not alter the lexical space or encoding
requirements of the underlying YANG type.

This mechanism is intentionally non-invasive.

This extension defines a normalization procedure for determining semantic
equivalence between values.  Unless explicitly defined by a future
extension, it does not alter the relational comparison semantics of the
underlying YANG type.  Operations based on equality (such as equality tests
and uniqueness evaluation) operate on the normalized form.  Relational
operators (&lt;, &lt;=, &gt;, &gt;=) continue to use the semantics of the
underlying type.

Existing YANG modules remain valid and unchanged.  The extension is applied
only where normalized forms are needed and has no effect on implementations
that do not support it.


# Terminology

{::boilerplate bcp14-tagged}


# Problem Statement

When string-derived types permit multiple lexical representations of the
same value, the following issues can arise:

* Semantically equivalent values may not compare equal.
* Duplicate entries may not be detected in keyed lists.
* Leaf-list uniqueness constraints may be violated.
* XPath equality comparisons may yield incorrect results.

Pattern restrictions alone do not solve this problem, because they validate
syntax but do not affect comparison semantics.


# Design Goals

The solution defined in this document:

* MUST NOT alter the lexical space or encoding requirements of the underlying type.
* MUST provide deterministic comparison semantics.
* MUST apply to equality and uniqueness operations.
* MUST be minimally invasive to existing YANG modules.
* MUST be opt-in and backward compatible.
* MUST be general across string-derived types.
* SHOULD be simple to implement.
* SHOULD avoid complex transformation languages.


# Proposed Solution

## Overview

This document defines a YANG extension that allows a string-derived type to
specify a normalization algorithm.

The resulting normalized form is used for equality-based operations,
including equality comparisons, list-key identity, leaf-list uniqueness, and
evaluation of the YANG `unique` statement.  The normalized representation is
not required to appear in an encoding or retrieval operation.

This extension does not modify the lexical space of an existing YANG type
and does not require changes to existing typedef definitions.  The extension
may be attached to existing types, derived types, or schema nodes where
normalized form behavior is desired.  Implementations that do not understand
the extension continue to process the lexical type normally.
Implementations that advertise support for the extension apply the specified
normalization algorithm for equality and uniqueness operations.

Use of this extension does not, by itself, change the behavior of
implementations that do not support it.

A leafref whose target type has a normalized form MUST use that normalized
form when determining whether its value identifies an existing target
instance.  When evaluating an instance-identifier, predicate values
corresponding to list keys or leaf-list values having a normalized form MUST
be validated according to the corresponding schema node's type and compared
using normalized equality.  The normalized representation is not required to
appear in the instance-identifier lexical value.

For NETCONF insertion before or after an existing user-ordered list or
leaf-list entry, values carried in `yang:key` or `yang:value` MUST conform
to the lexical space of the corresponding schema node.  When the server
resolves the target entry, it MUST use normalized equality if the
corresponding schema node has a normalized form.  The normalized
representation itself is not required to appear in `yang:key` or
`yang:value`.

When a NETCONF subtree-filter content-match node corresponds to a schema
node having a normalized form, the filter value MUST first be valid
according to the target node's YANG type.  The server MUST compare the
filter value and datastore value using their normalized forms.
Normalization does not expand the lexical space accepted by the type, and a
retrieved value continues to use the encoding rules of the underlying type.

## YANG Extension Definition

~~~ yang
module ietf-yang-normalized-form {
  yang-version 1.1;
  namespace
    "urn:ietf:params:xml:ns:yang:ietf-yang:normalized-form";
  prefix iynf;
  organization
    "IETF NETMOD Working Group";

  contact
    "WG Web: <https://datatracker.ietf.org/wg/netmod/>
     WG List: <mailto:netmod@ietf.org>";

  description
    "Defines an extension for declaring a normalized
     form for string-derived types.";
  revision 2026-07-01 {
    description "Initial revision.";
    reference "TBD: This document.";
  }
  identity normalized-form {
    description
      "Base identity for normalized forms.";
  }
  identity mac-48 {
    base normalized-form;
    description
      "Normalized form for a 48-bit MAC address.

       The input value MUST first be valid according to the
       lexical space of the type to which this normalized form
       is applied. All separators are removed and hexadecimal
       digits 'a' through 'f' are converted to uppercase. The
       resulting normalized form consists of 12 uppercase
       hexadecimal digits.";
  }
  extension normalized-form {
    argument form;
    description
      "Specifies a deterministic normalization form
       used to derive the normalized form of a value.";
  }
}
~~~

## Semantics

If a type includes the `normalized-form` extension:

1. A normalized form MUST be derived using the specified identity.
2. Equality comparisons, including `=` and `!=`, MUST use the normalized
   form.
3. List-key identity and list-key uniqueness MUST be evaluated using the
   normalized form.
4. Leaf-list uniqueness MUST be enforced using the normalized form.
5. Evaluation of the YANG `unique` statement MUST use the normalized form
   for each referenced descendant schema node that has a normalized form.
6. A `leafref` whose target type has a normalized form MUST use that
   normalized form when determining whether its value identifies an existing
   target instance.
7. When evaluating an `instance-identifier`, predicates that identify
   list-key or leaf-list values having a normalized form MUST use normalized
   equality after validating the predicate value against the corresponding
   schema node's lexical space.
8. Relational operators (&lt;, &lt;=, &gt;, &gt;=) are not changed by this
   extension and continue to use the semantics of the underlying YANG type.
9. The lexical representation MUST NOT be modified solely to satisfy this
   extension.

## Normalized identity

Normalized forms are identified by identity and are associated with a
deterministic algorithm.

### mac-48

The `mac-48` identity applies to 48-bit MAC addresses represented as six
hexadecimal octets.  The lexical representation is defined by the type that
uses the extension.

The normalization procedure is:

1. Validate the input against the lexical space of the type to which the
   normalized form is applied.
2. Remove the separator characters permitted by the lexical representation
   of the underlying type.
3. Convert hexadecimal digits to uppercase.
4. Use the resulting 12 hexadecimal digits as the normalized form of the MAC
   address.

Equivalent inputs include:

~~~
aa:bb:cc:dd:ee:ff
AA:BB:CC:DD:EE:FF
aa-bb-cc-dd-ee-ff
AA-BB-CC-DD-EE-FF
~~~

These values yield the same normalized form:

~~~
AABBCCDDEEFF
~~~


# Design Rationale

A normalized-form identity names a normalization algorithm rather than a
regular-expression pattern.  Pattern restrictions can constrain lexical
syntax, but they cannot in general express normalization procedures such as
case folding, separator removal, IPv6 textual normalization, or other
procedural transformations.  The identity therefore provides a stable name
for a fully specified normalization procedure.  A pattern may be useful as
part of the description of a simple normalization, but it is not sufficient
as the general mechanism defined by this document.


# Example Usage

The definition should be in a top level YANG module.  While new types with
the normalized form could be created it is also valid to just modify in
place IEEE mac-address to support the normalized form.


# IEEE MAC Address example

The following example show that the normalized form can be added to any
definition.  Below is an IEEE example.

~~~ yang
      leaf address {
        type ieee:mac-address;
        iynf:normalized-form "iynf:mac-48";
        mandatory true;
        description
          "A sample IEEE MAC address format.";
      }
~~~


# IETF MAC Address example

The following example show that the normalized form can be added to any
definition.  Below is an IETF example.

~~~ yang
      leaf address {
        type ietf:mac-address;
        iynf:normalized-form "iynf:mac-48";
        mandatory true;
        description
          "A sample IETF MAC address format.";
      }
~~~


# Comparison Example

The following values use different lexical representations but identify the
same underlying 48-bit MAC address:

~~~
aa:bb:cc:dd:ee:ff
AA-BB-CC-DD-EE-FF
~~~

Because both types declare the same `mac-48` normalized form, both values
yield the same normalized form:

~~~
   AABBCCDDEEFF
~~~

Implementations that support this extension compare the values using the
normalized form for equality and uniqueness operations, while preserving
each type's lexical requirements.


# List Key Example

~~~ yang
list fdb-entry {
    key "mac vlan";
    leaf mac {
      type mac-address;
      iynf:normalized-form "iynf:mac-48";
    }
    leaf vlan {
        type uint16;
    }
    leaf port {
        type string;
    }
}
~~~

A list key using the IETF typedef continues to accept only the IETF
colon-separated lexical form.  A corresponding IEEE model can use the IEEE
typedef and preserve the IEEE dash-separated lexical form.  In both cases,
the shared `mac-48` normalized form enables consistent equality and
duplicate detection across representations.


# Backward Compatibility

Existing YANG models are unaffected unless the extension is used.  The
lexical space and encoding requirements of the underlying type are
unchanged.  Behavior changes only for operations whose semantics depend on
equality or identity.


# Applicability

While motivated by MAC addresses, this mechanism can apply to any
string-derived type with multiple equivalent representations, including
case-insensitive identifiers, formatted identifiers, and normalized
encodings.


# Security Considerations

Normalization reduces ambiguity and helps prevent duplicate or conflicting
configuration entries.

Implementations MUST ensure that normalization algorithms are deterministic
and unambiguous.


# IANA Considerations

This document has no IANA actions.


--- back

# MAC Address Representation Background
{: #mac-address-background}

## Background

MAC Address Formats in the IETF and IEEE YANG modules are different.

## Current State
{: #current-state}

The IETF and IEEE YANG modules define MAC-address types derived from the
YANG string type, but use different lexical patterns and canonical
representations.  The issue is that the IETF and IEEE use different patterns
and have different canonical forms, which leads to a situation where
equivalent MAC Addresses will not match.

Both organizations have a long history of supporting these formats.

This appendix is meant to document the issue, and support the solution
proposed in this document.

For example, the following MAC Address is in IETF Canonical Format

~~~
   90:10:00:01:02:aa
~~~

For example, the following MAC Address is in IEEE Canonical Format

~~~
   90-10-00-01-02-AA
~~~

The MAC addresses are equivalent, but will not match if used in an XPath, or
as a key, or any string comparison.

There are several potential trouble spots in published IETF YANG modules.

## Detail
{: #detail}

### IETF Format
{: #ietf-format}

The IETF Format (from ietf-yang-types@2013-07-15.yang) {{RFC9911}} used in
the mac-address typedef is found below.

~~~
typedef mac-address {
 type string {
   pattern '[0-9a-fA-F]{2}(:[0-9a-fA-F]{2}){5}';
 }
 description
  "The mac-address type represents an IEEE 802 MAC address.
   The canonical representation uses lowercase characters.

   In the value set and its semantics, this type is equivalent
   to the MacAddress textual convention of the SMIv2.";
 reference
  "IEEE 802: IEEE Standard for Local and Metropolitan Area
             Networks: Overview and Architecture
   RFC 2579: Textual Conventions for SMIv2";
}
~~~

### IEEE Format
{: #ieee-format}

The IEEE Format used in the mac-address typedef {{IEEE-802-1Qcw}} is found
below.

~~~
typedef mac-address {
 type string {
   pattern "[0-9a-fA-F]{2}(-[0-9a-fA-F]{2}){5}";
 }
 description
   "The mac-address type represents a MAC address in the canonical
    format and hexadecimal format specified by IEEE Std 802. The
    hexadecimal representation uses uppercase characters.";
 reference
   "3.1, 8.1 of IEEE Std 802";
}
~~~

## Why Normalized Form Helps
{: #rational}

Normal form allows these definitions to remain.  The YANG extension
described in this document enables comparison of MAC addresses regardless of
the format.


# Acknowledgments
{:numbered="false"}

TODO acknowledge.
