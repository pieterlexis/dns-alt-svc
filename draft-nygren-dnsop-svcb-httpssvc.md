---
title: Service binding and parameter specification via the DNS (DNS SVCB and HTTPSSVC)
abbrev: SVCB and HTTPSSVC RRs for DNS
docname: draft-nygren-dnsop-svcb-httpssvc-latest
date: {DATE}
category: std

ipr: trust200902
area: General
workgroup: DNSOP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Schwartz
    name: Ben Schwartz
    organization: Google
    email: bemasc@google.com
 -
    ins: M. Bishop
    name: Mike Bishop
    org: Akamai Technologies
    email: mbishop@evequefou.be
 -
    ins: E. Nygren
    name: Erik Nygren
    org: Akamai Technologies
    email: erik+ietf@nygren.org

normative:

informative:

--- abstract

This document specifies the "SVCB" and "HTTPSSVC" DNS resource record types to
facilitate the lookup of information needed to make connections for
origin resources, such as for HTTPS URLs.  The SVCB DNS RR
mechanism allows an origin hostname to be served from multiple network
locations, each with associated parameters (such as transport protocol
configuration and keying material for encrypting TLS SNI).  It also
provides a solution for the inability of the DNS to allow a CNAME to
be placed at the apex of a domain name.  The HTTPSSVC DNS RR
is a more-specific variation of the SVCB RR which is specific
to the use-case of HTTPS and HTTP origin resources.

By allowing this information to be bootstrapped in the DNS,
it allows for clients to learn of alternative services before their first
contact with the origin.  This arrangement offers potential benefits to
both performance and privacy.

This document specifies both a general-purpose record (via SVCB), but
also specifies explicit behavior for HTTP and HTTPS URIs (via
HTTPSSVC).  As part of the latter, it also provides a way to indicate
that the origin supports HTTPS without having to resort to redirects,
allowing clients to remove HTTP from the bootstrapping process.

TO BE REMOVED: This proposal is inspired by and based on recent DNS
usage proposals such as ALTSVC, ANAME, and ESNIKEYS (as well as long
standing desires to have SRV or a functional equivalent implemented
for HTTP).  These proposals each provide an important function but are
potentially incompatible with each other, such as when an origin is
load-balanced across multiple hosting providers (multi-CDN).
Furthermore, these each add potential cases for adding additional
record lookups in-addition to AAAA/A lookups. This design attempts to
provide a unified framework that encompasses the key functionality of
these proposals, as well as providing some extensibility for
addressing similar future challenges.

TO BE REMOVED: The specific name for this RRtype is an open
topic for discussion.  "SVCB" and "HTTPSSVC" are meant as placeholders
as they are easy to replace.  Other names might include "B",
"SRV2", "SVCHTTPS", "HTTPS", and "ALTSVC".


--- middle

# Introduction

The SVCB and HTTPSSVC RRs are intended to address a number of challenges facing
clients and services, especially when origin resources are available
over a range of protocol configurations.  This is done in a way that
provides an extensible model to handle similar use-cases without
forcing clients to perform additional DNS lookups and without forcing
them to first make connections to a default service for the origin.

For example, when clients need to make a connection to fetch resources
associated with an HTTPS URI, they must first resolve A and/or AAAA
address resource records for the origin hostname.  This is adequate
when clients default to TCP port 443, do not support Encrypted SNI
{{!ESNI=I-D.ietf-tls-esni}}, and where the origin service operator
does not have a desire to put an CNAME at a zone apex (such as for
"https://example.com").  Handling situations beyond this within the
DNS requires learning additional information, and it is highly
desirable to minimize the number of round-trip and lookups required to
learn this additional information.

The SVCB and HTTPSSVC RRs also help in scenarios where an the operator of an origin
resource in a domain wishes to delegate operational control to one or
more service operators who manage other domains, such as for
delegating the origin resource "https://example.com" to a service
operator endpoint at "svc.example.net".  While this case can sometimes
be handled by a CNAME, that does not cover all use-cases.  It also is
inadequate when the service operator needs to provide a bound
collection of consistent configuration parameters through the DNS
(such as network location, protocol, and keying information).

This document starts off describing the SVCB RR as a general-purpose resource
record that can be applicable to a wide range of services.
As HTTPS is a primary use-case and with special requirements,
the HTTPSSVC RR is also defined and special-cased within this document.

It is intentional for SVCB to be able to support a wide range of
services and protocols, many of which may be able to use it directly.
For services not wishing to require the use
of an {{?Attrleaf}} label with SVCB (as is the case with HTTPS and driving the need
for a separate HTTPSSVC RRType), future documents may assign
additional dedicated RRTypes for those serviices.

