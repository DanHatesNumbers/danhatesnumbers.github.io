---
layout: post
title: Building a Certificate Authority in Rust - Part 1
categories: QuickLime PKI
---

In this post, I'm going to be exploring the core functionality of the Certificate Authority (which I'm going to be calling Quicklime), explaining what I'm planning on using as a central data store and what questions I don't yet have answers to. If you've not already done so, I suggest reading the [first part of this series]({{ site.baseurl }}{% post_url 2017-10-21-BuildingACAInRustPart0 %}), where I describe why I want to build a Certificate Authority in the first place and the basic functionality I want to implement.

In the previous post, I alluded to needing some way of recording what certificates have been issued and revoked by Quicklime, but I didn't go into detail on how I was going to record that information. This store of information will be involved in all of Quicklime's operations: Issuance, Revocation, OCSP responses and Monitoring.

However, each of these operations have different data needs. Issuance, for example, is write-only and only needs to record details about certificates that have been issued. OCSP responses need to know that a certificate has both been issued and if it has also been revoked and is probably going to be read-only unless some form of caching is implemented.

I recently read a really cool, in-depth article on LinkedIn's Engineering blog on logs and how they fill a central role in building distributed systems, called "[The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)". In this context, the author isn't referring to application logs that would typically be used for debugging or service monitoring, but:
> "an append-only, totally-ordered sequence of records ordered by time."

This type of log is also known as a Write-Ahead Log, a Transaction Log or a Commit Log.

The operations of a Certificate Authority can be thought of as a number of events over time related to issuance and revocation. When framed like this, an append only ordered sequence of records feels like a fairly natural abstraction for the data I want to record. Having a central location that records events which other components of the system can use to create derived views of the raw data to facilitate processing makes it easier to compose the system out of a number of small, single purpose components.

This approach to data storage has another advantage, because any of the components used to serve clients for issuance and revocation are almost entirely stateless. Any components used for displaying and analysing data can be rebuilt by replaying the log, applying the specific transformations required by that component until it is up to date with the latest record in the log, meaning almost all of the components can be rebuilt without worrying about preserving their data.

It seems like the most widely used solution for this problem is Apache Kafka, which is mentioned in the LinkedIn Engineering blog post, so I think I'm going to try using that as my source of truth for Quicklime.

So now we have our primary data store, how are we going to populate it and transform the log records to support Quicklime's operations? Let's start by thinking about the data sources and sinks:

- Issuing a new certificate
- Renewing an existing certificate
- Revoking a certificate
- OCSP responses
- Monitoring certificate status

# Issuing a new certificate

The process of issuing a certificate can be boken down into several stages:

- Receiving a Certificate Signing Request
- Validating the Certificate Signing Request
- Generating and Signing the certificate
- Generating the Revocation Challenge
- Generating the Renewal Challenge
- Recording the issuance along with information required to revoke and renew a certificate
- Returning the signed certificate to the requestor

I've not yet decided whether to accept normal PKCS #10 format Certificate Signing Requests[^1], or a format closer to Cloudflare's CFSSL[^2] requests which are JSON objects, or both. I'm envisioning a Rust & Rocket powered HTTP API handling the full issuance process, making use of bindings to OpenSSL to handle the actual cryptographic operations and a crate to provide an interface to Apache Kafka to record the issuance event. I haven't yet decided exactly what information I'm going to record in Kafka when issuing a certificate, so that's something I still need to think about.

# Renewing an existing certificate

Although this is normally handled using exactly the same mechanism as a regular certificate issuance, I think it'd be cool to be able to see a chain of certificates over time bound to a service name. So if I've got three nodes providing a highly available service that have all requested certificates for the same domain name, when those certificates are approaching their expiry time, I think it'd be cool to be able to see which certificates have been successfully replaced and identify problems with the automated replacement process. This might be handled by recording an issuance event in Kafka with optional fields that allow a new certificate to be linked to a previously issued certificate, because both events will have a lot of overlap in the information that needs to be recorded.

