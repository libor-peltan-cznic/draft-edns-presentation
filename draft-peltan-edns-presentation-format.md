---
title: EDNS Presentation Format
abbrev: edns-presentation-format
docname: @DOCNAME@
date: {DATE}

category: std
ipr: trust200902
keyword: Internet-Draft
stand_alone: yes
submissionType: IETF

author:
 -
    ins: L. Peltan
    name: Libor Peltan
    org: CZ.NIC
    email: libor.peltan@nic.cz
 -
    ins: T. Carpay
    name: Tom Carpay
    org: NLnet Labs
    email: tom@nlnetlabs.nl

--- abstract

This document describes textual and JSON representation format of EDNS option.
It also modifies the escaping rules of JSON representation of DNS messages, previously defined in {{!RFC8427}}.

--- middle

# Introduction

DNS records{{!RFC1035}} of any type can be converted between their binary Wire format, used in DNS messages transferred over the Internet, and textual Presentation format, used not only in Zone Files (called "master files" in the referenced document), but also to display the contents of DNS messages to humans by debugging utilities, and possible other use-cases.
The Presentation format can be however processed also programatically and also converted back to Wire Format unambiguously.

The EDNS{{!RFC6891}} option pseudorecord does not appear in Zone Files, but it sometimes needs to be converted to human-readable, but also machine-readable textual representation.
This document describes such a Presentation Format of OPT pseudorecord.
It is advised to use this when displaying an OPT pseudorecord to humans.
It is recommended to use this when the textual format is expected to be machine-processed further.

The JSON{{!RFC8259}} representation{{!RFC8427}} of DNS messages is also helpful as both human-readable and machine-readable format (despite the limitation in non-preservation of the order of options, which prevents reversing the conversion unambiguosly), but it did not define JSON representation of EDNS option pseudorecord.
This document defines it.

The aforementioned document{{!RFC8427}} also defined ambiguous and possibly conflicting rules for escaping special characters when representing DNS names in JSON.
This documents modifies and clarifies those rules.

# Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**",
"**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**",
"**RECOMMENDED**", "**NOT RECOMMENDED**", "**MAY**", and
"**OPTIONAL**" in this document are to be interpreted as described in
BCP 14 {{!RFC2119}}{{!RFC8174}} when, and only when, they appear in all
capitals, as shown here.

* EDNS(0) signifies EDNS version 0.

* "Decimal value" means an integer displayed in decimals with no leading zeroes.

* Base16 is the representation of arbitrary binary data by an even number of case-insensitive hexadecimal digits ({{!RFC4648, Section 8}}).

* "Followed by" in terms of strings denotes their concatenation, with no other characters nor space between them.

* Backslash is the character called also Reverse Solidus, ASCII code 0x5c.

* Zero-octet is an octet with all bits set to 0, i.e. ASCII code 0x00.


# Version-independent Presentation Format {#independent}

EDNS versions other than 0 are not yet specified, but an OPT pseudorecord with version field set to value other than zero might in theory appear in DNS messages.
This section specifies how to convert such OPT pseudorecord to Presentation format.
This procedure SHOULD NOT be used for EDNS(0).

OPT pseudorecord is in this case represented the same way as a RR of unknown type according to 
({{!RFC3597, Section 5}}).
In specific:

* Owner name is `.` (DNS Root Domain Name).

* TTL is Decimal value of the 32-bit big-endian integer appearing at the TTL position of OPT pseudorecord Wire format, see ({{!RFC6891, Section 6.1.3}}).