All behaviors described as applying to the SVCB RR also apply
to the HTTPSSVC RR unless explicitly stated otherwise.
The section {{#https}} describes additional behaviors
specific to the HTTPSSVC record.  For brevity, much of this
document will outside of the {{#https}} section (and outside
of introductory examples) will reference just the SVCB RR,
but those references should be taken to apply to both SVCB and HTTPSSVC.

The SVCB RR has two forms: 1) the "Alias Form" simply delegates operational
control for a resource; 2) the "Service Form" binds together
configuration information for a service endpoint.
The Service Form provides additional key=value parameters
within each RRData set.

TO BE REMOVED: If we use this for providing configuration for DNS
authorities, it is likely we'd specify a distinct "NS2" RRType that is
an instantiation of SVCB for authoritative nameserver delegation and
parameter specification, similar to HTTPSSVC.

TO BE REMOVED: Another open question is whether SVCB records
should be self-descriptive and include the service name
(eg, "https") in the RRData section to avoid ambiguity.
Perhaps this could be included as a svc="baz" parameter
for protocols that are not the default for the RRType?
Current inclination is to not do so.

## Introductory Example

As an introductory example for an HTTPS origin resource, a set of
example HTTPSSVC and associated A+AAAA records might be:

    www.example.com.  2H  IN CNAME   svc.example.net.
    ; AliasForm
    example.com.      2H  IN HTTPSSVC 0 svc.example.net.
    ; ServiceForm
    svc.example.net.  2H  IN HTTPSSVC 2 svc3.example.net. alpn=h3 port=8003 \
                                     esnikeys="..."
    svc.example.net.  2H  IN HTTPSSVC 3 svc2.example.net. alpn=h2 port=8002 \
                                     esnikeysref=esni-svc2.example.net.
    svc2.example.net. 300 IN A       192.0.2.2
    svc2.example.net. 300 IN AAAA    2001:db8::2
    svc3.example.net. 300 IN A       192.0.2.3
    svc3.example.net. 300 IN AAAA    2001:db8::3
    
In the preceding example, both of the "example.com" and
"www.example.com" origin names are aliased to use alternative service
endpoints offered as "svc.example.net" (with "www.example.com"
continuing to use a CNAME alias).  HTTP/2 is available on a cluster of
machines located at svc2.example.net with TCP port 8002 and HTTP/3 is
available on a cluster of machines located at svc3.example.net with
UDP port 8003.  ESNI keys specified which allows the SNI values of
"example.com" and "www.example.com" to be encrypted in the handshake
with these alternative service endpoints.  One of these keys is
provided as a literal value while the other is provided by an external
DNS name ("esni-svc2.example.net").  When connecting, clients will
continue to treat the authoritative origins as "https://example.com"
and "https://www.example.com", respectively.

For services other than HTTPS (as well as for HTTPS origins
with non-default ports), the SVCB RR and an {{?Attrleaf}} label will be used.
For example, to reach an example resource of
"baz://api.example.com:8765", the following Alias Form
SVCB record would be used to delegate to "svc4-baz.example.net."
which in-turn could return AAAA/A records and/or SVCB
records in ServiceForm.

    _8765._baz.api.example.com. 2H IN SVCB 0 svc4-baz.example.net.


## Goals of the SVCB RR

The goal of the SVCB RR is to allow clients to resolve a single
additional DNS RR in a way that:

* Provides service endpoints authoritative for an origin,
  along with parameters associated with each of these endpoints.
  In particular:
    * to support connecting directly to {{!HTTP3=I-D.draft-ietf-quic-http-20}} (QUIC transport)
      alternative service endpoints
    * to obtain the {{!ESNI}} keys associated with an alternative service endpoint
    * to support alternate TCP and UDP ports for the alternative
      service endpoint, beyond the default for the service
* Does not assume that all alternative service endpoints have the same parameters
  (such as ESNI keys) or capabilities (such as {{!HTTP3}}) or are even
  operated by the same entity.  This is important as DNS does not
  provide any way to tie together multiple RRs for the same name.
  For example, if www.example.com is a CNAME alias that switches
  between one of three CDNs or hosting enviroments, records (such as A and AAAA)
  for that name may have been sourced from different environments.
* Enables the functional equivalent of a CNAME at a zone apex (such as
  "example.com") for alternative service endpoints including HTTPS, and generally
  enables delegation of operational authority for an origin within the
  DNS to an alternate name.

Additional goals specific to HTTPSSVC and the HTTPS use-case include:

* Address a set of long-standing issues due to HTTP(S) clients not
  implementing support for SRV records, as well as due to a limitation
  that a DNS name can not have both a CNAME record as well as NS RRs
  (as is the case for zone apex names)
* Provide an HSTS-like indication signalling
  for the duration of the DNS RR TTL that the HTTPS scheme should
  be used instead of HTTP (see {{#hsts}}).

## Overview of the SVCB RR

This subsection briefly describes the SVCB RR in
a non-normative manner.  (As mentioned above, this all
applies equally to the HTTPSSVC RR which shares
the same encoding, format, and high-level semantics.)

The SVCB RR has two forms: AliasForm and ServiceForm.
SVCB RR entries with three primary non-empty fields are in ServiceForm.
When the third field is empty, this indicates that the SVCB RR
is in AliasForm.

1. SvcFieldPriority: The priority of this record (relative to others,
   with lower values preferred).  Applicable for the ServiceForm,
   and otherwise has value "0".  (Described in {{pri}}.)
2. SvcDomainName: The domain name of either the alias target (for 
   AliasForm) or the uri-host domain name of the alternative service
   endpoint (for ServiceForm).
3. SvcFieldValue: An Service field value containing key=value pairs
   describing the alternative service endpoint for the domain name specified in
   SvcDomainName (only for ServiceForm and otherwise empty).
   Described in {{svcfieldvalue}}.

Cooperating DNS recursive resolvers will perform subsequent record
resolution (for SVCB, A, and AAAA records) and return them in the
Additional Section of the response.  Clients must either use responses
included in the additional section returned by the recursive resolver
or perform necessary SVCB, A, and AAAA record resolutions.  DNS
authoritative servers may attach in-bailiwick SVCB, A, AAAA, and CNAME
records in the Additional Section to responses for an SVCB query.

When in the ServiceForm, the SvcFieldValue of the SVCB RR
provides an extensible data model for describing network
endpoints that are authoritative for the origin, along with
parameters associated with each of these endpoints.

For the HTTPS use-case with the HTTPSSVC RR, there is also direct mapping
from the SvcDomainName and SvcFieldValue into
HTTP Alternative Services (Alt-Svc) entries {{!AltSvc=RFC7838}}.

Together, these components provide a toolkit that has proven useful
and effective in {{!AltSvc}} for informing a client of services for an
origin.  However, making use of an alternative service requires
contacting the origin server first.  This creates an obvious
performance cost when alternative service endpoints can only be
received via a connection to the origin service: for example, users
wait for a full HTTP connection initiation (multiple roundtrips)
before learning of an alternative service that is preferred by the
origin.  The first connection also publicly reveals the user's
intended destination to all entities along the network path.



## Parameter for ESNI

This document also defines a parameter for Encrypted SNI {{!ESNI}}
keys, both as a general SVCB parameter and also as Alt-Svc parameter
which the SVCB parameter can be mapped into:

* esnikeys ({{esnikeys}}): The ESNIKeys structure from Section 4.1 of {{!ESNI}}
  for use in encrypting the actual origin hostname
  in the TLS handshake.


## Terminology

For consistency with {{!AltSvc}}, we adopt the following definitions:

* An "origin" is an information source as in {{!RFC6454}}.
  For services other than HTTPS, the exact definition will
  need to be provided by the document mapping that service
  onto the SVCB RR.
* The "origin server" is the server that the client would reach when
  accessing the origin in the absence of the SVCB record
  or an HTTPS Alt-Svc.
* An "alternative service" is a different server that can serve the
  origin over a specified protocol.

For example within HTTPS, the origin consists of a scheme (typically
"https"), a host name, and a port (typically "443").

Additional DNS terminology intends to be consistent
with {{?DNSTerm=RFC8499}}.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they
appear in all capitals, as shown here.



# The SVCB record type

The SVCB DNS resource record (RR) type (RRTYPE ???)
is used to locate endpoints that can service an origin.
There is special handling for the case of "https" origins.
The presentation format of the record is:

 RRName TTL Class SVCB SvcFieldPriority \
                         SvcDomainName SvcFieldValue

SvcFieldPriority is a number in the range 0-65535,
SvcDomainName is a domain name,
and SvcFieldValue is a set of key=value pairs present for the ServiceForm.
The SvcFieldValue is but empty and not included for the AliasForm.

The algorithm for resolving SVCB records and associated
address records is specified in {{resolution}}.

## Parameter specification via ServiceFieldValue {#svcfieldvalue}

TODO: WRITE ME in normative detail.  [Along with cleaning up the wire
format definition.]

In ServiceForm, the SvcFieldValue contains key=value pairs.
Keys are IANA-registered SvcParamKeys ({{#svcparamregistry}})
with both a case-insensitive string representation
a numeric representation in the range 0-65535.
They should only contain characters from the ranges
"a"-"z", "0"-"9", and "-".

Values are in a format specific to the SvcParamKey.
Their definition should specify both their presentation format
and wire encoding (e.g., domain names, binary data, or numeric values).

The presentation format for SvcFieldValue is a whitespace-separated
list of the SvcParamKey string identifier followed by a "=".
This is then followed by a value in <character-string> format
(which may be surrounded by double quotes),
as defined in {{!RFC1035}} Section 5.1.

TODO: Better define the formal grammar.  Per Section 5.1 of rfc1035,
both tabs and spaces are valid whitespace separators.

Unknown key types can be represented in presentation
format as "keyNNNNN" where NNNNN is the numeric
value of the key type.  In presentation format unknown values
should be represented as a base64 string.


## SVCB RDATA Wire Format

The RDATA for the SVCB RR consists of:

* a 2 octet field for SvcFieldPriority as an integer in network
  byte order.  For AliasForm, SvcFieldPriority MUST be 0.
* the uncompressed SvcDomainName, represented as
  a sequence of length-prefixed labels as in Section 3.1 of {{!RFC1035}}.
* the SvcFieldValue byte string, consuming the remainder of the record
  (so smaller than 65535 octets and constrained by the RRDATA
  and DNS message sizes).

AliasForm is defined by SvcFieldValue being empty and not present.

When SvcFieldValue is non-empty (ServiceForm), it contains a list of
SvcParamKey=SvcParamValue pairs with length-prefixes for the SvcParamValues,
each of which contains:

* a 2 octet field containing the SvcParamKey type as an
  integer in network byte order.
* a 2 octet field containing the length of the SvcParamValue
  as an integer between 0 and 65535 in network byte order
  (but constrained by the RRDATA and DNS message sizes).
* an octet string of the length defined by the previous field.

TODO: specify handling of ambiguity (eg, duplicates, validation, etc).

TODO: decide if we want special handling for any SvcParamKey ranges?
For example: range for greasing; experimental range;
range-of-manditory-to-use-the-RR vs range of
ignore-just-param-if-unknown.


## RRNames {#rrnames}

In the case of the SVCB RR, an origin is translated into the RRName
in the following manner:

1. RRTypes other than SVCB can define additional behavior
   for translating origins to RRNames.
   See {{#httpsrrnames}} below for the HTTPSSVC behavior.

2. The RRName is represented by prefixing the port and
   scheme with "_", then concatenating them with the host name,
   resulting in a domain name like "_8004._examplescheme.api.example.com.".

3. When a prior CNAME or SVCB record has aliased to
   an SVCB record, RRName shall be the name of the alias target.

Note that none of these forms alter the TLS origin or authority.
For example, clients MUST continue to validate TLS certificate
hostnames based on the origin host.

As an example:

    _8443._foo.api.example.com. 2H IN SVCB 0 svc4.example.net.
    svc4.example.net.  2H  IN SVCB 3 svc4.example.net. alpn="bar" port="8004" \
                                       esnikeys="..."

would indicate that "foo://api.example.com:8443" is aliased
to use ALPN protocol "bar" service endpoints offered as "svc4.example.net" 
on port 8004.



## SvcRecordType

The SvcRecordType is implicit based on the presence of SvcFieldValue
and defines the form of the SVCB RR.  Within an SVCB RRSet,
all RRs must have the same SvcRecordType.

If an RRSet contains a record in AliasForm, the client MUST ignore
any records in the set with ServiceForm.

When SvcFieldValue is empty, the SVCB SvcRecordType is defined to be
in AliasForm.

When SvcFieldValue is empty, the SVCB SvcRecordType is defined to be
in ServiceForm.


## SVCB records: AliasForm

When SvcRecordType is AliasForm, the SVCB record is to be treated
similar to a CNAME alias pointing to the domain name specified in
SvcDomainName.  SVCB RRSets MUST only have a single resource
record in this form.  If multiple are present, clients or recursive
resolvers SHOULD pick one non-determinstically.

The common use-case for this form of the SVCB record is as an
alternative to CNAMEs at the zone apex where they are not allowed.
For example, if an operator of https://example.com wanted to
point HTTPS requests to a service operating at svc.example.net,
they would publish a record such as:

    example.com. 3600 IN SVCB 0 svc.example.net.

The SvcDomainName MUST point to a domain name that contains
another SVCB record and/or address (AAAA and/or A) records.

Note that the RRName and the SvcDomainName MAY themselves be CNAMEs.
Clients and recursive resolvers MUST follow CNAMEs as normal.

Due to the risk of loops, clients and recursive resolvers MUST
implement loop detection.  Chains of consecutive SVCB and CNAME
records SHOULD be limited to (8?) prior to reaching terminal address
records.

The SvcFieldValue in this form MUST be empty (and not present
in presentation format).

As legacy clients will not know to use this record, service
operators will likely need to retain fallback AAAA and A records
alongside this SVCB record, although in a common case
the target of the SVCB record might have better performance, and
therefore would be preferable for clients implementing this specification
to use.

Note that SVCB AliasForm RRs do not alias to RRTypes other than
address records (AAAA and A), CNAMEs, and ServiceForm SVCB records.
For example, an AliasForm SVCB do not alias to an HTTPSSVC record, or
vice-versa.

## SVCB records: ServiceForm

When SvcRecordType is the ServiceForm, the combination of
SvcDomainName and SvcFieldValue parameters within each resource record
associates an alternative service and associated parameters with an origin.

For the HTTPSSVC RR, a direct mapping ({#map2altsvc}) is provided
below from HTTPSSVC RRs to Alt-Svc ({{!AltSvc}}) values.  Other protocols
leveraging SVCB directly may either specify that Alt-Svc should be used or should
specify their own semantics.

Clients MUST ignore SvcFieldValue parameters that they do not understand.

TODO: do we want to have a way to indicate the RR should be
ignored if a client does NOT understand a SvcFieldValue parameter?

This construction is intended to be extensible in two ways.  First,
any extensions that are made to the Alt-Svc format for transmission
over origin connections can be easily added as HTTPSSVC and/or SVCB parameters.

Second, by defining a way to map non-HTTPS schemes and non-default
ports ({{#rrnames}}), we provide a way for the SVCB to be used for
them as needed.  However, by using the origin name for the RRName for
scheme https and port 443 ({{#httpsrrnames}}) we allow HTTPSSVC
records to be included at the end of CNAME chains for existing site
implementations without requiring changes in the zone containing the
origin.

### Special handling of "." for SvcDomainName in ServiceForm {#svcdomainnamedot}

For SvcForm SRVB RRs, if SvcDomainName is has the value "."  then the
RRNAME for the final SVCB record MUST be used in-place of the
SvcDomainName.  (In the case of a CNAME or a AliasForm SVCB record
pointing to a ServiceForm SVCB record and SvcDomainName "."
then it is the RRNAME for the terminal SVCB record that must be
used as the effective SvcDomainName.)

### SvcFieldPriority  {#svcfieldpri}

As RRs within an RRSET are explicitly unordered collections, the
SvcFieldPriority value is introduced to indicate priority.
SVCB RRs with a smaller SvcFieldPriority value SHOULD be given
preference over RRs with a larger SvcFieldPriority value.

When receiving an RRSET containing multiple SVCB records with the
same SvcFieldPriority value, clients SHOULD apply a random shuffle within a
priority level to the records before using them, to ensure randomized
load-balancing.


## General SvcParamKeys for use in SvcFieldValue

A few SvcParamKeys are defined here for general use.

### SvcParamKey: alpn

TODO: should we call this "alpn" or "proto"?  Alt-Svc calls it
protocol-id so perhaps "proto" is better and easier for most
people to understand?

This the "alpn" SvcParamKey defines the Application Layer Protocol
(ALPN, as defined in {{!RFC7301}) supported by this alternative service.

The presentation format of the SvcParamValue is a UTF-8 string
identifying the protocol.  The wire format of SvcParamValue
is the corresponding set of octet values identifying the protocol.

Clients MUST ignore SVCB RRs where the "alpn" SvcParamValue
is unknown or unsupported.

### SvcParamKey: port

This the "port" SvcParamKey defines the TCP or UDP port
that should be used to contact this alternative service.

The presentation format of the SvcParamValue is a numeric value
between 0 and 65535.  The wire format of the SvcParamValue
is the corresponding 2 octet numeric value in network byte order.

### SvcParamKey: esnikeys

The SvcParamKeys for ESNI are defined below in {{#esnikeys}}.

### SvcParamKey: ips

TODO: do we also want to include a way to include/inline
a list of A and AAAA values to optimize for when
SvcDomainName has not yet been resolved?
Or is this unneeded complexity?
The current proposal is to create a separate draft
for this purpose defining an "ips" SvcParamKey
that can be used as a hint while waiting on SvcDomainName.

# Client behaviors

## Client resolution {#resolution}

When attempting to resolve a name HOST, clients should follow in-order:

1. Issue parallel AAAA/A and SVCB queries for the name HOST.
   The answers for these may or may not include CNAME pointers
   before reaching one or more of these records.

2. If an SVCB record of AliasForm SvcRecordType is returned for HOST,
   clients should loop back to step 1 replacing HOST with SvcDomainName,
   subject to loop detection heuristics.

3. If one or more SVCB record of ServiceForm SvcRecordType is returned
   for HOST, clients should combine SvcDomainName and SvcFieldValues
   to construct a set of alternative services.  (For the HTTPS
   use-case, they are used to synthesize equivalent Alt-Svc Field
   Values.)  If one of these alternative services is selected to be
   used in a connection, the client will need to resolve AAAA and/or A
   records for SvcDomainName.

4. If only AAAA and/or A records are present for HOST (and no SVCB),
   clients should make a connection to one of the IP addresses
   contained in these records and proceed normally.

When selecting between AAAA and A records to use, clients may
use an approach such as {{!HappyEyeballsV2=RFC8305}}

Some possible optimizations are discussed in {{optimizations}}
to reduce latency impact in comparison to ordinary AAAA/A lookups.

## Constructing alternative services

Each SVCB RR with a ServiceForm SvcRecordType can be used
to construct an alternative service that candidate for clients
to consider using instead of the origin.  For the HTTPS use-case
(as well as any other services using Alt-Svc) this is covered
in {#map2altsvc}.  Other services may define their own specific
behaviors.

Non-normatively, the typical behavior will follow:

1. Construct the list of SVCB RRs and remove those
   with SvcParamKey="alpn" and where SvcParamValue
   is unknown or unsupported.

2. Sort the list by SvcFieldPriority.

3. For highest-priority SVCB RR (lowest numeric SvcFieldPriority),
   attempt to make a connection to the alternative service with an
   endpoint defined by the domain name contained within SvcDomainName
   while using the application protocol and port defined by parameters
   in the RR's SvcFieldValue.

4. Authenticate the connection using the name specified
   by the origin.

Combining this with Alt-Svc allows alternative services
provided over a connection to an authoritative origin
to override those received through DNS.


# DNS Server Behaviors

Recursive DNS servers SHOULD resolve SvcDomainName records and include
them in the Additional Section (along with any relevant CNAME
records).  For AliasForm, recursive DNS servers SHOULD attempt
to resolve and include A, AAAA, and SVCB records.  For
ServiceForm, recursive DNS servers SHOULD attempt to resolve and
include A and AAAA records.  

Authoritative DNS servers SHOULD return A, AAAA, and SVCB records (as
well as any relevant CNAME records) in the Additional Section for any
in-bailiwick SvcDomainNames.

DNS servers SHOULD treat the SvcParam portion of the SVCB RR
as opaque and SHOULD NOT try to alter their behavior based
on its contents.


# Performance optimizations {#optimizations}

For optimal performance (i.e. minimum connection setup time), clients
SHOULD issue address (AAAA and/or A) and SVCB queries
simultaneously, and SHOULD implement a client-side DNS cache.
With these optimizations in place, and conforming DNS servers,
using SVCB does not add network latency to connection setup.

A nonconforming recursive resolver might return an SVCB response with
a nonempty SvcDomainName, without the corresponding address records.  If
all the SVCB RRs in the response have nonempty SvcDomainName values,
and the client does not have address records for any of these values in
its DNS cache, the client SHOULD perform an additional address query for
the selected SvcDomainName.

The additional DNS query in this case introduces a delay.  To avoid
causing a delay for clients using a nonconforming recursive resolver,
domain owners SHOULD choose the SvcDomainName to be a name in the
origin hostname's CNAME chain if possible.  This will ensure that the required
address records are already present in the client's DNS cache as part of the
responses to the address queries that were issued in parallel.


# Using SVCB with HTTPS and HTTP {#https}

There is special handling for the HTTPS and HTTP use-cases,
and the HTTPSSVC RRType is defined as an instantiation
of SVCB, specific to the https and http schemes.
This handling includes a mapping from HTTPSSVC records 
directly into Alt-Svc entries.

As mentioned above, the SVCB wire format and presentation format are
identical to HTTPSSVC, and both share a SvcParamKey registry.  SVCB
semantics apply equally to HTTPSSVC unless specified otherwise.

The presence of a HTTPSSVC record for an HTTP or HTTPS service also
provides an indication that all resources are available over HTTPS, as
discussed in {{#hsts}}.  This also allows a HTTPSSVC RR to apply to
pre-existing HTTP scheme URLs but while ensuring that a secure and
authenticated HTTPS connection is still used.

For the HTTPS use-case, the HTTPSSVC RR extends the concept
introduced in the HTTP Alternative Services proposed standard
{{!AltSvc}}.  Alt-Svc defines:

* an extensible data model for describing alternative network endpoints
  that are authoritative for an origin
* the "Alt-Svc Field Value", a text format for representing this
  information
* standards for sending information in this format from a server to a
  client over HTTP/1.1 and HTTP/2.

Each ServiceForm HTTPSSVC RR provides a set of information that can be
mapped into an Alt-Svc Field Value.  A client receiving this
information during DNS resolution can skip the initial connection and
proceed directly to an alternative service.


## RRNames for HTTPSSVC record {#httpsrrnames}

The HTTPSSVC RR extends the behavior for determining
an RRName specified above in {{#rrnames}}.

In particular, if the scheme is "https" with port 443, or the scheme
is "http" and the port is 80, then the RRName for the HTTPSSVC RR is
equal to the origin host name.  In this case, when a prior CNAME or
HTTPSSVC record has aliased to an HTTPSSVC record, RRName shall continue to be
the name of the alias target.

For schemes and ports other than https with port 443 and http with port 80,
the port and scheme continue to be prefixed to the hostname
as described in {{#rrnames}}.

Note that none of these forms alter the HTTPS origin or authority.
For example, clients MUST continue to validate TLS certificate
hostnames based on the origin host.



## Mapping ServiceForm to Alt-Svc {#map2altsvc}

To construct an Alt-Svc Field Value (as defined in Section 4 of
{{!AltSvc}}) from a HTTPSSVC record:

* The SvcDomainName is mapped into the uri-host portion of alt-authority.
  (If SvcDomainName is ".", the special handling described in
  {{#svcdomainnamedot}} MUST be applied first.)

* The SvcParamValue of the "port" service parameter is mapped to the
  port portion of the alt-authority.

* The SvcParamValue of the "alpn" service parameter is mapped to the
  protocol-id.  This MUST follow the normalization and encoding
  requirements for protocol-id specified in {{!AltSvc}} Section 3.

* For HTTPSSVC parameters with defined mappings to Alt-Svc, each should be
  included as a parameter, typically as the SvcParamKey name
  "=" a defined encoding of the SvcParamValue.

For example, if the operator of https://www.example.com
intends to include an HTTP response header like

    Alt-Svc: h3="svc.example.net:8003"; ma=3600; foo=123, \
             h2="svc.example.net:8002"; ma=3600; foo=123

they could also publish an HTTPSSVC DNS RRSet like

    www.example.com. 3600 IN HTTPSSVC 2 svc.example.net. \
                                        alpn=h3 port=8003 foo=123
                             HTTPSSVC 3 svc.example.net. \
			                alpn=h2 port=8002 foo=123

Where "foo" is a hypothetical future HTTPSSVC and Alt-Svc parameter.

This data type can also be represented as an Unknown RR as described in
{{!RFC3597}}:

    www.example.com. 3600 IN TYPE??? \\# TBD:WRITEME


## Differences from Alt-Svc as transmitted over HTTP

Publishing an alternative services form HTTPSSVC record in DNS is intended
to be equivalent to transmitting the corresponing Alt-Svc value over
HTTPS, and receiving an HTTPSSVC record is intended to be equivalent to
receiving this field value over HTTPS.  However, there are some small
differences in the intended client and server behavior.

### Omitting Max Age and Persist

When publishing an HTTPSSVC record in DNS, server operators MUST omit the
"ma" parameter, which encodes the "max age" (i.e. expiration time) of
an Alt-Svc Field Value.  Instead, server operators SHOULD encode the
expiration time in the DNS TTL, and MUST NOT set a TTL longer than the
intended "max age".

When receiving an HTTPSSVC record, clients SHOULD synthesize a new "ma"
parameter from the DNS TTL if the resulting alt-value is being passed to
a subsystem that might employ caching.

When publishing an HTTPSSVC record, server operators MUST omit the
"persist" parameter, which indicates whether the client should use
this record on other network paths.  When receiving an HTTPSSVC record,
clients MUST discard any records that contain a "persist" flag.
Disabling persistence is important to prevent a local adversary in one
network from implanting a forged DNS record that allows them to
track users or hinder their connections after they leave that network.

TODO: we might be able to clean up the above if we don't
define ma or persist or clear HTTPSSVC parameters?

### Multiple records and preference ordering {#pri}

Server operators MAY publish multiple ServiceForm HTTPSSVC
records as an RRSET.  When converting a collection of alt-values
into an HTTPSSVC RRSET, the server operator MUST set the
overall TTL to a value no larger than the minimum
of the "max age" values (following Section 5.2 of {{!RFC2181}}).

Each RR MUST contain exactly one alt-value, as described
in Section 3 of {{!AltSvc}}.

TODO: do we need the above?  Clearing up rules on parameter
normalization may obviate it.

As discussed in {{#svcfieldpriority}}, HTTPSSVC RRs with
a smaller SvcFieldPriority value SHOULD be sorted ahead of and given
preference over RRs with a larger SvcFieldPriority value. 

Alt-values received via HTTPS SHOULD be preferred over any Alt-value
received via a HTTPSSVC DNS RRSET.



### Constructing Alt-Svc equivalent headers

TODO: this has some overlap with the less normative text above.

For a client to construct the equivalent of an Alt-Svc HTTP response header:

1. For each RR, construct an Alt-value with a template:
       PROTOCOLID="URIHOST:PORT"
   Filling into this template:
   * The SvcDomainName MUST be inserted as URIHOST.  If
     SvcDomainName is has the value "." then the RRNAME for the final
     HTTPSSVC record MUST be inserted as URIHOST.  (In the case of a
     CNAME or a HTTPSSVC AliasForm record pointing to an HTTPSSVC
     record with ServiceForm and SvcDomainName "." then it is
     the RRNAME for the terminal HTTPSSVC record that must be inserted as
     the uri-host.)  The trailing "." MUST NOT be included in URIHOST.
   * The SvcParamValue of the "port" service parameter MUST be inserted
     as the value PORT as a numeric integer.
   * The SvcParamValue of the "alpn" service parameter MUST be inserted
     as PROTOCOLID.  This MUST follow the normalization and encoding
     requirements for protocol-id specified in {{!AltSvc}} Section 3.
   * Note that for the Alt-Svc use-case, the "port" and "alpn" HTTPSSVC
     parameters are mandatory and clients MUST ignore HTTPSSVC RRs missing
     either or both of these two parameters.
   * For HTTPSSVC parameters with defined mappings to Alt-Svc, each that
     us understood SHOULD be appended as a parameter,
     typically as the SvcParamKey name "=" a defined encoding
     of the SvcParamValue.  When present, these MUST be appended
     to the template following a literal "; ", and if multiple
     parameters are present then they must also be separated
     by literal "; " values.
2. The RRs SHOULD be ordered by increasing SvcFieldPriority, with shuffling
   for equal SvcFieldPriority values.  Clients MAY choose to further
   prioritize alt-values where address records are immediately
   available for the alt-value's SvcDomainName.
3. The client SHOULD concatenate the thus-transformed-and-ordered
   SvcFieldValues in the RRSET, separated by commas.  (This is
   semantically equivalent to receiving multiple Alt-Svc HTTP response
   headers, according to Section 3.2.2 of {{?HTTP=RFC7230}}).

### Granularity and lifetime control

Sending Alt-Svc over HTTP allows the server to tailor the Alt-Svc
Field Value specifically to the client.  When using an HTTPSSVC DNS
record, groups of clients will necessarily receive the same Alt-Svc
Field Value.  Therefore, this standard is not suitable for uses that
require single-client granularity in Alt-Svc.

Some DNS caching systems incorrectly extend the lifetime of DNS
records beyond the stated TTL.  Server operators MUST NOT rely on
HTTPSSVC records expiring on time, and MAY shorten the TTL to compensate.


### HTTP Strict Transport Security {#hsts}

By publishing an HTTPSSVC record, the server
operator indicates that all useful HTTP resources on that origin are
reachable over HTTPS, similar to HTTP Strict Transport Security
{{!HSTS=RFC6797}}.  When an HTTPSSVC record is present for an origin,
all "http" scheme requests for that origin SHOULD logically be redirected
to "https".

Prior to making an "http" scheme request, the client SHOULD perform a lookup
to determine if an HTTPSSVC record is available for that origin.  To do so,
the client SHOULD construct a corresponding "https" URL as follows:

1. Replace the "http" scheme with "https".

2. If the "http" URL explicitly specifies port 80, specify port 443.

3. Do not alter any other aspect of the URL.

This construction is equivalent to Section 8.3 of {{HSTS}} , point 5.

If an HTTPSSVC record is present for this "https" URL, the client
should treat this as the equivalent of receiving an HTTP "307 
Temporary Redirect" redirect to the "https" URL.
Because HTTPSSVC is received over an often insecure channel (DNS),
clients MUST NOT place any more trust in this signal than if they
had received a 307 redirect over cleartext HTTP.

If the HTTPSSVC query results in a SERVFAIL error, and the connection
between the client and the recursive resolver is cryptographically protected
(e.g. using TLS {{!RFC7858}} or HTTPS {{!RFC8484}}), the client SHOULD
abandon the connection attempt and display an error message.  A SERVFAIL
error can occur if the domain is DNSSEC-signed, the recursive resolver is
DNSSEC-validating, and an active attacker between the recursive resolver
and the authoritative DNS server is attempting to prevent the upgrade to
HTTPS.

Similarly, if the client enforces DNSSEC validation on A/AAAA
RRs, it SHOULD abandon the connection attempt if the HTTPSSVC RR fails
to validate.

### Cache interaction

If the client has an Alt-Svc cache, and a usable Alt-Svc value is
present in that cache, then the client SHOULD NOT issue an HTTPSSVC DNS
query.  Instead, the client SHOULD proceed with alternative service
connection as usual.

If the client has a cached Alt-Svc entry that is expiring, the
client MAY perform an HTTPSSVC query to refresh the entry.


# Extensions to enhance privacy


## Alt-Svc and SVCB/HTTPSSVC parameter for ESNI keys {#esnikeys}

Both SVCB/HTTPSSVC and Alt-Svc "esnikeys" parameters are defined for specifying
ESNI keys corresponding to an alternative service.
The value of the parameter is an ESNIKeys structure {{!ESNI}}
or the empty string.  ESNI-aware clients SHOULD prefer alt-values
and SVCB/HTTPSSVC RRs with non-empty esnikeys.

Both the SVCB SvcParamValue presentation format as well
as the Alt-Svc parameter value is the ESNIKeys structure {{!ESNI}}
encoded in {{!base64=RFC4648}} or the empty string.
The SVCB SvcParamValue wire format is the octet string
containing the binary ESNIKeys structure.

This parameter MAY also be sent in Alt-Svc HTTP response
headers and HTTP/2 ALTSVC frames.



### Handling a mixture of alternatives not supporting esnikeys

The Alt-Svc specification states that "the client MAY fall back to using
the origin" in case of connection failure {{!AltSvc}}.  This behavior is
not suitable for ESNI, because fallback would negate the privacy benefits of
ESNI.

Accordingly, any connection attempt that uses ESNI MUST fall back only to
another alt-value that also has the esnikeys parameter.  If the parameter's
value is the empty string, the client SHOULD connect as it would in the
absence of any ESNIKeys information.

For example, suppose a server operator has two alternatives.  Alternative A
is reliably accessible but does not support ESNI.  Alternative B supports
ESNI but is not reliably accessible.  The server operator could include a
full esnikeys value in Alternative B, and mark Alternative A with esnikeys=""
to indicate that fallback from B to A is allowed.

Other clients and services implementing SVCB or HTTPSSVC with esnikeys
are encouraged to take a similar approach.


# Interaction with other standards

The purpose of this standard is to reduce connection latency and
improve user privacy.  Server operators implementing this standard
SHOULD also implement TLS 1.3 {{!RFC8446}} and OCSP Stapling
{{!RFC6066}}, both of which confer substantial performance and privacy
benefits when used in combination with SVCB records.

To realize the greatest privacy benefits, this proposal is intended for
use with a privacy-preserving DNS transport (like DNS over TLS
{{!RFC7858}} or DNS over HTTPS {{!RFC8484}}).
However, performance improvements, and some modest privacy improvements,
are possible without the use of those standards.

This RRType may be extended to support schemes other than "https" and
"http".  Any such scheme MUST have an entry under the SVCB RRType in
the IANA DNS Underscore Global Scoped Entry Registry
{{!Attrleaf=I-D.ietf-dnsop-attrleaf}}.  The scheme SHOULD have an
entry in the IANA URI Schemes Registry {{!RFC7595}}.  The scheme
SHOULD be one for which Alt-Svc is defined, unless another binding is
defined.



# Security Considerations

SVCB/HTTPSSVC RRs and Alt-Svc Field Values are intended for distribution over untrusted
channels, and clients are REQUIRED to verify that the alternative
service is authoritative for the origin (Section 2.1 of {{!AltSvc}}).
Therefore, DNSSEC signing and validation are OPTIONAL for publishing
and using SVCB and HTTPSSVC records.

TBD: expand this section in more detail.  In particular:
* Just as with {{!AltSvc}}, clients must validate the TLS server certificate
  against hostname associated with the origin.  Clients MUST NOT
  use the SvcDomainName as any part of the server TLS certificate validation.
* ...


# IANA Considerations

## New registry for Service Parameters {#svcparamregistry}

The "Service Binding (SVCB) Parameter Registry" defines the name space
for parameters, including string representations and numeric
SvcParamKey values.  This registry is shared with other RRTypes
extending SVCB, such as HTTPSSVC.

ACTION: create and include a reference to this registry.

### Procedure

A registration MUST include the following fields:

* Name: Service parameter key name
* SvcParamKey: Service parameter key numeric identifier (range 0-65535)
* Meaning: a short description
* Pointer to specification text

Values to be added to this name space require Expert Review (see
{{!RFC5226}}, Section 4.1).

### Initial contents

The "Service Binding (SVCB) Parameter Registry" shall initially
be populated with the registrations below:

| SvcParamKey | NAME        | Meaning                | Reference       |
| ----------- | ------      | ---------------------- | --------------- |
| 0           | key0        | Reserved               | (This document) |
| 1           | alpn        | ALPN for alternative   | (This document) |
|             |             | service                |                 |
| 2           | port        | Port for alternative   | (This document) |
|             |             | service                |                 |
| 3           | esnikeys    | ESNI keys literal      | (This document) |
| 65280-65534 | keyNNNNN    | Private Use            | (This document) |
| 65535       | key65535    | Reserved               | (This document) |

TODO: do we also want to reserve a range for greasing?


## Registry updates

Per {{?RFC6895}}, please add the following entry to the data type
range of the Resource Record (RR) TYPEs registry:

| TYPE     | Meaning                | Reference       |
| ------   | ---------------------- | --------------- |
| SVCB     | Service Location       | (This document) |
|          | and Parameter Binding  |                 |
| HTTPSSVC | HTTPS Service Location | (This document) |
|          | and Parameter Binding  |                 |


Per {{?Attrleaf}}, please add the following entries to the DNS Underscore
Global Scoped Entry Registry:

| RR TYPE   | _NODE NAME | Meaning           | Reference       |
| --------- | ---------- | ----------------- | --------------- |
| HTTPSSVC  | _https     | Alt-Svc for HTTPS | (This document) |
| HTTPSSVC  | _http      | Alt-Svc for HTTPS | (This document) |


Per {{?AltSvc}}, please add the following entries to the HTTP Alt-Svc
Parameter Registry:

| Alt-Svc Parameter | Meaning              | Reference       |
| ----------------- | -------------------- | --------------- |
| esnikeys          | Encrypted SNI keys   | (This document) |



# Acknowledgements and Related Proposals

There have been a wide range of proposed solutions over the years to
the "CNAME at the Zone Apex" challenge proposed.  These include
{{?I-D.draft-bellis-dnsop-http-record-00}},
{{?I-D.draft-ietf-dnsop-aname-03}}, and others.

Thank you to Ian Swett, Ralf Weber, Jon Reed, 
Martin Thompson, Lucas Pardue, Ilari Liusvaara,
Tim Wicinski, Tommy Pauly, Chris Wood,
and others for their feedback and suggestions on this draft.


--- back

# Additional examples

## Equivalence to Alt-Svc records

The following:

    www.example.com.  2H  IN CNAME   svc.example.net.
    example.com.      2H  IN HTTPSSVC 0 svc.example.net.
    svc.example.net.  2H  IN HTTPSSVC 2 svc3.example.net. \
                                        alpn=h3 port=8003 esnikeys="ABC..."
    svc.example.net.  2H  IN HTTPSSVC 3 . alpn=h2 port=8002 esnikeys="123..."
    esni.example.net. 12H IN TXT   "123..."

is equivalent to the Alt-Svc record:

    Alt-Svc: h3="svc3.example.net:8003"; esnikeys="ABC..."; ma=7200, \
             h2="svc.example.net:8002"; esnikeys="123..."; ma=7200

for the origins of both "https://www.example.com" and "https://example.com".

# Comparison with alternatives

The SVCB and HTTPSSVC records type closely resembles some existing
record types and proposals.  A complaint with all of the alternatives
is that web clients have seemed unenthusiastic about implementing
them.  The hope here is that by providing an extensible solution that
solves multiple problems we will overcome the inertia and have a path
to achieve client implementation.

## Differences from the SRV RRTYPE

An SRV record {{?RFC2782}} can perform a similar function to the SVCB record,
informing a client to look in a different location for a service.
However, there are several differences:

* SRV records are typically mandatory, whereas clients will always
  continue to function correctly without making use of Alt-Svc or SVCB.
* SRV records cannot instruct the client to switch or upgrade
  protocols, whereas Alt-Svc can signal such an upgrade (e.g. to
  HTTP/2).
* SRV records are not extensible, whereas SVCB and HTTPSSVC and
  Alt-Svc can be extended with
  new parameters.  For example, this is what allows the incorporation of
  ESNI keys in SVCB.
* Using SRV records would not allow a client to skip processing of the
  Alt-Svc information in a subsequent connection, so it does not confer
  a performance advantage.

## Differences from the proposed HTTP record

Unlike {{?I-D.draft-bellis-dnsop-http-record-00}}, this approach is
extensible to cover Alt-Svc and ESNIKeys use-cases.  Like that
proposal, this addresses the zone apex CNAME challenge.

Like that proposal it remains necessary to continue to include
address records at the zone apex for legacy clients.


## Differences from the proposed ANAME record

Unlike {{?I-D.draft-ietf-dnsop-aname-03}}, this approach is extensible to
cover Alt-Svc and ESNIKeys use-cases.  This approach also does not
require any changes or special handling on either authoritative or
master servers, beyond optionally returning in-bailiwick additional records.

Like that proposal, this addresses the zone apex CNAME challenge
for clients that implement this.

However with this SVCB proposal it remains necessary to continue
to include address records at the zone apex for legacy clients.
If deployment of this standard is successful, the number of legacy clients
will fall over time.  As the number of legacy clients declines, the operational
effort required to serve these users without the benefit of SVCB indirection
should fall.  Server operators can easily observe how much traffic reaches this
legacy endpoint, and may remove the apex's address records if the observed legacy
traffic has fallen to negligible levels.


## Differences from the proposed ESNI record

Unlike {{!ESNI}}, this approach is extensible and covers
the Alt-Svc case as well as addresses the zone apex CNAME challenge.

By using the Alt-Svc model we also provide a way to solve
the ESNI multi-CDN challenges in a general case.

Unlike ESNI, this is focused on the specific case of HTTPS,
although this approach could be extended for other protocols.
It also allows specifying ESNI keys for a specific port, not an entire host.

## SNI Alt-Svc parameter

Defining an Alt-Svc sni= parameter
(such as from {{!AltSvcSNI=I-D.bishop-httpbis-sni-altsvc}}) would
have provided some benefits to clients and servers not implementing ESNI,
such as for specifying that "_wildcard.example.com" could be sent as an SNI
value rather than the full name.  There is nothing precluding SVCB from
being used with an sni= parameter if one were to be defined, but it
is not included here to reduce scope, complexity, and additional potential
security and tracking risks.

# Design Considerations and Open Issues

This draft is intended to be a work-in-progress for discussion.
Many details are expected to change with subsequent refinement.
Some known issues or topics for discussion are listed below.

## Record Name

Naming is hard.  The "SVCB" and "HTTPSSVC" are proposed as placeholders
that are easy to search for and replace when a final
name is chosen.
Other names for this record might include B, ALTSVC,
HTTPS, HTTPSSRV, HTTPSSVC, SVCHTTPS, or something else.

## Applicability to other schemes

The focus of this record is on optimizing the common case of the
"https" scheme.  It is worth discussing whether this is a valid
assumption or if a more general solution is applicable.  Past efforts
to over-generalize have not met with broad success.

## Wire Format

Advice from experts in DNS wire format best practices would be greatly
appreciated to refine the proposed details, overall.

## Where to include Priority

The SvcFieldPriority could alternately be included as a pri= Alt-Svc attribute.
It wouldn't be applicable for Alt-Svc returned via HTTP, but it is also
not necessarily needed by DNS servers.  It is also not used for AliasForm RRs.
Regardless, having a series of sequential numeric
values in the textual representation has risk of user error, especially
as MX, SRV, and others all have their own variations here.

## Whether to include Weight

Some other similar mechanisms such as SRV have a weight in-addition
to priority.  That is excluded here for simplicity.  It could always be
added as an optional SVCB parameter.


# Change history

* draft-nygren-dnsop-svcb-httpssvc-00
    * Generalize to a SVCB record, with special-case
      handling for Alt-Svc and HTTPS separated out
      to dedicated sections.
    * Split out a separate HTTPSSVC record for
      the HTTPS use-case.
    * Remove the explicit SvcRecordType=[01] and instead
      make the AliasForm vs ServiceForm be implicit.
      This was based on feedback recommending against
      subtyping RRTYPE.
    * Remove one optimization.
    
* draft-nygren-httpbis-httpssvc-03
    * Change redirect type for HSTS-style behavior 
      from 302 to 307 to reduce ambiguities.
    
* draft-nygren-httpbis-httpssvc-02
    * Remove the redundant length fields from the wire format.
    * Define a SvcDomainName of "." for SvcRecordType=1 
      as being the HTTPSSVC RRNAME.
    * Replace "hq" with "h3".

* draft-nygren-httpbis-httpssvc-01
    * Fixes of record name.  Replace references to "HTTPSVC" with "HTTPSSVC".
    
* draft-nygren-httpbis-httpssvc-00
    * Initial version
    