Being able to link a certificate with the certificate it is replacing in this manner will require that the renewals service can quickly look up the details of an existing certificate. The reason for this is two-fold: firstly to validate that the certificate being replaced is a real certificate that Quicklime has issued. Secondly to verify that the requestor is in posession of the certificate they are requesting a replacement for. I'm thinking of implementing renewal by recording a specific random string and returning this to the requesting party and requiring a signed digest of that string in the renewal request.

The renewal service will probably be a Rust & Rocket powered HTTP API to handle the renewal process and writing data to Apache Kafka. There will need to be some sort of key-value store as a cache of data that has been transformed from Apache Kafka, which can be updated by a small Rust program subscribing to the event stream and updating the state in the cache.

# Revoking a certificate

Although my envisioned use case probably won't need to support revocation, I want to try and implement it to explore how the process works. My initial thoughts on how this would work would be similar to certificate replacement, but in order to support the case of having lost the certificate, I can't really require that the certificate holder present a signed digest of the random string generated at issuance. However, I think that revoking a certificate is a less sensitive operation than being able to request a certificate for a domain. So even though revocation by an unauthorised party would cause some administritive headaches, I'm willing to accept the tradeoff needed to support revoking a lost certificate.

The system to handle revocations will need to be able to query a list of all the issued certificates that are both still valid and have not been revoked, along with their corresponding revocation secret. It will also need to be able to record the revocation events in Apache Kafka, and be monitoring revocation events to be able to update it's cache of valid certificates. This will probably follow the same design as the Renewal service, using a Rust & Rocket powered HTTP API, a key-value store as a cache and a small Rust program to update the cache, unless I encounter any major issues during implementation.

# OCSP responses

For the OCSP responder, I'll be handling requests from clients that don't support the CertificateStatusRequest TLS extension, which requires that they do the OCSP check themselves, as well as requests from certificate holders who are performing the OCSP lookup periodically to provide stapled responses to clients who have supplied the CertificateStatusRequest extension in their TLS handshake. Thankfully both of these use cases can be handled by the same service using the same data. The OCSP responder will need a cache of all currently valid certificates that haven't expired with an optional cached signed timestamped OCSP response if an OCSP response has recently been requested. While the implementation of the OCSP responder will be similar to the Renewal and Revocation services, the HTTP API will be populating the cache with the OCSP responses it generates, with a small Rust program monitoring issuance and revocation events to maintain the certificates in the cache. Although scaling the OCSP responders to multiple nodes would mean nodes are unable to make use of cached responses generated by another node, I'd rather incurr what feels like a small inefficiency to reduce the complexity of having a shared central cache.

# Monitoring Certificate Status

Finally, the monitoring dashboard will need to know about all certificates that have been issued by Quicklime, as well as their current status. I'll probably implement this as a Single Page App using React & Redux, mostly because they're technologies we use at my workplace and becoming more familiar with them would be useful for my day job. Powering the dashboard will likely be another Rust & Rocket powered HTTP API, a key-value store cache and a small Rust program updating the cache from the Apache Kafka log.

# Supporting Capabilities and Open Questions

In the previous sections, I talked about the core operations of the Certificate Authority that I'm planning on building, but theres a whole heap of other things to consider, almost all of which I'm going to have to figure out as I go.

- **Deployment, Hosting and DNS**

    I'm hoping I can handle this using Kubernetes to orchestrate Docker containers and provide DNS for services. My Kubernetes cluster will be running on Fedora 26 VMs on VMWare Workstation if possible because it's what I'm familiar with.

- **Application Logging and Monitoring**

    This is more of a nice to have than a must have, but because I want to experiment with horizontal scaling, being able to determine if services are healthy and inspect logs to identify problems would be helpful. I have experience with New Relic and App Dynamics as monitoring platforms, but both of these are Software as a Service offerings, so I want to look into self-hosted options.

- **User Authentication and Authorization**

    In order to control access to the Certificate Authority, I'm going to need some way of handling Authentication and Authorization of both users and software clients. My initial idea for this is an LDAP server as a user store, with an OAuth 2.0 and OpenID Connect to handle delegated authorization once a user has authenticated against the LDAP server. However this might more complex than it currently seems.

- **An interface to Apache Kafka**

    All of the services involved in this system will require a way of communicating with Apache Kafka, so I need to find a good Rust crate to handle communicating with it.

