---
title: EDNS Presentation and JSON Format
abbrev: edns-presentation-and-json-format
docname: @DOCNAME@
date: {DATE}

category: std
ipr: trust200902
keyword: Internet-Draft
stand_alone: yes
submissionType: IETF
updates: 8427

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
    email: tomcarpay@gmail.com

informative:
  IANA.RCODEs:
    target: https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-6
    title: DNS RCODEs

  IANA.EDNS.EDE:
    target: https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#extended-dns-error-codes
    title: EDNS Extended Error Codes

  IANA.EDNS.DAU:
    target: https://www.iana.org/assignments/dns-sec-alg-numbers/dns-sec-alg-numbers.xhtml
    title: DNS Security Algorithm Numbers

  IANA.EDNS.DHU:
    target: https://www.iana.org/assignments/ds-rr-types/ds-rr-types.xhtml#ds-rr-types-1
    title: DNSSEC DS RR Type Digest Algorithms

  IANA.EDNS.N3U:
    target: https://www.iana.org/assignments/dnssec-nsec3-parameters/dnssec-nsec3-parameters.xhtml#dnssec-nsec3-parameters-3
    title: DNSSEC NSEC3 Hash Algorithms

--- abstract

This document describes the textual and JSON representation formats of EDNS options.
It also modifies the escaping rules of the JSON representation of DNS messages, previously defined in RFC8427.

--- middle

# Introduction

A DNS record{{!RFC1035}} of any type can be converted between its binary Wire format and textual Presentation format. The Wire format is used in DNS messages transferred over the Internet, while the Presentation format is used not only in Zone Files (called "master files" in the referenced document), but also to display the contents of DNS messages to humans by debugging utilities and possible other use-cases.

The Presentation format can, however, be processed programatically and also converted back to Wire Format unambiguously.

The EDNS{{!RFC6891}} option pseudo-record does not appear in Zone Files, but it sometimes needs to be converted to human-readable or even machine-readable textual representation.
This document describes such a Presentation Format of the OPT pseudo-record.
It is advised to use this when displaying an OPT pseudo-record to humans.
It is recommended to use this when the textual format is expected to be machine-processed further.

The JSON{{!RFC8259}} representation{{!RFC8427}} of DNS messages is also helpful as both human-readable and machine-readable format (despite the limitation in non-preservation of the order of options, which prevents reversing the conversion unambiguously), but it did not define a JSON representation of EDNS option pseudo-record.
This document defines it.

The aforementioned document{{!RFC8427}} also defined ambiguous and possibly conflicting rules for escaping special characters when representing DNS names in JSON.
This document modifies and clarifies those rules.

# Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**",
"**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**",
"**RECOMMENDED**", "**NOT RECOMMENDED**", "**MAY**", and
"**OPTIONAL**" in this document is to be interpreted as described in
BCP 14 {{!RFC2119}}{{!RFC8174}} when, and only when, they appear in all
capitals, as shown here.

* Base16 is the representation of arbitrary binary data by an even number of case-insensitive hexadecimal digits ({{!RFC4648, Section 8}}).

* Backslash is the character, also called Reverse Solidus, ASCII code 0x5c.

* ID-string is a string of characters containing only (uppercase or lowercase) letters, digits, dashes, and underscores, and its first character is a (uppercase or lowercase) letter.

* "Note" denotes a sentence that is not normative. Instead, it points out some non-obvious consequences of previous statements.

# Generic EDNS Presentation Format {#independent}

A malformed EDNS record or a record of an unsupported EDNS version can be converted to Presentation format using this generic method.
OPT pseudo-record is, in this case, represented the same way as a RR of unknown type according to {{!RFC3597, Section 5}}.
In specific:

* Owner Name is the Owner Name of the OPT record.
Note that this is usually `.` (DNS Root Domain Name) unless malformed.

* TTL is the 32-bit big-endian integer appearing at the TTL position of the OPT pseudo-record Wire format, see {{?RFC6891, Section 6.1.3}}.

