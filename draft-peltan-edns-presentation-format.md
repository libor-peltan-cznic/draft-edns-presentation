%%%
Title = "EDNS Presentation Format"
abbrev = "edns-presentation-format"
docname = "@DOCNAME@"
category = "std"
ipr = "trust200902"
area = "Internet"
#workgroup = "DNSOP Working Group"
date = @TODAY@

[seriesInfo]
name = "Internet-Draft"
value = "@DOCNAME@"
stream = "IETF"
status = "standard"

[[author]]
initials = "L."
surname = "Peltan"
fullname = "Libor Peltan"
organization = "CZ.NIC"
[author.address]
 email = "libor.peltan@nic.cz"
[author.address.postal]
 country = "CZ"
%%%

.# Abstract

This document describes textual presentation format of EDNS option.

{mainmatter}

# Introduction

DNS records[@!RFC1035] of any type can be converted between their binary Wire format, used in DNS messages transferred over the Internet, and textual Presentation format, used not only in Zone Files (called "master files" in the referenced document), but also to displaying contents of DNS messages to human by debugging utilities, and possibly other use-cases.
The Presentation format can be however processed also programatically and also converted back to Wire Format unambiguously.

EDNS[@!RFC6891] option pseudorecord does not appear in Zone Files, but it sometimes needs to be converted to human-readable, but also machine-readable textual representation.
This document describes such Presentation Format of OPT pseudorecord.
It is advised to use this when displaying an OPT pseudorecord to humans.
It is recommended to use this when the textual format is expected to be machine-processed further.

# Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**",
"**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**",
"**RECOMMENDED**", "**NOT RECOMMENDED**", "**MAY**", and
"**OPTIONAL**" in this document are to be interpreted as described in
BCP 14 [@!RFC2119;@!RFC8174] when, and only when, they appear in all
capitals, as shown here.

EDNS(0) stands for EDNS version 0.

Decimal value means an integer displayed in decimals with no leading zeroes.

Base16 is the representation of arbitrary binary data by an even number of case-insensitive hexadecimal digits [@!RFC4648], section 8.

# Version-independent Presentation Format {#independent}

EDNS versions other than 0 are not yet specified, but an OPT pseudorecord with version field set to value other than zero might in theory appear in DNS messages.
This section specifies how to convert such OPT pseudorecord to Presentation format.
This procedure SHOULD NOT be used for EDNS(0).

OPT pseudorecord is in this case represented the same way as a RR of unknown type according to [@!RFC3597], section 5. In specific:

* Owner name is `.` (DNS Root Domain Name).

* TTL is Decimal value of the 32-bit big-endian integer appearing at the TTL position of OPT pseudorecord Wire format, see [@!RFC6891], section 6.1.3.

