# Fly.io SRE Hiring Process

We don’t trust interviews; we think they select for people who are good at interviewing, and lock out a lot of people who don’t look shiny on paper. 

Our process is different from a lot of companies, and revolves around work sample problems; put simply: we’re going to ask you to let us put you in some scenarios that come up a bunch in operations work at Fly, and see how you handle them.

**Our Stack**

Both the role and the challenges will make more sense if you know a bit about our architecture.

Before we get too in the weeds, a suggestion: the best, fastest way to get a handle on what we’re up to at Fly is to try us out. If you’re interested in this role and haven’t already, you should sign up for a Fly account right now. Our signup will ask for a credit card number, because we’re a hosting provider and people constantly try to defraud hosting providers. Email us once you've signed up and we'll load you up with free credits.

The Fly.io speed run is a good way to get started:

https://fly.io/docs/speedrun/

You can skim the rest of this section, or refer back to it, or whatever.

The basic idea of Fly involves 3 different pieces.

Piece 1: we run a global CDN on our own hardware in racks around the world. The CDN uses BGP Anycast to direct Internet users to the closest rack. The heart of the CDN is a Rust-based proxy that handles HTTPS, HTTP, and TCP connections around routes them around our network.

Piece 2: we connect worker servers — beefy Intel servers -- to the CDN and feed them Docker containers (from a Docker registry we run) from our customers. The workers convert containers to Firecracker micro-vms. We boot up the VMs, connect them to the network, and tell the CDN to route traffic to them. The connectivity between the CDN and the worker servers occurs over WireGuard.

Piece 3: we host a GraphQL API that lets customers send us Docker containers and tell us how they want them run; which IP addresses to use, how certificates should work, how many instances to run, the geographic regions the instances should run in, what kinds of disks to connect, what ports to expose to the CDN. Most users drive this API with our `flyctl` command line interface.

Behind the scenes there’s a bunch of infrastructure that knits these three pieces together:

* We use HashiCorp Nomad to schedule VMs on worker servers. When you use `flyctl` to tell our API to run your instances in SYD and HKG, what our API is doing behind the scenes is dynamically building new Nomad constraints, and Nomad figures out where your jobs should run in response.
* We use HashiCorp Consul to propagate configuration across our fleet. For instance, when you ask us to start a new instance, and Nomad starts it, our Nomad driver registers a service in Consul that tells every machine in the fleet about the existence of that instance. Our DNS server looks for those Consul services as they’re registered in order to dynamically update the DNS. There’s a bunch of stuff that works this way.
* We run clustered Postgres for users, using Stolon, an open-source HA Postgres project, to manage clusters with read-replicas distributed around the world, using Consul.
* We use WireGuard to connect everything in our network together. WireGuard is a very fast, very secure, very low-level VPN fabric. We build a network on top of it with Python code and Consul. All of our internal routing is static.
* We use NATS, the messaging system, to propagate other kinds of updates across our network. NATS is faster and simpler than Consul, but makes fewer reliability promises.
* Part of our API is implemented in Go, and linked to that API server is a Docker Registry implementation; when you send us an app, what you’re really doing is pushing a Docker container to our registry, and our workers pull from it.
* If you’re booting up a Docker app but don’t have Docker running locally, or want to use buildpacks instead of Dockerfiles, we boot up little VMs (running on Fly like anything else) that know how to build and package apps for you. 
* We have a large ElasticSearch cluster that we funnel logs to (we primarily use Vector to get logs into ES). We have Kibana dashboards for logs from different parts of our service. We expose logs to our customers (for their apps) as a feature.
* We host a big VictoriaMetrics cluster — VictoriaMetrics is a clustered Prometheus server — to track metrics for everything in our system. We’ve got fine-grained metrics for almost everything that happens here. Metrics are our primary alerting system.

We’ve written a bunch of stuff about this. Some good posts to read, if you haven’t already:

https://fly.io/blog/docker-without-docker/ — our overall architecture.

https://fly.io/blog/persistent-storage-and-fast-remote-builds/ — how disks and builds work

https://fly.io/blog/measuring-fly/ — how metrics work

We’re happy to answer questions about all of this. The easiest way to do that is to set up a call, which is something we should do.

## Challenge 1: Deployment

We’d like you to run your own Nomad instance inside of Fly. Nomad in Nomad. It’s Nomads all the way down.

Specifically, what we’d like you to do is build a Dockerized deployment of a single-node development Nomad “cluster” (we don’t need multiple Nodes; the Nomad documentation tells you how to do this for dev/testing Nomad).

We’d like this Nomad instance to actually run something. We don’t care what; a shell script is fine. Don’t go nuts; we just want your Nomad instance to actually work.

Nomad generates logs and we want you to collect them. Specifically: we want your Nomad instance to include an ElasticSearch instance. You would never deploy a Dockerfile that runs both Nomad and ElasticSearch in the real world, but this is just a work-sample challenge. Hook Nomad’s logs up to ES. 

Run Kibana, too, so we can actually see the logs.

We do not want you to go nuts with this. There’s a second challenge and we can’t ask for more of your time than a typical job interview process would ask. So, instead of literally going nuts making this Nomad deployment “actually good, give us a bulleted list of things you’d explore doing to make it good. You can keep your writeup simple! We just want to know what stuff you’re thinking about.

We are not timing you on this challenge (we will eventually offer people jobs, so I guess there’s theoretically a time limit, but there’s no stopwatch). You can start it whenever you want. You can pick it up on Tuesday and put it down until Friday. It might take you a couple hours. **It should not take you days.** If it is, something is going wrong, and you should talk to us. 

You can start this whenever you’d like, though, again, we think you should also set up a call with us to talk about Fly before you sink time into stuff.

## Challenge 2: Network Debugging

This one is simple. 

When you’re done with the first challenge, we’re going to give you an SSH login to an app running on Fly. We’ll give you some documentation about what the app is trying to do, but the tl;dr is that it has semi-complicated network connectivity.

The network is broken. Unbreak it!

We’ve spent a lot of time last year in Linux network plumbing. Linux networking has gotten deceptively complicated. We want to make sure you’re comfortable diving into a misconfiguration and figuring out what’s gone wrong.

I’d like to write 6 more paragraphs about this but there isn’t much more to know going into it. Be comfortable with iproute2, network sysctls, relatively low-level TCP/IP, and WireGuard. 

Again: none of this is explicitly timed. If it’s getting out of hand for you, reach out to us. Ask lots of questions.
