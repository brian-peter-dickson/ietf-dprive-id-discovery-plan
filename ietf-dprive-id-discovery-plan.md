---
title: Planning document for DNS resolver identity, discovery, privacy and security.
abbrev: DPRIVE ID-Discovery-plan
docname: draft-ietf-dprive-id-discovery-plan-00
category: info

ipr: trust200902
area: General
workgroup: DPRIVE
keyword: DNS, Encryption, Privacy 

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Dickson
    name: Brian Dickson
    organization: GoDaddy
    email: brian.peter.dickson@gmail.com


normative:

informative:

--- abstract

This document is a planning document to discuss IETF drafts (including requirements) for DNS resolver identity, discovery, privacy and security.

--- middle

# Introduction & Scope

The deployment models for DNS resolution as originally documented, included only stub resolvers (clients) and full resolvers. In addition, there was no particular guidance concerning the use of network address translation (NAT) or of private address space (RFC1918). Subsequent deployment experience now not only frequently includes both of these, but also the use of forwarders (entities which are not full resolvers, but act as both clients and servers in an intermediary role), and more complex topologies (including directed graphs which may not be trees and are not guaranteed to be acyclic).

New standards have arisen which would benefit from the ability to query resolvers and forwarders for their local topology, where the entities may not have globally unique IP addresses and may not have their own fully qualified domain names (FQDNs). The information that such entities may further want to publish could include their names, their IP addresses, their roles, the methods by which they can be contacted by clients, as well as Trust Anchors (TAs) for DNSSEC. The published information may be signed if such DNSSEC TAs are used.

This document intends to discuss the overall plan for a set of interrelated proposals (information, experimental, or standard track), requirements for those proposals, as well as preliminary content for those proposals. The proposals may be kept together in a follow-on document, or may be split into separate documents. The choices of WG may differ if these proposals are separated, depending on their relevance and scope. This may in turn lead to progress in investigating, developing and standardizing potential experimental methods of meeting those requirements or solidifying those proposals.

The motivation for this work is to facilitate discovery of actual full resolvers to which requests are forwarded, to enable the bypassing of intermediate entities when both possible and appropriate, and to enable selection of an appropriate subset according to the express policy goals of a given client. The methods are intended to support independent operation of intermediaries, incremental upgrades and deployments, and authentication of discovered information. The intent is to maximize the ability to achieve client privacy (e.g. via DoT or DoH), to verify the privacy status, to identify problems caused by incompatible network configurations (e.g. overlapping NAT addresses/scopes), and to at long last provide a "DNS traceroute" functionality of some manner.

# Document Work Via GitHub

The authors are working on this document via GitHub at https://github.com/brian-peter-dickson/ietf-dprive-id-discovery-plan. Feedback via pull requests and issues are invited there. 

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document also makes use of DNS Terminology defined in {{!RFC8499}}

# Threat Model and Problem Statement

Currently, protocols such as DoT provide encryption between the user's stub resolver and a recursive resolver. This potentially provides (1) protection from observation of end user DNS queries and responses, (2) protection from on-the-wire modification DNS queries or responses (including potentially forcing a downgrade to an unencrypted communication).

The problem is that the stub resolver is currently unable to distinguish between a forwarder and a resolver -- they both return the desired results when queries are sent appropriately. Thus, an encrypted channel to a forwarder may result in queries being forwarded over a non-encrypted channel, defeating the purpose of using encryption. 

## Problem Statements (paraphrased)

In order for a stub resolver to establish an encrypted connection to the actual (ultimate) resolver in a forwarder chain to a resolver, it must be possible to for the resolver and some or all of the forwarders to have unique names, to publish the information about their immediate topological neighbors in the forwarder chain, for the stub to validate the published information, and for the stub to integrate the component topologies into a complete picture of the available resolvers and protocols.

