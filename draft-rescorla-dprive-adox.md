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


## Authenticating the Server

Recursive servers MUST authenticate the authoritative server using the
procedures associated with the relevant protocol, {{!RFC6125}} and
{{!RFC2818}} for DoT and DoH respectively. This is in principle
compatible with having the server authenticated either with the WebPKI
or with TLSA records {{!DANE=RFC6698}}, or both. In order for this to
work properly, however, the recursive resolver must know at the
time it connects whether it will be willing to accept the authoritative
server's credentials.

This can be addressed in several ways:

1. Require a particular form of authentication (e.g., the WebPKI or
   TLSA records) as mandatory.
1. Have the SVCB record indicate what kind(s) of authentication the
   authoritative resolver supports, allowing the recursive to filter
   out incompatible advertisements.

One challenge with TLSA records in this context is that they may
not be in the recursive resolver's cache at the time when it wants
to connect to the authoritative. This can create added latency
if the recursive resolver must then first retrieve TLSA records
for the authoritative. If we wish for servers to authenticate
with DANE, we will also probably want some mechanism to carry
the TLSA records in the TLS handshake, as, for instance
defined in {{?I-D.ietf-tls-dnssec-chain-extension}}.

[[OPEN ISSUE: Resolve this.]]

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
