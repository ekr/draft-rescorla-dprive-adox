---
title: Signaling Authoritative DNS Encryption
abbrev: SAD
docname: draft-rescorla-dprive-adox-latest
category: info

ipr: trust200902
area: General
workgroup: TODO Working Group
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:

 -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    email: tpauly@apple.com

 -
    ins: E. Rescorla
    name: Eric Rescorla
    organization: Mozilla
    email: ekr@rtfm.com

 -
    ins: C. A. Wood
    name: Christopher A. Wood
    organization: Cloudflare
    email: caw@heapingbits.net

 -
    ins: "D. Schinazi"
    name: "David Schinazi"
    organization: "Google LLC"
    email: dschinazi.ietf@gmail.com



normative:
  RFC2119:

informative:



--- abstract

This document defines a mechanism for signaling that a given
authoritative DNS server is reachable by encrypted DNS.

--- middle

# Introduction

The IETF has defined a number of mechanisms for carrying DNS queries
over encrypted transport {{!RFC8484}} {{!RFC7858}}
{{?I-D.ietf-dprive-dnsoquic}}. However, there is no scalable
way for a recursive resolver to know that a given authoritative
resolver supports encrypted transport, which inhibits the deployment
of encryped DNS for queries from recursive resolvers. This specification
defines a mechanism for carrying that signal, using the
DNS SVCB {{!I-D.ietf-dnsop-svcb-https}} record.



# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Overview of Operation

The mechanism defined in this document works by placing an additional set
of glue records at the referring resolver. These records allow the recursive
resolver to learn that the servers associated with the NS records it receives
also support encrypted transport, as shown below:

~~~~
Recursive                .com                      ns.example.invalid
                   Authoritative Server      (Authoritative for example.com)
NS example.com? ------------>

<----- example.com NS ns.example.invalid
       ns.example.invalid A 192.0.2.1
       ns.example.invalid SVCB <DoT>

<--------------  TLS connection to ns.example ------------>
A example.com? ------------------------------------------->
~~~~

The recursive resolver starts by contacting the authoritative server
for .com and asks for the NS records for example.com.
[[OPEN ISSUE: Is this right?]]
The
authoritative returns the NS record pointing at ns.example.invalid and
also returns two glue records for ns.example.invalid: an A record with
its IP address and a SVCB record indicating that it supports DNS over
TLS (DoT). This additional record is the only difference from the
current situation, and allows the recursive resolver to know that
it can reach ns.example.invalid over encrypted transport.


# Use of SVCB Records to Signal Encrypted Transport

Any given authoritative resolver name can have one or more SVCB
records. These records MUST contain the ALPN key, with one of
the following values:

|:value|:semantics|
|dot|DNS over TLS as defined in {{RFC7858}}|
|h2|DNS over HTTPS as defined in {{RFC8484}}|

[[TODO: tokens for DoQ and DoHQ]]

For instance, the following pair of records would indicate that
ns.example.invalid could be reached by either DoT or DoH
but prefers DoT.

~~~~
   IN SVCB 0 ns.example.invalid alpn=dot
   IN SVCB 1 ns.example.invalid alpn=h2
~~~~

[[TODO: Tommy, can you fill in if this is wrong?]].

Upon determining that a given nameserver supports a compatible
encrypted transport, an implementation MUST only use encrypted
transport for the rest of the cache lifetime for that information
and MUST hard fail with error [[TODO]] if it is unable to establish a connection.
If multiple encrypted transports are available, an implementation
SHOULD try all of them before declaring failure.


## Caching and lifetime

TODO:
- How is caching handled now for the IP addresses
- Do you apply it to other NS records?
- You can't just allow later glue to override earlier glue.


## Authenticating the Server

Recursive servers MUST authenticate the authoritative server
using the procedures associated with the relevant protocol,
{{!RFC6125}} and {{!RFC2818}} for DoT and DoH respectively.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