* CLASS is a text representation of the 16-bit integer at the CLASS position of OPT pseudorecord Wire format (UDP payload size happens to appear there).
This will usually result in `CLASS####` (where #### will be the Decimal value), but it might also result for example in `IN` or `CH` if the value is 1 or 4, respectively.

* TYPE is either `TYPE41` or `OPT`.

* RDATA is formatted by `\#`, its legth as Decimal value, and data as Base16 as per [@!RFC3597], section 5.

Example: `. 16859136 CLASS1232 TYPE41 \# 6 000F00020015`

# EDNS(0) Presentation Format

EDNS(0) Presentation Format follows RR format of master file [@!RFC1035], section 5.1, including quotation of non-printable characters, multi-line format using round brackets, and semicolons denoting comments.

Owner Name MAY be omitted. If it is present, it MUST be `.` (DNS Root Domain Name).

TTL MAY be omitted. If it is present, it MUST be `0` (zero).
Please note, that this differs from DNS RR wire-to-txt conversion, as well as Version-independent format by (#independent).

CLASS MAY be omitted. If it is present, it SHOULD be `IN`. The exception is when the EDNS pseudo-record appears in a context of other class (e.g. a DNS message querying `CH` class).

TYPE MUST be `EDNS0`.

RDATA consists of at least three <character-string>s [@!RFC1035], section 5.1.
It is recommended to use the multi-line format with comments at each field with more human-readable form of the contents of each option.
See examples in (#examples).

The first three <character-string>s denote:

* Flags as Base16 (four case-insensitive hexadecimal digits).

* EXTENDED-RCODE (only the upper 8 bits) as Decimal value.
It is recommended to add a comment with resulting RCODE (all 12 bits) and its Description.

* UDP payload size as Decimal value.

The rest of <character-string>s are one <character-string> per option.
The options MUST be displayed in the same order as they appear in Wire format.
Each option consists of following parts, with no separators nor spaces between them:

* Option name, which is defined for each option

* Opening square bracket `[`

* OPTION-LENGTH as Decimal value

* Closing square bracket `]`

* Equal sign `=`

* Option value, which is defined for each option

If Option value is empty or omitted, the equal sign MUST be omitted as well.

## Unrecognized Option

At least all the options listed below MUST be recognized.
Their Presentation format is specified by following subsections.
Other options MAY be recognized as well, if the description of their Presentation format is part of their specification.

Unrecognized option name is `OPTXX`, where `XX` stands for its OPTION-CODE, and option value is displayed as Base16.

## LLQ Option

TODO

## NSID Option

NSID (OPTION-CODE 3 [@!RFC5001]) Option name is `NSID` and Option value is displayed in Base16.

It is recommended to add a comment with ASCII representation of the value.

## DAU, DHU and N3U Options

DAU, DHU, and N3U (OPTION-CODES 5, 6, 7, respectively [@!RFC6975]) Options names are `DAU`, `DHU`, and `N3U`, respectively, and their Option values cosist of comma-separated lists of ALG-CODEs as Decimal values.

## Edns-Client-Subnet Option

EDNS Client Subnet (OPTOIN-CODE 8 [@!RFC7871]) Option name is `ECS` and Option value consists of comma-separated list of fields:

* FAMILY, which is `IP` for IPv4 (Address Family Number 1), `IP6` for IPv6 (Address Family Number 2), or Decimal value otherwise.

* SOURCE PREFIX-LENGTH as Decimal value

* SCOPE PREFIX-LENGTH as Decimal value

* ADDRESS represented as:

  * Comma-separated list of textual IPv4 addresses [TODO reference], if FAMILY is 1
  
  * Comma-separated list of testual IPv6 addresses [@!RFC2373], section 2.2, if FAMILY is 2
  
  * Base16 otherwise or if the ADDRESS field is of unexpected length

## EDNS EXPIRE Option

EDNS EXPIRE (OPTION-CODE 9 [@!RFC7314]) Option name is `EXPIRE` and its value, if present, is displayed as Decimal value.

## Cookie Option

DNS Cookie (OPTION-CODE 10 [@!RFC7873]) Option name is `COOKIE` and its Option value consists of Client Cookie as Base16, followed by a comma, followed by Server Cookie as Base16.
The comma and Server Cookie is displayed only if OPTION-LENGTH is greater than 8.

## Edns-Tcp-Keepalive Option

edns-tcp-keepalive (OPTION-CODE 11 [@!RFC7828]) Option name is `KEEPALIVE` and its value is displayed as Decimal value.

## Padding Option

Padding (OPTION-CODE 12 [@!RFC7830]) Option name is `PADDING` and its value SHOULD be omitted.
If the value is present, it is displayed as Base16.

## CHAIN Option

CHAIN (OPTION-CODE 13 [@!RFC7901]) Option name is `CHAIN` and its value is displayed as a textual Fully-Qualified Domain Name.

## Edns-Key-Tag Option

ends-key-tag (OPTION-CODE 14 [@!RFC8145], section 4) Option name is `KEYTAG` and its value is displayed as a comma-separated list of Decimal values.

## Extended DNS Error Option

Extended DNS Error (OPTION-CODE 15 [@!RFC8914]) Option name is `EDE` and Option value consists of INFO-CODE as Decimal value, followed by a single space, followed by EXTRA-TEXT as a string of UTF-8 characters.
If EXTRA-TEXT is empty, the preceding space MUST be omitted as well.

It is recommended to add a comment with Purpose of given code[@!RFC8914], seciton 5.2.

(Note that the (usual) presence of space within this <character-string> requires it to be inside quotes as a whole, not only the part behind `=`. Also note that non-ASCII characters and trailing zero octet in EXTRA-TEXT will be escaped by the rules of RFC1035.)

# Examples of EDNS(0) Presentation Format {#examples}

Following example shall illustrate the features of EDNS(0) Presentation format described above.
It may not make really sense and should not appear in normal DNS operation.

```
. 0 IN EDNS0 (
	4000	; DNSSEC OK
	1	; RCODE=23 BADCOOKIE
	1232
	COOKIE[16]=36714f2e8805a93d,4654b4ed3279001b
	"EDE[12]=18 bad cookie"	; Prohibited
	OPT1234[4]=000004d2
	PADDING[114]
	)
```

TODO

# EDNS(0) JSON Format

TODO part of this document, or separately?

# Security Considerations {#security}

None.

# Acknowledgements

TODO

{backmatter}

# Implementation Status

**Note to the RFC Editor**: please remove this entire appendix before publication.

None yet.

# Change History

**Note to the RFC Editor**: please remove this entire appendix before publication.

* edns-presentation-format-00

> Initial public draft.
