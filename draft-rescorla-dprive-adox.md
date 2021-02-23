---
title: Signaling Authoritative DNS Encryption
abbrev: SAD
docname: draft-rescorla-dprive-adox-latest
category: info

ipr: trust200902
area: General
workgroup: dprive
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
    ins: D. Schinazi
    name: David Schinazi
    organization: Google LLC
    email: dschinazi.ietf@gmail.com

 -
    ins: C. A. Wood
    name: Christopher A. Wood
    organization: Cloudflare
    email: caw@heapingbits.net



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

# Overview of Operation {#overview}

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
for .com and asks for the NS records for example.com. Note that .com
is not authoritative for the example.com apex, and will not sign the
NS RRset; see {{?RFC4035}}, Section 2.2, and {{security}} for details.
The authoritative returns the NS record pointing at ns.example.invalid and
also returns a glue records for ns.example.invalid indicating that it
supports DNS over TLS (DoT), in much the same way as it might have sent an
IP address for that server. This additional record is the only difference
from the current situation, and allows the recursive resolver to know that
it can reach ns.example.invalid over encrypted transport.


# Use of SVCB Records to Signal Encrypted Transport

Any given authoritative resolver name can have one or more DNS Server
SVCB records, as defined in {{?I-D.schwartz-svcb-dns}}.

For instance, the following pair of records would indicate that
ns.example.invalid could be reached by either DoT or DoH (over
both TCP and QUIC).

~~~~
   ns.example.invalid 7200 IN SVCB 1 . alpn=dot
   ns.example.invalid 7200 IN SVCB 1 . alpn=h2,h3 dohpath=/dns-query{?dns}
~~~~

Upon determining that a given nameserver supports a compatible
encrypted transport, an implementation MUST only use encrypted
transport for the rest of the cache lifetime for that information
and MUST hard fail with error if it is unable to establish a connection.
If multiple encrypted transports are available, an implementation
SHOULD try all of them before declaring failure.

[[OPEN ISSUE: figure out error details]]

## Caching and lifetime

Note that in the common case where the name of the target
authoritative resolver is out-of-bailiwick {{?RFC7719}} for the
referring resolver, then the SVCB record may not be retained
for future queries. This can create a situation in which a given
authoritative resolver will be queried over encrypted transport for
one name and over unencrypted transport for another. This is not the
end of the world (HTTPS has historically operated in this way, with
the security properties being attached to the reference), but is also
not ideal. In order to prevent this, resolvers which are also
authoritative for their own name SHOULD send SVCB glue records
in the additional data section so that they can be properly cached,
and the TTL for these SVCB records SHOULD match that of the
corresponding NS records in the same RRset.

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

# Example

A complete example is shown below.

~~~~
Recursive  x.root-servers.org  ns.a.invalid         ns.example.invalid
             (Auth. for .)     (Auth for .invalid)  (Auth for .example.invalid)

<== TLS handshake ==>
NS .invalid? ------->
<- .invalid NS ns.op.invalid
   ns.op.invalid A 198.51.100.1
   _dns.ns.op.invalid SVCB alpn=dot


<============ TLS handshake ===========>
NS example.invalid? ------------------->
<----- example.invalid NS ns.example.invalid
       ns.example.invalid A 203.0.113.1
       _dns.ns.example.invalid SVCB alpn=dot

<====================== TLS Handshake ======================>
A www.example.invalid? ------------------------------------->
<---------------------------- www.example.invalid A 192.0.2.1
~~~~

In this case, the recursive wants to resolve www.example.invalid.

Resolution proceeds in three phases.

Initially, the recursive connects to the root server. We assume that
the recursive knows that the root server is able to do DoT, either
because it has been preconfigured with his information or because it
has connected to that root server before. It performs an NS query for
".invalid" (we are assuming QMIN) and receives:

- An NS record pointing to ns.op.invalid
- A glue A record for ns.op.invalid = 198.51.100.1
- A SVCB record stating that ns.op.invalid speaks DoT

Next, the recursive resolver forms a TLS connection to ns.op.invalid
and requests an NS record for example.invalid. It receives:

- An NS record pointing to ns.example.invalid
- A glue A record for ns.example.invalid = 203.0.113.1
- A SVCB record stating that ns.example.invalid speaks DoT

Finally, the recursive resolver forms a TLS connection to
ns.example.invalid and request an A record for www.example.invalid and
receives the A record of 192.0.2.1.


# Security Considerations {#security}

The primary security property delivered by this mechanism is
confidentiality of the query and response. As long as (1) all queries
in the resolution chain, including to the authoritative resolver are
encrypted and (2) all resolvers in the resolution chain are
trustworthy, then even an on-path attacker cannot discover the
name being resolved or its response. However, if either of these conditions
is violated, then an attack is possible:

- If the connection to the authoritative resolver is not encrypted,
  then the request and response can be read directly.
- If one of the earlier connections is not encrypted, then the
  attacker can substitute their own NS records.
  records from the additional_data, forcing the resolution back to unencrypted mode.
- If one of the resolvers is untrustworthy, then they can
  substitute their own NS records.

DNSSEC signing only partly mitigates these issues because delegations
at top-level zones are not signed, as per {{?RFC4035}}, Section
2.2. In practice, this means a recursive resolver attempting to
resolve a zone apex query, such as example.com in {{overview}}, cannot
assume the NS answer is authentic. While NS records received from the
authoritative server may be signed, in order to retrieve them, the
recursive resolver will have to contact the servers listed by the
unverified NS records received from the referring server, at which
point it has leaked the zone apex to the (potentially fake)
authoritative server (as well as to the referring server).

If the recursive resolver is attempting to resolve a specific
subdomain from the resolver (e.g., server-1234.example.com), then it
may be able to protect against this attack by (1) using query
minimization {{?QMIN=RFC7816}} and (2) querying the (alleged)
authoritative for its DNSSEC-signed NS and SVCB records
and only once it has received those, attempting to retrieve
the actual subdomain. If the domain is DNSSEC signed, then
this prevents a malicious referring resolver from redirecting
the recursive resolver to their own authoritative and learning
the true subdomain. However, if, as is common, the recursive
is just trying to resolve the apex name or one of the common
"service" names such as "www.example.com", then this procedure
does not provide additional protection.

Encryption does not mitigate all leakage. In some circumstances, an
on-path attacker may learn the identity of the authoritative server if,
for example, that server only serves a small number of domains. The
attacker can learn information about what is being resolved by observing
whether or not server is queried.

As a secondary property, the mechanism in this document can provide some level
of integrity for DNS responses, again under the condition that each resolver
in the chain is trustworthy. By contrast, DNSSEC provides integrity even if
the resolvers are untrustworthy.




# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
