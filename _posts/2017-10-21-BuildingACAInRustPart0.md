---
layout: post
title: Building a Certificate Authority in Rust - Part 0
categories: QuickLime PKI
---

Recently, I've been playing with Rust and the Rocket crate to simultaneously learn some more Rust and also because I find working with HTTP services interesting. After spending a bit of time getting a Hello World running, I tried expanding it with a route that accepts a POST request with a JSON array of numbers and returns the sum of that array. I was struggling with getting Serde to work with a list of `i32`s, turns out I was using a slice when I should have been using a Vec. But once I'd got that figured out, I started thinking about what I could build that would be both non-trivial and that I would find interesting.

And that's how I ended up thinking that building a Certificate Authority based around a Rust HTTP API would be fun.

Some caveats about what I'm planning on building:

- It will NOT be production ready and should not be used in a production setting
- I'm going to be figuring stuff out as I go, so please don't take anything I write as best practise. I'll be writing about what worked for me
- There may be simpler or easier ways of achieving what I'm setting out to build, but this is a learning exercise

Now, with that out the way, what the heck am I actually trying to build?

When I was writing this post, I started working on an explanation of what a certificate is and how a Certificate Authority fits into the larger picture of PKI, but then I remembered that Julia Evans wrote a great post explaining everything I was going to try and explain, so instead of re-writing all of that, I suggest heading over to her blog and reading her post "[Dissecting an SSL certificate](https://jvns.ca/blog/2017/01/31/whats-tls/)". Even if you already understand what a Certificate Authority does and how Public Key Infrastructure works, check out the rest of Julia's blog!.

Julia's explanation of how certificates works talks about getting your certificate signed by a Certificate Authority that other computers trust. But what if you're in control of all the computers that will need to trust your certificates? In that case, it's possible to operate your own Certificate Authority.

In my case, I want to play around with running a CA to learn more about what's involved, with the overall goal of being able to build things for my local network that can provision their own certificates to communicate securely. I also want to build a non-trivial HTTP API using Rust and Rocket, which I can use to automate the process of issuing those certificates.

So thats the core of the idea, Certificate Authority based around a Rust and Rocket powered HTTP API, but it's not the whole plan. In addition to the core service, I want some way of recording what certificates have been issued and any certificates I've revoked. But that revocation information isn't much use without some way of querying it and there are two ways of doing that.

The first is to maintain a Certificate Revocation List, which is just a big list of all the certificates that your Certificate Authority has received revocation requests for, signed by the Certificate Authority that is valid for a short period of time. On a small scale, these work fine, but every time a client starts a TLS connection, it needs to check if the Certificate Revocation List has expired, download it if the cached copy has expired, and check if the certificate that has been presented has been revoked.

However Certificate Revocation Lists are being deprecated in favour of a second approach called Online Certificate Status Protocol, or OCSP. OCSP is a protocol that operates over HTTP that allows a client to query the Certificate Authority directly to determine if a certificate has been revoked. OCSP can also be used by a site operator to obtain a timestamped OCSP response signed by the Certificate Authority at regular intervals. This signed timestamped response is included in the TLS handshake if the client includes the Certificate Status Request extension when they start the TLS connection. This is known as OCSP Stapling and helps mitigate some privacy and performance issues with non-stapled OCSP.

In addition to the Certificate Authority API and an OCSP service, I'd also quite like some way of looking at all of the certificates that have been issued by the CA and any certificates that are due to expire or have been revoked in some form of web interface.

Finally, because all of the moving parts that this system will entail, I want a way of easily deploying and updating all of the pieces. That's where I think Kubernetes comes in. I've been hearing a lot about it over the last year or so and it sounded like it solves the problem of managing a lot of containers nicely. I've also heard that setting up a Kubernetes cluster is complicated, so I thought it would make for an interesting learning experience. If you've not heard of Kubernetes or aren't sure what it's for, Julia Evans (yep, the same Julia Evans I mentioned earlier) has written a great post called "[Reasons Kubernetes is cool](https://jvns.ca/blog/2017/10/05/reasons-kubernetes-is-cool/)".

So that, in a nutshell, is the plan. I'll go into more detail on the specific design for the whole system, the process of bootstrapping the Kubernetes cluster, and the process for building the system in future blog posts.