- **A Key-Value store for caching**

    A lot of the services require a transformed representation of the Apache Kafka log and will have a Rust program monitoring for new records and updating this transformed representation. I think I'll probably end up using something like Redis or another easy to use Key-Value store to handle this requirement. I'll also need a Rust crate that will allow me to communicate with whatever Key-Value store I decide to use.

- **An approach to distributing intermediate CA Certificates**

    The services in the Issuance, Renewal and OCSP Reponder roles will all need access to signing certificates issued by the Root CA Certificate. I'm going to need a plan for generating these and getting them on to the instances running these services. I might be able to generate these manually and look into a way of distributing them using a secrets management service, which I'm hoping is built into Kubernetes.

- **Bootstraping TLS certificates for HTTP APIs**

    All of the services in this Certificate Authority will only be operating over TLS using certificates that chain to the Root CA Certificate I'll be generating. This is a bit of a chicken and egg situation, where the services that issue the certificates will need to use themselves to issue their own certificates. I'm not quite sure how to solve this yet.

- **Validation of Domain Ownership**

    In order to prevent a compromised container for being able to request arbitrary certificates, I will need a way of validating that a certificate signing request originated from a valid node that belongs to the service in the certificate request. Because I'll also be trusting the Root CA certificate on my personal devices and have no way of restricting the domains I trust the Root CA to issue for, I need to prevent compromised nodes from requesting certificates for non-cluster domains, such as Google. In public facing certificate authorities this domain ownership validation is typically done either through email, using a well known address (admin@domain for example) or the address in the domain's WHOIS record, or by publishing a DNS TXT record with a value specified by the Certificate Authority. I'm not sure yet what approach I'm going to take for validating that a node isn't issuing fradulent requests.

- **A hosted container registry**

    Because I'll be experiementing a lot while building the Dockerfiles for the containers used for each role, I don't want to be publishing each iteration to a public Docker registry like the Docker Hub. In order to allow the Kubernetes cluster to be able to pull the Docker images for each container without publishing them to Docker Hub, I'll need to set up a local container registry to host them.

- **Stateful Application Management**

    This is a particular concern for the Apache Kafka instance, but I don't know how Kubernetes handles persistant storage and I want to make sure that rebuilding the node hosting the Apache Kafka instance won't result in data loss. I don't anticipate this being a major problem to solve, but it'll be something I need to learn fairly early on.

- **Backups**

    In addition to handling the persistent storage needs of the Apache Kafka service, I also need to figure out a way of handling backups of its data in the event of a disaster recovery scenario.

- **Configuration Management**

    Each of the services is going to need access to configuration information so they know where to find their cache instance, what credentials to use to access it, any API keys they need to access other services, amongst other things. This is another problem that I don't expect to be especially difficult to solve, but I don't know how I'm going to tackle it yet.

- **Are there existing IETF standard for what I'm implementing?**

    What I'm trying to build certainly isn't new and there is lots of prior work on PKI and the protocols that make it work. The Internet Engineering Taskforce had a (now concluded) Working Group[^3] for PKI standard like OCSP, the format used for Certificate Signing Requests and certificate management protocols. Before embarking on any major development work, I want to research the existing IETF standards to determine if I can avoid reinventing the wheel and instead implement an existing, well thought out protocol.

Hopefully this post has provided a good in-depth analysis of how Quicklime is going to be architected and highlighted some of the outstanding design decisions. I'm sure there will be plenty more design and implementation decisions that I can't think of at this point in time, so I'll be sure to document them as I come across them and explain the rationale behind the decisions I make and the thought process I went through to arrive at those decisions.

[^1]: You can read more about PKCS #10 in the [IETF RFC 2986](https://tools.ietf.org/html/rfc2986)
[^2]: You can read more about what CFSSL is and how to use it to build a PKI setup in two blog posts on Cloudflare's blog: [Introducing CFSSL - CloudFlare's PKI toolkit](https://blog.cloudflare.com/introducing-cfssl/) and [How to build your own public key infrastructure](https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/)
[^3]: [IETF PKI Working Group](https://datatracker.ietf.org/wg/pkix/about/)