* CLASS is a text representation of the 16-bit integer at the CLASS position of the OPT pseudo-record Wire format (UDP payload size happens to appear there).
This will usually result in `CLASS####` (where #### will be the integer), but it might also result, for example in `IN` or `CH` if the value is 1 or 4, respectively.

* TYPE is either `TYPE41` or `OPT`.

* RDATA is formatted by `\#`, its length as a decadic number, and data as Base16 as per {{!RFC3597, Section 5}}.

Example:

~~~
. 16859136 CLASS1232 TYPE41 \# 6 000F00020015
~~~

# Generic EDNS JSON representation {#jindependent}

A malformed EDNS record or a record of an unsupported EDNS version can be converted to JSON using this generic method.
The OPT pseudo-record is, in this case, represented in JSON as an object with following members:

* `NAME` - String with the Owner Name of the OPT record.
Note that this is usually `.` (DNS Root Domain Name) unless malformed.
See [jsonescaping](#jsonescaping) for representing DNS names in JSON.

* `TTL` - Integer with the 32-bit big-endian value appearing at the TTL position of the OPT pseudo-record Wire format, see {{?RFC6891, Section 6.1.3}}.

* `CLASS` - Integer with the 16-bit value at the CLASS position of the OPT pseudo-record Wire format (UDP payload size happens to appear there).

* `TYPE` - Integer with the value 41.

* `RDATAHEX` - String with the pseudo-record RDATA formatted as Base16.

Example:

~~~
{
    "NAME": ".",
    "TTL": 16859136,
    "CLASS": 1232,
    "TYPE": 41,
    "RDATAHEX": "000f00020015"
}
~~~

# Common Concept

Let's first divide the information contained in the EDNS record into <em>FIELD</em>s: `Version`, `FLAGS`, `RCODE`, and `UDPSIZE` <em>FIELD</em>s are based on the OPT record header, one other <em>FIELD</em> is based on every EDNS option that appears in the OPT record RDATA.
Each <em>FIELD</em> has a defined <em>FIELD-NAME</em>, which is an ID-string, and <em>FIELD-VALUE</em> of type <em>FIELD-TYPE</em>, which is one of the following:

* <em>int</em>, a non-negative integer

* <em>ID-NAME</em>, a mnemonic string denoting a numeric value defined by this document, other referenced RFC, and/or referenced IANA table; mnemonics that are not ID-strings MUST NOT be used

* <em>ID-CODE</em>, a non-negative integer prefixed with a fixed ID-string

* <em>mixed</em>, a variant type that can be any of the above-defined types

* <em>base16</em>, an even number of hexadecimal (case-insensitive) digits representing a string of arbitrary octets

* <em>list</em>, a variable-sized (possibly empty) list of values of homogenous type defined above (possibly <em>mixed</em>)

* <em>dname</em>, a Fully-Qualified Domain Name

* <em>string</em>, a string of arbitrary octets where quoting and escaping is used to represent it as ASCII string

* <em>object</em>, a defined fixed number of <em>SUBFIELD</em>s, each having its <em>FIELD-NAME</em> and <em>FIELD-TYPE</em> defined according to the rules above (nested <em>object</em>s are forbidden)

# EDNS Presentation Format

The EDNS Presentation Format follows the RR format of the master file ({{!RFC1035, Section 5.1}}), including quotation of non-printable characters, multi-line format using round brackets, and semicolons denoting comments.
However, one difference is that &lt;character-string&gt;s are not limited in size (to 255 represented octets).

Depending on the use-case, implementations MAY choose to display only RDATA.
In the event that the resource-record-like Presentation format is desired, the following applies:

* Owner Name MUST be `.` (DNS Root Domain Name).

* TTL MAY be omitted.
If it is present, it MUST be `0` (zero).
Note that this differs from DNS RR wire-to-text conversion as well as [Generic Presentation Format](#independent).

* CLASS MAY be omitted.
If it is present, it MUST be `ANY`.

* TYPE MUST be `EDNS`.

RDATA consists of &lt;character-string&gt;s, each <em>FIELD</em> is represented by at least two of them.
First represented <em>FIELD</em>s are `Version`, `FLAGS`, `RCODE`, and `UDPSIZE` in this order; however, `Version` MAY be omitted if the EDNS Version is zero.
The rest of <em>FIELD</em>s respect the EDNS options in the same order as they appear in the OPT record, including possibly repeated options.
The following paragraph defines how a single <em>FIELD</em> is represented with &lt;character-string&gt;s.

The first &lt;character-string&gt; is the <em>FIELD-NAME</em> concatenated (no spaces in between) with a colon (`:`) and SHOULD NOT be enclosed in quotes.
The rest depends on the <em>FIELD-TYPE</em>:

* <em>int</em> is represented as a decadic number with no leading zeroes

* <em>ID-NAME</em> or <em>ID-CODE</em> is represented as-is

* <em>base16</em> is represented as-is, zero-length <em>base16</em> as an empty string enclosed in quotes (`""`)

* <em>list</em> is represented as a comma-separated list of its items with no spaces; an empty list as an empty string enclosed in quotes (`""`)

* <em>dname</em> is represented according to the rules of representing Domain names in the master file ({{!RFC1035, Section 5.1}}); Internationalized Domain Name (IDN) labels MAY be expressed in their U-label form, as described in {{!RFC5890}}.

* <em>string</em> is represented as &lt;character-string&gt; according to {{!RFC1035, Section 5.1}}; and SHOULD be enclosed in quotes even when not containing any spaces

* <em>object</em> is represented by the same number of &lt;character-string&gt;s as how many <em>SUBFIELD</em>s it has; their <em>FIELD-NAME</em>s are ignored and <em>FIELD-VALUE</em>s are represented in their defined order

Note that each <em>object</em> has fixed number of &lt;character-string&gt;s, other types have one.
This is cruical for parsing, the colon plays only decorative role, strings might also end with a colon.

# EDNS Representation in JSON

The EDNS OPT record can be represented in JSON as an object called `EDNS`.
Each <em>FIELD</em> is represented as one object member (name-value pair) ,where the name is <em>FIELD-NAME</em> and the value depends on <em>FIELD-TYPE</em>:

* <em>int</em> is represented as an Integer

* <em>ID-NAME</em>, <em>ID-CODE</em> or <em>base16</em> is represented as a String

* <em>mixed</em> is represented as a String even when it happens to be <em>int</em>

* <em>list</em> is represented as a JSON Array containing its members in specified order

* <em>dname</em> is represented as a String with quotation rules in [jsonescaping](#jsonescaping)

* <em>string</em> is represented as a String according to {{!RFC8259, Section 7}}

* <em>object</em> is represented as a JSON object with each <em>SUBFIELD</em> represented as one of its member according to rules above (note that nested <em>object</em>s are forbidden)

Note that the order of members is not preserved in JSON. The <em>FIELD</em>s `FLAGS`, `RCODE`, and `UDPSIZE` MUST be represented, `Version` MAY be omitted if the EDNS Version is zero.

# Field Definitions

## Version {#version}

EDNS Version is represented by <em>FIELD-NAME</em> `Version`, its <em>FIELD-TYPE</em> is <em>int</em> and <em>FIELD-VALUE</em> is the EDNS Version.

## Flags {#eflags}

EDNS FLAGS is represented by <em>FIELD-NAME</em> `FLAGS` and its <em>FIELD-TYPE</em> is a <em>list</em> of <em>mixed</em>:

* <em>ID-NAME</em> `DO` if the DO bit is set

* <em>ID-CODE</em> `BITn` for each `n`-th bit (other than DO) set

Examples of Presentation format:

~~~
FLAGS: ""
~~~

~~~
FLAGS: DO,BIT1
~~~

~~~
FLAGS: BIT3,BIT7,BIT14
~~~

## Extended RCODE {#extrcode}

Extended RCODE is represented by <em>FIELD-NAME</em> `RCODE` and its <em>FIELD-TYPE</em> is a <em>mixed</em>.

For the sake of readability, it is RECOMMENDED to compute the whole DNS Message Extended RCODE from both the OPT record and the DNS Message Header.
If the whole DNS Message Extended RCODE is computed and has a mnemonic in {{IANA.RCODEs}}, the <em>FIELD-VALUE</em> MAY be this mnemonic as <em>ID-NAME</em>.
If the whole DNS Message Extended RCODE is computed and no mnemonic is available (or used), the <em>FIELD-VALUE</em> is an <em>int</em> with the computed Extended RCODE.
If the whole DNS Message Extended RCODE cannot be computed, the <em>FIELD-VALUE</em> is an <em>ID-CODE</em> `EXT##`, where `##` stands for DNS Message Extended RCODE with the lower four bits set to zero (i.e. the four-bit left shift still applies).

Examples of Presentation format:

~~~
RCODE: NXDOMAIN
~~~
~~~
RCODE: 3841
~~~
~~~
RCODE: EXT3840
~~~

## UDP Payload Size {#udpsize}

UDP Payload Size is represented by <em>FIELD-NAME</em> `UDPSIZE`, its <em>FIELD-TYPE</em> is <em>int</em> and <em>FIELD-VALUE</em> is the UDP Payload Size.

## Unrecognized Option {#unrecognized}

EDNS options that are not part of this specification, and their own specifications do not specify their <em>FIELD-NAME</em> and <em>FIELD-VALUE</em> MUST be displayed according to this subsection.
Other options (specified below or otherwise) MAY be displayed so as well.

Unrecognized EDNS option is represented by <em>FIELD-NAME</em> `OPT##`, where `##` stands for its OPTION-CODE, its <em>FIELD-TYPE</em> is <em>base16</em> and <em>FIELD-VALUE</em> is its OPTION-VALUE encoded as Base16.

## LLQ Option

The LLQ (OPTION-CODE 1 {{?RFC8764}}) option is represented by <em>FIELD-NAME</em> `LLQ` and its <em>FIELD-VALUE</em> is a <em>list</em> of <em>int</em>s with LLQ-VERSION, LLQ-OPCODE, LLQ-ERROR, LLQ-ID, and LLQ-LEASE in this order.

Example of Presentation format:

~~~
LLQ=1,1,0,0,3600
~~~

## NSID Option

The NSID (OPTION-CODE 3 {{?RFC5001}}) option is represented by <em>FIELD-NAME</em> `NSID` and its <em>FIELD-VALUE</em> is an object with two <em>SUBFIELD</em>s in the following order:

* first <em>FIELD-NAME</em> is `HEX` and <em>FIELD-VALUE</em> is a <em>base16</em> representation of the OPTION-VALUE

* second <em>FIELD-NAME</em> is `TEXT` and <em>FIELD-VALUE</em> is a <em>string</em> representation of the OPTION-VALUE

The `TEXT` value MAY be substituted with an empty string (for example, if the OPTION-VALUE contains non-printable characters).
Within JSON, the `TEXT` <em>SUBFIELD</em> MAY be omitted if it is an empty string.

## DAU, DHU and N3U Options {#dau}

The DAU, DHU, and N3U (OPTION-CODES 5, 6, 7, respectively {{!RFC6975}}) options are represented by <em>FIELD-NAME</em>s `DAU`, `DHU`, and `N3U`, respectively, and their `FIELD-VALUES` are <em>list</em>s of <em>int</em>s with their ALG-CODEs.

Within Presentation format, their <em>FIELD-VALUE</em>s MAY be substituted with <em>list</em>s of <em>ID-NAME</em>s with the textual mnemonics of the ALG-CODEs found in their respective IANA registries {{IANA.EDNS.DAU}}{{IANA.EDNS.DHU}}{{IANA.EDNS.N3U}}.

Examples of Presentation format:

~~~
DAU: 8,10,13,14,15
DHU: 1,2,4
N3U: 1
~~~
~~~
DAU: RSASHA256,RSASHA512,ECDSAP256SHA256,ECDSAP384SHA384,ED25519
DHU: SHA-1,SHA-256,SHA-384
N3U: SHA-1
~~~

## Edns-Client-Subnet Option

The EDNS Client Subnet (OPTION-CODE 8 {{?RFC7871}}) option is represented by <em>FIELD-NAME</em> `ECS` and its <em>FIELD-TYPE</em> is a <em>string</em>.
If FAMILY is either IPv4 (`1`) or IPv6 (`2`) and the OPTION-LENGTH matches the expected length, the <em>FIELD-VALUE</em> is a slash-separated (no spaces) tuple of:

* the textual IPv4 or IPv6 address ({{!RFC1035, Section 3.4.1}}, {{!RFC4291, Section 2.2}}), respectively

* SOURCE PREFIX-LENGTH as a decadic number

* SCOPE PREFIX-LENGTH as a decadic number, SHOULD be omitted (including the separating slash) if zero

Otherwise, the <em>FIELD-VALUE</em> is a <em>string</em> with base16-representation of the OPTION-VALUE.

Examples of Presentation format:

~~~
ECS: "1.2.3.4/24"
~~~
~~~
ECS: "1234::2/56/48"
~~~
~~~
ECS: "000520000102030405060708"
~~~

## EDNS EXPIRE Option

The EDNS EXPIRE (OPTION-CODE 9 {{?RFC7314}}) option is represented by <em>FIELD-NAME</em> `EXPIRE` and its <em>FIELD-VALUE</em> is a <em>mixed</em>:

* <em>ID-NAME</em> `NONE` if OPTION-LENGTH is zero

* <em>int</em> with EXPIRE value otherwise

## Cookie Option

The DNS Cookie (OPTION-CODE 10 {{?RFC7873}}) option is represented by <em>FIELD-NAME</em> `COOKIE` and its <em>FIELD-VALUE</em> is a <em>list</em> of <em>base16</em> with the Client Cookie and, if OPTION-LENGTH is greater than 8, the Server Cookie.

## Edns-Tcp-Keepalive Option {#keepalive}

The edns-tcp-keepalive (OPTION-CODE 11 {{?RFC7828}}) option is represented by <em>FIELD-NAME</em> `KEEPALIVE` and its <em>FIELD-VALUE</em> is an <em>int</em> with the TIMEOUT in tenths of seconds.

## Padding Option {#padding}

The Padding (OPTION-CODE 12 {{?RFC7830}}) option is represented by <em>FIELD-NAME</em> `PADDING` and its <em>FIELD-VALUE</em> is an <em>object</em> with two <em>SUBFIELD</em>s:

* first <em>FIELD-NAME</em> is `LENGTH` and its <em>FIELD-VALUE</em> is the OPTION-LENGTH as <em>int</em>

* second <em>FIELD-NAME</em> is `HEX` and its <em>FIELD-VALUE</em> is a <em>string</em> with base16-representation of OPTION-DATA

If the OPTION-DATA consists only of zeroes (0x00 octets), the `HEX` <em>SUBFIELD</em> SHOULD be an empty <em>string</em>.
Within JSON, the `HEX` <em>SUBFIELD</em> MAY be omitted if it is an empty string.

## CHAIN Option

The CHAIN (OPTION-CODE 13 {{?RFC7901}}) option is represented by <em>FIELD-NAME</em> `CHAIN` and its <em>FIELD-VALUE</em> is the Closest trust point as <em>dname</em>.

## Edns-Key-Tag Option

The edns-key-tag (OPTION-CODE 14 {{?RFC8145, Section 4}}) option is represented by <em>FIELD-NAME</em> `KEYTAG` and its <em>FIELD-VALUE</em> is the list of Key Tag values as <em>list</em> of <em>int</em>s.

## Extended DNS Error Option {#ede}

The Extended DNS Error (OPTION-CODE 15 {{?RFC8914}}) option is represented by <em>FIELD-NAME</em> `EDE` and its <em>FIELD-VALUE</em> is an <em>object</em> with three <em>SUBFIELD</em>s:

* first <em>FIELD-NAME</em> is `CODE` and its <em>FIELD-VALUE</em> is the INFO-CODE as <em>int</em>

* second <em>FIELD-NAME</em> is `Purpose` and its <em>FIELD-VALUE</em> is the Purpose (first presented in {{?RFC8914, Section 5.2}} and then governed by {{IANA.EDNS.EDE}}) as <em>string</em>, or an empty <em>string</em>

* third <em>FIELD-NAME</em> is `TEXT` and its <em>FIELD-VALUE</em> is the EXTRA-TEXT as <em>string</em> (possibly of zero length)

Within JSON, the `Purpose` <em>SUBFIELD</em> MAY be omitted if it is an empty string.
The same applies for `TEXT` <em>SUBFIELD</em>.

Examples of Presentation format:

~~~
EDE: 18 "Prohibited" ""
~~~
~~~
EDE: 6 "DNSSEC Bogus" "signature too short"
~~~

# Examples of EDNS Presentation Format {#eexamples}

The following examples shall illustrate the features of EDNS Presentation format described above.
They may not make much sense and should not appear in normal DNS operation.

~~~
. 0 IN EDNS (
    Version: 0
    FLAGS: DO
    RCODE: BADCOOKIE
    UDPSIZE: 1232
    EXPIRE: 86400
    COOKIE: 36714f2e8805a93d,4654b4ed3279001b
    EDE: 18 "Prohibited" "bad cookie\000"
    OPT1234: 000004d2
    PADDING: 113 ""
    )
~~~
~~~
. 0 IN EDNS ( FLAGS: 0 RCODE: BADSIG UDPSIZE: 4096 EXPIRE: NONE
              NSID: 6578616d706c652e636f6d2e "example.com."
              DAU: 8,10 KEEPALIVE: 600 CHAIN: zerobyte\000.com.
              KEYTAG: 36651,6113 PADDING: 8 "df24d08b0258c7de" )
~~~

# Examples of EDNS Representation in JSON {#jexamples}

The following examples are the JSON equivalents of the examples in [eexamples](#eexamples).
They may not make much sense and should not appear in normal DNS operation.

~~~
"EDNS": {
    "Version": 0,
    "FLAGS": [ "DO" ],
    "RCODE": "BADCOOKIE",
    "UDPSIZE": 1232,
    "EXPIRE": 86400,
    "COOKIE": [ "36714f2e8805a93d", "4654b4ed3279001b" ],
    "EDE": {
        "CODE": 18,
        "Purpose": "Prohibited",
        "TEXT": "bad cookie\u0000"
    },
    "OPT1234": "000004d2",
    "PADDING": {
        "LENGTH": 113
    }
}
~~~
~~~
"EDNS": { "FLAGS": [ ], "RCODE": "BADSIG", "UDPSIZE": 4096,
          "EXPIRE": "NONE", "NSID": { "HEX": "6578616d706c652e636f6d2e",
          "TXT": "example.com." }, "DAU": [ 8, 10 ], "KEEPALIVE": 600,
          "CHAIN": "zerobyte\\000.com.", "KEYTAG": [ 36651, 6113 ],
          "PADDING": { "LENGTH": 8, "HEX": "df24d08b0258c7de" } }
~~~

# Guidelines for Future EDNS Options {#guidelines}

This draft describes the presentation and JSON format of those ENDS options that are known at the time of writing.
Other EDNS options fall in the category of Unrecognized Options ([unrecognized](#unrecognized)), unless specified otherwise.
The following guidelines shall help define them.

When defining new EDNS options, it is recommended to specify their <em>FIELD-NAME</em>s, <em>FIELD-TYPE</em>s and the construction of <em>FIELD-VALUE</em>s so that the EDNS Presentation and JSON format comprehensibly handles them.
Those formats should follow the semantics of the options' values rather than the syntax in order to make them more human-readable.
If it is necessary to define a new <em>FIELD-TYPE</em>, care must be taken to define its representation in Presentation and JSON format in a similar fashion like in this document.

# Forward-Compatibility Considerations {#future}

This specification of ENDS Presentation and JSON format prefers displaying textual mnemonics over potentially cryptic numeric values wherever possible, which is desirable for human readers.
It refers to several IANA tables collecting the definitions of those mnemonics.
Those tables may be getting updated throughout time, and for human readers, it is still beneficial that the EDNS formats reflect those updates.
However, this may cause difficulties for algorithms implementing the reverse process of converting EDNS Presentation and/or JSON format back to wire format, because they might not understand some new mnemonics.
This limitation has to be taken into consideration.

Similarly, new documents may define Presentation and JSON format of newly defined EDNS options according to (or not according to) the guidelines above ([guidelines](#guidelines)).
This is, again, beneficial for human readers, as otherwise all new EDNS options would have to be represented as Unrecognized Options ([unrecognized](#unrecognized)).
However, this may also cause difficulties for algorithms implementing the reverse process of converting EDNS Presentation and/or JSON format back to wire format, because they might not understand some new options.

# Update Representing DNS Messages in JSON {#jsonescaping}

This section is not related to EDNS.
This section updates {{!RFC8427, Section 2.6}}, including erratum 5439, which introduced contradicting MUSTs for escaping backslashes.

In order to solve this contradiction and correctly represent a DNS name in JSON, it MUST be first converted to textual Presentation format according to {{!RFC1035, Section 5.1}} (called master file format in the referenced document), and the resulting &lt;character-string&gt; subsequently is inserted into JSON as String ({{!RFC8259, Section 7}}).

Note that the previous paragraph prescribes the following escaping strategy:
In the first step, every problematic character (non-printable, backslash, dot within Label, or any octet) is either substituted with the sequence `\DDD`, where `DDD` is the three-digit decimal ASCII code, or in some cases (backslash, dot, any printable character) just prepended with a backslash.
In the second step, every quote (`"`) and backslash (`\`) in the resulting &lt;character-string&gt; is prepended with another backslash.
Note that the JSON escaping sequence `\uXXXX` (where `XXXX` is a hexadecimal Unicode code) is thus never needed.

Moreover, the following requirements from {{!RFC8427}} still hold:
The name MUST be represented as an absolute Fully-Qualified Domain Name.
Internationalized Domain Name (IDN) labels MUST be expressed in their A-label form, as described in {{!RFC5890}}.

Example: the name with the Wire format `04005C2E2203636F6D00` can be represented in JSON as:

~~~
"NAME": "\\000\\\\\\046\".com."
~~~

but also as (among other ways):

~~~
"NAME": "\\000\\092\\.\\\".c\\om."
~~~

# IANA Considerations {#iana}

This document has no IANA actions.

# Security Considerations {#security}

This document only describes the textual representation of binary data and therefore has no security impact on related protocols.

When implementing software, care must be taken to handle possibly inconsistent or broken input data.

# Acknowledgements

TODO

# Implementation Status

**Note to the RFC Editor**: Please remove this entire appendix before publication.

This version of this specification draft-peltan-edns-presentation-format-02 has been implemented in Knot DNS 3.3.2.

# Change History

**Note to the RFC Editor**: Please remove this entire appendix before publication.

* edns-presentation-format-00

> Initial public draft.

* edns-presentation-format-01

> Added Guidelines for Future EDNS Options, dummy IANA Considerations and Security Considerations.

* edns-presentation-format-02

> Substantial re-work with common FIELD-TYPE specifications, bigger changes in presentation format and smaller in JSON.

--- back
