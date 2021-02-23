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
over encrypted transport {{!DOH=RFC8484}} {{!DOT=RFC7858}}
{{?DOQ=I-D.ietf-dprive-dnsoquic}}. However, there is no scalable
way for a recursive resolver to know that a given authoritative
resolver supports encrypted transport, which inhibits the deployment
of encryped DNS for queries from recursive resolvers. This specification
defines a mechanism for carrying that signal, using the
DNS SVCB {{?SVCB=I-D.ietf-dnsop-svcb-https}} record.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Overview of Operation

The mechanism defined in this document works by using the DNS SVCB
{{SVCB}} record to indicate that a given server supports TLS. The
recursive resolver can obtain these records in two distinct ways:

- In the additional data block of the response that referred
  the recursive to the target authoritative resolver.
- By directly resolving a SVCB query for the target authoritative
  resolver.

As a practical matter, the first of these options is preferred
as it allows the recursive to learn that the authoritative
server supports encrypted transport without an additional round
trip, as shown below:

~~~~
Recursive                .com                      ns.example.invalid
                   Authoritative Server      (Authoritative for example.com)
NS example.com? ------------>

<----- example.com NS ns.example.invalid
       ns.example.invalid A 192.0.2.1
       _dns.ns.example.invalid SVCB alpn=dot

<--------------  TLS connection to ns.example ------------>
A example.com? ------------------------------------------->
~~~~

The recursive resolver starts by contacting the authoritative server
for .com and asks for the NS records for example.com.
[[OPEN ISSUE: Is this right?]]
The
authoritative returns the NS record pointing at ns.example.invalid and
also returns a glue records for ns.example.invalid
indicating that it supports DNS over
TLS (DoT), in much the same way as it might have sent an
IP address for that server.
This additional record is the only difference from the
current situation, and allows the recursive resolver to know that
it can reach ns.example.invalid over encrypted transport.


# Use of SVCB Records to Signal Encrypted Transport

Any given authoritative resolver name can have one or more DNS Server
SVCB records, as defined in {{?I-D.schwartz-svcb-dns}}.

For instance, the following pair of records would indicate that
ns.example.invalid could be reached by either DoT or DoH (over
both TCP and QUIC).

~~~~
   _dns.ns.example.invalid 7200 IN SVCB 1 . alpn=dot
   _dns.ns.example.invalid 7200 IN SVCB 1 . alpn=h2,h3 dohpath=/dns-query{?dns}
~~~~

Upon determining that a given nameserver supports a compatible
encrypted transport, an implementation MUST only use encrypted
transport for the rest of the cache lifetime for that information
and MUST hard fail with error [[TODO]] if it is unable to establish a connection.
If multiple encrypted transports are available, an implementation
SHOULD try all of them before declaring failure.


## Caching and lifetime

Note that in the common case where the name of the target
authoritative resolver is out-of-bailiwick {{?RFC7719}} for the
referring resolver, then the SVCB record will likely not be retained
for future queries. This can create a situation in which a given
authoritative resolver will be queried over encrypted transport for
one name and over unencrypted transport for another. This is not the
end of the world (HTTPS has historically operated in this way, with
the security properties being attached to the reference), but is also
not ideal. In order to prevent this, resolvers which are also
authoritative for their own name SHOULD send SVCB records back
for themselves in the additional data section so that they can
be properly cached.

[[OPEN ISSUE: Do people cache out-of-bailiwick DNSSEC-signed records?]]
[[OPEN ISSUE: How often is the case where ns.example.invalid is not
authoritative for itself? Should we encourage people to accept out-of-bailiwick
responses in that case?]]



TODO:
- How is caching handled now for the IP addresses
- Do you apply it to other NS records?
- You can't just allow later glue to override earlier glue.


## Authenticating the Server

Recursive servers MUST authenticate the authoritative server
using the procedures associated with the relevant protocol,
{{!RFC6125}} and {{!RFC2818}} for DoT and DoH respectively.

# Security Considerations

The primary security property delivered by this mechanism is
confidentiality of the query and response. As long as (1) All queries
in the resolution chain, including to the authoritative resolver are
encrypted and (2) All resolvers in the resolution chain are
trustworthy, then even an on-path attacker cannot discover the
name being resolved or its response. However, if either of these conditions
is violated, then an attack is possible:

- If the connection to the authoritative resolver is not encrypted,
  then the request and response can be read directly.
- If one of the earlier connections is not encrypted, then the
  attacker can substitute their own NS records or strip SVCB
  records from the additional_data, forcing the resolution back to unencrypted mode.
- If one of the resolvers is untrustworthy, then they can
  sending SVCB records or substitute their own NS records.

Note: DNSSEC signing might mitigate some these issues, but it is
undesirable to have a system which depends on universal DNSSEC.
[[TODO: Tommy, can you analyze this?. I think NS aren't signed
so basically it's hopeless]]

An on-path attacker does, however, likely learn the identity of
the authoritative server; if that server only serves a small
number of domains then the attacker will learn information
about what is being resolved.

As a secondary property, this mechanism can provide some level
of integrity for DNS responses, again under the condition that
each resolver in the chain is trustworthy. By contrast,
DNSSEC provides integrity even if the resolvers are untrustworthy.




# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.