* CLASS is a text representation of the 16-bit integer at the CLASS position of OPT pseudorecord Wire format (UDP payload size happens to appear there).
This will usually result in `CLASS####` (where #### will be the Decimal value), but it might also result for example in `IN` or `CH` if the value is 1 or 4, respectively.

* TYPE is either `TYPE41` or `OPT`.

* RDATA is formatted by `\#`, its legth as Decimal value, and data as Base16 as per ({{!RFC3597, Section 5}}).

Example: `. 16859136 CLASS1232 TYPE41 \# 6 000F00020015`

# EDNS(0) Presentation Format

The EDNS(0) Presentation Format follows RR format of the master file ({{!RFC1035, Section 5.1}}), including quotation of non-printable characters, multi-line format using round brackets, and semicolons denoting comments.

Depending on use-case, implementations MAY choose to display only RDATA.
In the case the resource-record-like Presentation format is desired, the following applies:

* Owner Name MUST be `.` (DNS Root Domain Name).

* TTL MAY be omitted.
If it is present, it MUST be `0` (zero).
Note that this differs from DNS RR wire-to-text conversion, as well as Version-independent format in (#independent).

* CLASS MAY be omitted.
If it is present, it MUST be `ANY`.

* TYPE MUST be `EDNS0`.

RDATA consists of at least three &lt;character-string&gt;s ({{!RFC1035, Section 5.1}}), one for each field.
Each field consists of a Field-name, followed by an equal sign (`=`), followed by a Field-value.
If the Field-value is empty or omitted, the equal sign MUST be omitted as well.
For each field, the Field-name and the Field-value are defined by this document, or by the specification of the respective EDNS Option.
If it is not, a generic Field-name and Field-value from (#unrecognized) applies.
However, those generics MAY be used for any Option at all times.

The first three fields, (#flags), (#extrcode), and (#udpsize) MUST always be present.
The rest of the fields are based on Options in the OPT record {{!RFC6891, Section 6.1.2}}.
They MUST be presented in the same order as they appear in wire format.
It is recommended to use the multi-line format with comments at each field, together with a more human-readable form of the contents of each option when available.
See examples of this in (#examples).

## Flags {#flags}

The first field's Field-name is `FLAGS` and its Field-value is `0` (zero) if the EDNS flags is zero.

Otherwise, the Field-value consists of comma-separated list of the items `BIT##`, where `##` is a Decimal value.
`BITn` is present in the list if and only if `n`-th bit (the most significant bit being `0`-th) of flags is set to `1`.
If the Flag of the bit is specified in (https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-13), the Flag SHOULD be used insted of `BIT##`.
(So far, the only known Flag is `DO`.)

Examples: `FLAGS=0`; `FLAGS=DO`; `FLAGS=DO,BIT1`; `FLAGS=BIT3,BIT7,BIT15`.

## Extended RCODE {#extrcode}

The second field's Field-name is `RCODE` and its Field-value is `RCODE###`, where `###` stands for the DNS message extended RCODE as Decimal value, computed from both the OPT record and the DNS Message Header.
If the lower four bits of extended RCODE in DNS Message Header can not be used, the Field-value is `UNKNOWNRCODE###`, where `###` stands for the DNS message extended RCODE as Decimal value, with the lower four bits set to zero (i.e. the four-bit left shift still applies).
If the extended RCODE has been computed completely and it is listed in (https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-6), its Name should be used insted of `RCODE###`.
The Name is case-insensitive.

Examples: `RCODE=NXDOMAIN`; `RCODE=BADSIG`; `RCODE=RCODE3841`; `RCODE=UNKNOWNRCODE3840`.

## UDP Payload Size {#udpsize}

The third field's Field-name is `UDPSIZE` and its Field-value is the UDP payload size as Decimal value.

## Unrecognized Option {#unrecognized}

EDNS options that are not part of this specification and their own specifications do not specify their Field-name and Field-value MUST be displayed according this subsection.
Other options (specified below or otherwise) MAY be displayed so as well.

Unrecognized option Field-name is `OPT##`, where `##` stands for its OPTION-CODE, and Field-value is its OPTION-VALUE displayed as Base16.

## LLQ Option

TODO

## NSID Option

The NSID (OPTION-CODE 3 {{!RFC5001}}) Field-name is `NSID` and Field-value is its OPTION-VALUE displayed as Base16.

It is recommended to add a comment with ASCII representation of the value.

## DAU, DHU and N3U Options

The DAU, DHU, and N3U (OPTION-CODES 5, 6, 7, respectively {{!RFC6975}}) Field-names are `DAU`, `DHU`, and `N3U`, respectively, and their Field-values consist of comma-separated lists of ALG-CODEs as Decimal values.

Example: `DAU=15`; TODO some realistic example

## Edns-Client-Subnet Option

The EDNS Client Subnet (OPTION-CODE 8 {{!RFC7871}}) Field-name is `ECS` and if FAMILY is neither IPv4 (`1`) nor IPv6 (`2`), its Field-value is the whole OPTION-VALUE as Base16.
Otherwise, it consists of the textual IPv4 or IPv6 address (TODO ref, {{!RFC2373, Section 2.2}}), followed by a slash (`/`), followed by SOURCE PREFIX-LENGTH as Decimal value, followed by another slash, followed by SCOPE PREFIX-LENGTH as Decimal value.
If SCOPE PREFIX-LENGTH is zero, it MUST be omitted together with the second slash.

Examples: `ECS=1.2.3.4/24`; `ECS=1234::2/56/48`; `ECS=000520000102030405060708`.

## EDNS EXPIRE Option

The EDNS EXPIRE (OPTION-CODE 9 {{!RFC7314}}) Field-name is `EXPIRE` and its Field-value, if present, is displayed as Decimal value.

## Cookie Option

The DNS Cookie (OPTION-CODE 10 {{!RFC7873}}) Field-name is `COOKIE` and its Field-value consists of the Client Cookie as Base16, followed by a comma, followed by the Server Cookie as Base16.
The comma and Server Cookie is displayed only if OPTION-LENGTH is greater than 8.

## Edns-Tcp-Keepalive Option

The edns-tcp-keepalive (OPTION-CODE 11 {{!RFC7828}}) Field-name is `KEEPALIVE` and its Field-value is displayed as Decimal value.

## Padding Option

The Padding (OPTION-CODE 12 {{!RFC7830}}) Field-name is `PADDING` and its Field-value is its OPTION-VALUE displayed as Base16.
If the OPTION-VALUE consists only of zero-octets, it SHOULD be substituted with an alternative Field-value `[###]`, where `###` stands for OPTION-LENGTH as Decimal value.

## CHAIN Option

The CHAIN (OPTION-CODE 13 {{!RFC7901}}) Field-name is `CHAIN` and its Field-value is displayed as a textual Fully-Qualified Domain Name.

## Edns-Key-Tag Option

The edns-key-tag (OPTION-CODE 14 {{!RFC8145, Section 4}}) Field-name is `KEYTAG` and its Field-value is displayed as a comma-separated list of Decimal values.

## Extended DNS Error Option

The Extended DNS Error (OPTION-CODE 15 {{!RFC8914}}) Field-name is `EDE` and the Field-value is its INFO-CODE as Decimal value.
It is recommended to add a comment with Purpose of the given code ({{!RFC8914}}, seciton 5.2).

If the EXTRA-TEXT is nonempty, it MUST be displayed as another field, with Field-name `EDETXT` and Field-value being the EXTRA-TEXT string as-is.

Note that RFC1035-style escaping applies to all non-printable and non-ASCII caracters, including some eventual UTF-8 bi-characters and possible trailing zero-octet.
Also note that any presence of spaces requires the whole &lt;character-string&gt; to be enclosed in quotes, not just the Field-value.

Examples: `EDE=18; Prohibited`; `EDE=6; DNSSEC_BOGUS; "EDETXT=signature too short"`

# Examples of EDNS(0) Presentation Format {#examples}

The following example shall illustrate the features of EDNS(0) Presentation format described above.
It may not make really sense and should not appear in normal DNS operation.

```
. 0 IN EDNS0 (
    FLAGS=DO
    RCODE=BADCOOKIE
    UDPSIZE=1232
    COOKIE=36714f2e8805a93d,4654b4ed3279001b
    EDE=18; Prohibited
    "EDETXT=bad cookie\000"
    OPT1234=000004d2
    PADDING=[113]
    )
```

TODO

# Update Representing DNS Messages in JSON {#jsonescaping}

This section is not related to EDNS.
This section updates {{!RFC8427, Section 2.6}}, including erratum 5439, which introduces contradicting MUSTs for escaping of backslashes.

In order to represent a DNS name in JSON, it MUST be first converted to textual Presentation format according to ({{!RFC1035, Section 5.1}}) (called master file format in the referenced document), and the resulting &lt;character-string&gt; subsequently inserted into JSON as String.

In other words, in the first step, every problematic character (non-printable, backslash, dot within Label, or any octet) is substituted with the sequence `\DDD`, where `DDD` is the three-digit decimal ASCII code, or, in some cases (backslash, dot, any printable character), alternatively just prepended with a backslash; in the second step, every quote (`"`) and backslash (`\`) in the resulting &lt;character-string&gt; is prepended with another backslash.
As a consequence, the JSON escaping sequence `\uXXXX` (where `XXXX` is a hexadecimal Unicode code) is never needed.

The name MUST be represented as an absolute Fully-Qualified Domain Name.
Internationalized Domain Name (IDN) labels MUST be expressed in their A-label form, as described in {{!RFC5890}}.

Example: the name with the Wire format `04005C2E2203646F6D00` can be represented in JSON as `"NAME": "\\000\\\\\\046\".com."`, but also as (among other ways) `"NAME": "\\000\\092\\.\\\".c\\om."`.

# Version-independent JSON representation {#jindependent}

EDNS versions other than 0 are not yet specified, but an OPT pseudorecord with version field set to value other than zero might in theory appear in DNS messages.
This section specifies how to represent such OPT pseudorecord in JSON.
This procedure SHOULD NOT be used for EDNS(0).

TODO!
Ideas?

# EDNS(0) Representation in JSON

EDNS(0) OPT record can be represented in JSON as an object called EDNS0.
It MUST contain the three members (name/value pairs), (#jflags), (#jextrcode), and (#judpsize).
The rest of the members are based on Options in the OPT record {{!RFC6891, Section 6.1.2}}.
For each member, its name and value are defined by this document, or by the specification of the respective EDNS Option.
If it is not, a generic name and value from (#junrecognized) applies.
However, those generics MAY be used for any Option at all times.
Note that the order of members is not preserved in JSON.

## Flags {#jflags}

The JSON member name is `FLAGS` and its value is an Array of Strings `BIT##`, where `##` is a Decimal value.
`BITn` is present in the Array if and only if `n`-th bit (the most significant bit being `0`-th) of flags is set to `1`.
If the Flag of the bit is specified in (https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-13), the Flag SHOULD be used insted of `BIT##`.
(So far, the only known Flag is `DO`.)

## Extended RCODE {#jextrcode}

The JSON member name is `RCODE` and its value is a String containing Field-value from (#extrcode).

## UDP Payload Size {#judpsize}

The JSON member name is `UDPSIZE` and its value is an Integer with UDP payload size.

## Unrecognized Option {#junrecognized}

EDNS options that are not part of this specification and their own specifications do not specify their JSON member name and value MUST be displayed according this subsection.
Other options (specified below or otherwise) MAY be displayed so as well.

Unrecognized option JSON member name is `OPT##`, where `##` stands for its OPTION-CODE as Decimal value, and its value is a String containing its OPTION-VALUE encoded as Base16.

## LLQ Option

TODO

## NSID Option

The NSID (OPTION-CODE 3 {{!RFC5001}}) JSON member name is `NSID` and its value is a String containing its OPTION-VALUE encoded as Base16.
TODO make it two-field object with both Base16 and unicode.

## DAU, DHU and N3U Options

The DAU, DHU, and N3U (OPTION-CODES 5, 6, 7, respectively {{!RFC6975}}) JSON member names are `DAU`, `DHU`, and `N3U`, respectively, and their values are Arrays of Integers with ALG-CODEs.

Example: TODO

## Edns-Client-Subnet Option

The EDNS Client Subnet (OPTION-CODE 8 {{!RFC7871}}) JSON member name is `ECS` and its value is an Object with following members:

* `FAMILY` - Integer with FAMILY
* `IP` - String with textual IPv4 or IPv6 address (TODO ref, {{!RFC2373, Section 2.2}}), or a String with ADDRESS encoded as Base16 if FAMILY is neither `1` or `2`
* `SOURCE` - Integer with SOURCE PREFIX-LENGTH
* `SCOPE` - Integer with SCOPE PREFIX-LENGTH, omitted if zero

Examples: TODO

## EDNS EXPIRE Option

The EDNS EXPIRE (OPTION-CODE 9 {{!RFC7314}}) JSON member name is `EXPIRE` and its value is either an Integer or `null`.

## Cookie Option

The DNS Cookie (OPTION-CODE 10 {{!RFC7873}}) JSON member name is `COOKIE` and its value is an Array containing a String with the Client Cookie encoded as Base16 and, if present, another String with Server Cookie encoded as Base16.

## Edns-Tcp-Keepalive Option

The edns-tcp-keepalive (OPTION-CODE 11 {{!RFC7828}}) JSON member name is `KEEPALIVE` and its value is an Integer.

## Padding Option

The Padding (OPTION-CODE 12 {{!RFC7830}}) JSON member name is `PADDING` and its value is a String with its OPTION-VALUE encoded as Base16.
If the OPTION-VALUE consists only of zero-octets, it SHOULD be substituted with an alternative value consisting of an Array containing a single Integer denoting the OPTION-LENGTH.

## CHAIN Option

The CHAIN (OPTION-CODE 13 {{!RFC7901}}) JSON member name is `CHAIN` and its value is a String with the OPTION-VALUE in the form of a textual Fully-Qualified Domain Name.
See (#jsonescaping) for representing DNS names in JSON.

## Edns-Key-Tag Option

The edns-key-tag (OPTION-CODE 14 {{!RFC8145, Section 4}}) JSON member name is `KEYTAG` and its value is an Array of Integers.

## Extended DNS Error Option

The Extended DNS Error (OPTION-CODE 15 {{!RFC8914}}) JSON member name is `EDE` and its value is an Object with following members:

* `INFO-CODE` - Integer with the INFO-CODE
* `Purpose` - String with Purpose of the INFO-CODE ({{!RFC8914}}, seciton 5.2)
* `EXTRA-TEXT` - String with the EXTRA-TEXT

The EXTRA-TEXT member MUST be omitted if empty.
If its value contains non-printable or special (backslash, quote) characters, they MUST be escaped by the means of JSON Strings ({{!RFC8259, Section 7}}).

# Examples of EDNS(0) Representation in Json {#jexamples}

The following example is the JSON representation of the first example in (#examples).

```
"EDNS0": {
    "FLAGS": [ "DO" ],
    "RCODE": "BADCOOKIE",
    "UDPSIZE": 1232,
    "COOKIE": [ "36714f2e8805a93d", "4654b4ed3279001b" ],
    "EDE": {
        "INFO-CODE": 18,
        "Purpose": "Prohibited",
        "EXTRA-TEXT": "bad cookie\u0000"
    },
    "OPT1234": "000004d2",
    "PADDING": [ 113 ]
}
```

TODO, equivalent to (#examples).

# Security Considerations {#security}

None.

# Acknowledgements

TODO

# Implementation Status

**Note to the RFC Editor**: please remove this entire appendix before publication.

None yet.

# Change History

**Note to the RFC Editor**: please remove this entire appendix before publication.

* edns-presentation-format-00

> Initial public draft.


--- back