This requires:
* Server identity assignment: use a pre-existing (known() FQDN, or generate globally-unique locally-significant name
* Server identity discovery: use a global name with local-only significance to discovery the server identity (name)
* Authoritative signed publication of topologically-local information under the server's name (FQDN or generated name):
   * Identity of local server's name and address(es), and trust anchor(s)
      * Trust anchor may not be needed per se if an FQDN is used underneath a secure zone.
   * Function (resolver or forwarder)
   * Each upstream server's name and address(es)
   * Each upstream server's trust anchor(s) (if the upstream server does not have an FQDN in a signed zone)
   * Each upstream server's function
* Publication of a well-known relative DNAME to pass through queries to upstream server(s)
* Facilitation of incremental deployment

# Details, Analysis, and Explanations

The problem statements (above) and requirements (below) are informed by analysis on the goal(s), transitive aspects of methods available (including masking), and security goals (e.g. identity proofs).

## Main Goal and Related Problems

The main goal of this set of proposals is to enable a stub client to establish an encrypted connection to one of its current resolvers, i.e. a resolver used by the forwarders that the stub currently uses (with no configuration change on the stub client).

There are several assumptions about how an encrypted connection can get established, that require information not available to the stub client currently, and for which the discovery process itself can be problematic in non-trivial topologies (i.e. one or more forwarders).

Encrypted transport, such as DoT or DoH, presumes the use of TLS, which requires both the DNS resolution of a server name, and the validation of a TLS certificate received upon attempting a connection to the server at the address (from the DNS resolution step) using an SNI value of the server name.

Forwarders and resolvers do not technically require DNS names, and may not have globally unique addresses. Without either globally unique addresses or DNS names, provisioning TLS certificates via Certificate Authorities (CAs) is difficult or impossible.

The solution to this involves the assignment of globally-unique names (but with local-only signficance) to forwarders and resolvers which do not have names, and providing an in-band (in-DNS) method to discover a resolver's name. The in-band aspect is necessary for diagnostic procedures using existing DNS tools, as well as backwards compatibility. Use of a single well-known name allows for the discovery of the name of the first forwarder or resolver (i.e. forwarder or resolver with this new functionality). However, the immediate consequence of using a single well-known name, is that the stub client cannot also directly determine the name of any further forwarders or resolver, as it will be "masked" by the response generated by the first forwarder.

Having each (implementing) forwarder do its own discovery of its upstream forwarders/resolvers, and publishing that information, is necessary to enable the stub client to obtain the names of those resolvers.

The use of DNSSEC is required to assure the stub client that the results (of name discovery) have not been tampered with by on-path intermediaries. The use of TLSA records allows the servers to create and publish certificates for these locally-scoped names, and for the stub clients (and intermediate forwarders and resolvers) to validate the certificates.

The local trust anchors for these locally-scoped yet globally-unique DNS zones are necessary, but not sufficient, to bootstrap the security of the DNSSEC components of this proposal. There may be a desire, or possibly requirement, that additional methods are used for publishing these trust anchors, such as via DHCP or 802.1x or similar mechanisms, or via device configuration management systems where possible/appropriate (e.g. enterprises).

The use of DNAME is designed to enable a method of forwarder "pass-through" for obtaining DNS resource records from upstream forwarders or resolvers, if/when those servers cannot be contacted directly (such as if address/port blocking is implemented anywhere in the path, or if there is an address conflict across NAT boundaries in the path).

A query to FQDN.q.GRANDPARENT.q.PARENT.q.FORWARDER which is sent to FORWARDER, would first be resolved inside FORWARDER itself, for which FORWARDER is authoritative. The name "q.FORWARDER" would be a well-known DNAME to ".", which means that FORWARDER would first perform a QNAME rewrite to "FQDN.q.GRANDPARENT.q.PARENT.", and then attempt to resolve this by sending the request to its upstream, i.e. PARENT. This pattern would be repeated at PARENT, and then at GRANDPARENT, which ultimately would resolve "FQDN.", as either a forwarder or as a resolver.

If the FQDN in the previous example is relevant to validation of any information related to the UPSTREAMs of any of these forwarders, the DNSSEC signatures would be included along with the responses in the full chain of DNAME responses, allowing independent confirmation that the full chain of trust anchors and names and addresses is unmodified, modulo a bad actor providing the forwarder service.

The presumption is that the forwarder chain is unavoidably trusted, and specifically configured (or learned) by all the parties operating the intermediate servers. Incremental improvements to the trust anchor mechanisms (as previously mentioned) by each party, would further improve the trustworthiness of the chain. It is no worse than any preexisting trust relationships, and may provide protection against on-path adversaries in the transitive chain of forwarders (by using DNSSEC to validate results, and potentially by using encrypted transport between forwarders and upstreams).

Finally, it should be noted that all of the above is backwards compatible, and would pass through non-upgraded forwarders transparently, even if the DNS resolution done by forwarders was the only available path to the resolver(s), i.e. that direct connectivity being blocked is not an impediment to the discovery aspect of this proposal.

# Requirements

The requirements of different interested stakeholders are outlined below. 

## Mandatory Requirements
1. Each implementing party should be able to independently take incremental steps to meet requirements without the need for close coordination (e.g. loosely coupled) 
2. Use a secure transport protocol between a client and an upgraded server (forwarder or resolver)
3. Use a secure transport protocol between an upgraded forwarder and an upgraded server (forwarder or resolver)
4. The secure transport MUST only be established when referential integrity can be verified, MUST NOT have circular dependencies, and MUST be easily analyzed for diagnostic purposes. 
5. The upgraded server (forwarder or resolver) MUST have the option to specify their secure transport preferences (e.g. what specific protocols are supported). This SHALL include a method to publish a list of secure transport protocols (e.g. DoH, DoT and other future protocols not yet developed). In addition this SHALL include whether a secure transport protocol MUST always be used (non-downgradable) or whether a secure transport protocol MAY be used on an opportunistic (not strict) basis. 
6. The upgraded forwarder MUST have the option to vary their preferences on a server to server basis, due to the fact that individual upstream servers (forwarders or resolvers) may be operated independently, and may offer different transport protocols (encrypted or otherwise). 
7. The specification of secure transport preferences MUST be performed using the DNS and MUST NOT depend on non-DNS protocols.
8. For the secure transport, TLS 1.3 (or later versions) MUST be supported and downgrades from TLS 1.3 to prior versions MUST not occur.

## Optional Requirements
1. QNAME minimisation SHOULD be implemented in all steps of recursion 
2. DNSSEC validation SHOULD be performed
3. If upstream servers indicate that (1) multiple secure transport protocols are available or that (2) a secure transport and insecure transport are available, then per the recommendations in {{?RFC8305}} (aka Happy Eyeballs) a recursive server SHOULD initiate concurrent connections to available protocols. Consistent with Section 2 of {{?RFC8305}} this would be: (1) Initiation of asynchronous DNS queries to determine what transport protocols are reachable, (2) Sorting of resolved destination transport protocols, (3) Initiation of asynchronous connection attempts, and (4) Establishment of one connection, which cancels all other attempts. The local server SHOULD prefer secure transport over insecure transport among available transport protocols from step (2). The local server MAY abort the process prematurely once the first secure transport is confirmed as available.

# Operation

Each element of the overall topology (stub client, forwarders, resolver(s)) performs certain operations at particular times.

These typically are triggered by start-up, or by topological changes, or by periodic re-validation of current state (such as in response to TTL expiry on relevant resource record sets).

The minimum relevant deployment is an upgraded stub resolver, and at least one upgraded forwarder or resolver.

## Server Initialization

A server is either a resolver or a forwarder. A forwarder is both a client and a server, and both of the following steps are required. A resolver is not a client per se, and only the second step (authoritative zone publication) is performed.

### Client Initialization

This step is performed by forwarders. The forwarder must either learn the IP addresses of its upstream servers (e.g. via DHCP) or have those configured.

Once the addresses are known, the server does the following for each address:
1. Send a query for the well-known name "resolver-name.arpa" to the ip address
1. Use the name in the response RDATA as the publication point for the information to obtain:
 * Upstream server's name
 * Upstream server's address(es)
 * Upstream server's Trust Anchor (if applicable)
 * Upstream server's function

### Server Name and Trust Anchor Assignment

If the server has an FQDN, this is used as the server name.

Otherwise, the server MUST generate a globally unique, locally scoped name.

The generated name is a random 12-character name of type hex-32 (alphabet 0-9a-v), with TLD suffix of ".zz", which contains 60 bits of entropy. If the PNRG is properly seeded, collisions in generated labels are exceedingly unlikely.

A suitable method must be used for seeding the local PNRG in order to avoid predictable collisions among such assigned names. This includes obtaining adequate sources of entropy.

For example, "example1245.zz" would not be generated due to the 'x' character falling outside the alphabet used. However, "linenoise000.zz" would be a possible result of a randomly generated name using this method.

### Authoritative Data Publication

Every server will publish the following information. (Some of the information is only required if the server's function is "forwarder".)

1. The domain name "resolver-name.arpa" is published with RRTYPE <TBD> and RDATA of the server's name.
1. If the server's name is an FQDN, AND the FQDN is a zone cut, AND the zone is DNSSEC signed with secure delegation and contains all of the remaining mandatory information, that FQDN-named zone is used as-is.
1. Otherwise, the server's name is used as the name of the zone, is an island of security, and requires generation of DNSSEC public/private keys. The public key is published in the zone apex as a DNSKEY record, and the zone is signed with the private key.
1. The remaining data here is published under a zone of SERVER_NAME, e.g. "linenoise000.zz".
1. The admitedly-redundant apex RRTYPE <TBD> has RDATA of SERVER_NAME, e.g. "linenoise000.zz".
1. The server function has name "server-function", RRTYPE "TXT", and value of either "resolver" or "forwarder".
1. The server's address(es) are apex A and/or AAAA record(s).
1. The trust anchor is an apex DNSKEY or DNSKEYs (plural if both KSK and ZSK are used).
1. A well-known DNAME record of name "q" with RDATA of "." is required.
1. If the server function is "forwarder", additional records with name "upstream", of type <TBD>, and values of the names obtained in the Client Initialization phase are added to the zone.
1. For each name in the list of "upstream", a set of records is created for the addresses, trust anchor(s), and function, with name of UPSTREAM_NAME (underneath the current zone SERVER_NAME, i.e. UPSTREAM_NAME.SERVER_NAME).
1. The types A, AAAA, and DNSKEY (or is it DS?) are at the exact name UPSTREAM_NAME.SERVER_NAME, while the function is at "function".UPSTREAM_NAME.SERVER_NAME.

## Stub Client Initialization

# Security Considerations

This entire document concerns the security of DNS traffic, so a specific section on security is superfluous. 

# IANA Considerations

This document has no actions for IANA. Yet.

# Changelog

Version 00: Intended for ongoing discussion prior to IETF 107 or in various WG email lists.

# APPENDIX: Perspectives and Use Cases

FIXME.

The DNS resolving process involves several entities.  These entities have different interests/requirements, and hence it does make sense to examine the interests of those entities separately - though in many cases their interests are aligned.  Four different entities can be identified, and their interests are described in the following sections:

- Users
- Operators
- Implementors / Software Developers
- Researchers

## The User Perspective and Use Cases

The privacy and confidentiality of Users (that is, users as in clients of recursive resolvers, which in turn forward/resolve the user's DNS requests by contacting authoritative servers) can be improved in several ways.  We call this "minimisation of exposure", and there are currently three ways to reduce that exposure:

  * Qname minimisation {{?RFC7816}}, reducing the amount of information to that which is absolutely necessary to resolve a query
  * Aggressive NSEC/local auth cache {{?RFC8198}}, reducing the amount of outgoing queries in the first place
  * Encryption, removing exposure of information while in transit 

As recursors typically forwards queries received from the user to authoritative servers.  This creates a transitive trust between the user and the recursor, as well as the authoritative server, since information created by the user is exposed to the authoritative server.  However, the user never has a chance to identify which data was exposed to which authoritative party (via which path).

Also, Users would want to be informed about the status of the connections which were made on their behalf, which adds a fourth point

Encryption/privacy status signaling

**TODO**: Actual requirements - what do users "want"? Start below:

## The Operator Perspective and Use Cases

Operators of authoritative services have to provide stable and fast DNS services, and interact with a wide range of clients, not all of them authoritative servers. The operator side actually consists of two sides:

  * The "upstream" facing side of recursive resolvers
  * The "downstream" side of authoritative servers

Those two sides are typically operated by different entities, but many entities operate "both sides".  Even though that is discouraged (**TODO** source), the two sides might even be operated on the same nameserver.

  * Maybe different technical perspectives for operators
    * Intelligence (sharing information)
    * SLD popularity for marketing
  * Focus initially on Second Level Domains (SLDs) initially
    * Is there a difference for TLDs vs. SLDs from a "protocol" perspective?
  * Monitoring and aggregated data analysis
  * Signaling provisioning information
    * New record type for finding authoritative server key and authentication?  Use SRV?  (Being able to use different servers for serving up DNS-over-{TCP,UDP} vs DNS-over-TLS responses may be valuable.
    * Signal secure transport details (DNS-over-TLS, DNS-over-QUIC, EncryptedSNI, connectionless, etc.), perhaps in an extensible manner?  Minimize RTTs and reduce need for trials.
    * Large provider use cases where the NS names are out of bailiwick for the zone (e.g. small number of distinct NS records serving 100k+ zones)
  * EDNS client subnet (JL: Not sure ECS crosses the cost/benefit threshold to be included as a requirement and many CDNs that run auth servers will likely say ECS is quite operationally important)
  * Decide between TLS and connectionless (such as COSE-based messages)
  * Costs of TLS connection vs. connectionless
    * Technical solution, e.g. encryption of the DNS query, shouldn't enable an attack vector for DDoS or resource exhaustion. For example, only if the client uses DNS-over-TLS, the upstream query to the authoritative will be over DNS-over-TLS also.  If the client uses UDP, the resolver won't invest resources in DNS-over-TLS to prevent a potential resource exhaustion attack.
    * Reuse connection state (if any) and examine resumption considerations
    * Minimize server-side state (eg, with session tickets)
    * Need empirical studies on capacity, traffic, attack vectors
    * Evaluate impact on architecture and footprint expansion
    * Analyze optimal persistent connection time/time-out
    * Analyze optimal number of persistent connections recursive resolvers should maintain
    * Consider operational concerns with respect to capabilities signaling
    * Develop a profile that has operational advantages for operators

**TODO**: Actual requirements - what do operators “want”?

## The Implementor / Software Vendor Perspective and Use Cases

Implementer requirements follows requirements from user and operator perspectives:

  * Non-functional requirements, e.g. diversity of implementations
  * Horizontal vs. vertical scaling, for example similar to http servers
  * Use of DANE {{?RFC6698}} for authentication: strict vs. opportunistic
  * Incremental deployment
  * Cache reuse vs. downgrade?  Does the cache need to be partitioned?  When can an in-cache answer retrieved via cleartext be served encrypted to a recursive query?
  * (Use of TCP fast open) - but this might be a requirement for the actual encryption protocol

**TODO**: Actual requirements of implementors - essentially, they follow what Operators need?

--- back

# Acknowledgments
{:numbered="false"}

TODO
