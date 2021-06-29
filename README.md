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

Nomad generates logs and we want you to collect them. Specifically: we want your Nomad instance to include and ElasticSearch instance. You would never deploy a Dockerfile that runs both Nomad and ElasticSearch in the real world, but this is just a work-sample challenge. Hook Nomad’s logs up to ES. 

Run Kibana, too, so we can actually see the logs.

We do not want you to go nuts with this. There’s a second challenge and we can’t ask for more of your time than a typical job interview process would ask. So, instead of literally going nuts making this Nomad deployment “actually good, give us a bulleted list of things you’d explore doing to make it good. You can keep your writeup simple! We just want to know what stuff you’re thinking about.

We are not timing you on this challenge (we will eventually offer people jobs, so I guess there’s theoretically a time limit, but there’s no stopwatch). You can start it whenever you want. You can pick it up on Tuesday and put it down until Friday. It might take you a couple hours. **It should not take you days.** If it is, something is going wrong, and you should talk to us. 

You can start this whenever you’d like, though, again, we think you should also set up a call with us to talk about Fly before you sink time into stuff.

## Challenge 2: Network Debugging

This one is simple. 

We've designed a system that can be installed as a client and server. The network is broken. Unbreak it! 

We’ve spent a lot of time last year in Linux network plumbing. Linux networking has gotten deceptively complicated. We want to make sure you’re comfortable diving into a misconfiguration and figuring out what’s gone wrong. Be comfortable with iproute2, network sysctls, relatively low-level TCP/IP, and WireGuard. 

Here's how to get going:

### Setup

Two apps need to be installed on your account. One is `sre-network-server-[some_name]` and the other is `sre-network-client-[some_name]`. The initial part of the app name is used in the test script so is important (you could get things working with different names but it'll be harder)!

The setup for these applications is much more annoying than a typical Fly app, because part of the challenge is a misconfigured image we're providing, and we want you to figure out how it's broken by spelunking in a running instance and not by reading the code in our repo. We'll walk you through installing them here, and you should feel free to ask us questions. Just know that this *isn't* how normal Fly users boot up applications!

They both need to be installed from the same docker image - `flyio/sre-network-challenge:latest`. 

`sre-network-server-*` needs to have a secret set for `TYPE=SERVER` to configure it correctly.

Example flyctl commands to setup:

```
UNIQUE_NAME=myname   # You can just use your name, or similar here.
flyctl apps create --no-config --org personal sre-network-client-${UNIQUE_NAME} 
flyctl apps create --no-config --org personal sre-network-server-${UNIQUE_NAME} 
flyctl secrets set -a sre-network-server-${UNIQUE_NAME} TYPE=SERVER
flyctl deploy -i flyio/sre-network-challenge:latest -a sre-network-client-${UNIQUE_NAME}
flyctl deploy -i flyio/sre-network-challenge:latest -a sre-network-server-${UNIQUE_NAME}
```

This will leave you with two apps running, named `sre-network-client-${UNIQUE_NAME}` and `sre-network-server-${UNIQUE_NAME}`.

### SRE network work sample challenge instructions

Of these two apps, one (the server) runs a simple HTTP server. The other (the client) makes requests of it. 

Configure your computer as a Wireguard peer for the Fly private network [https://fly.io/docs/reference/privatenetwork/#private-network-vpn] and log in with SSH (you can use `flyctl wireguard create` to create a "real" WireGuard connection, and `flyctl ssh issue --agent` to add credentials to your agent to log in with normal SSH. The server is `sre-network-server-${UNIQUE_NAME}.internal` and the client is `sre-network-client-${UNIQUE_NAME}.internal`).

These apps are currently configured incorrectly!

What we want is for the client to talk to the server using a WireGuard tunnel, which is set up (badly) on both instances.

What's happening now is that the client is talking directly to the server, using our private 6PN networks [https://fly.io/docs/reference/privatenetwork/]. This is what you'd normally want to do on Fly! But it's not what we want to do for this challenge.

There is a script, `test-connection.sh` that tries to connect to the server from the client first over 6pn, and then over wireguard. The end result should be that connections over 6pn should be blocked, and connections over wireguard should work.  The opposite is the case right now.

You are free to modify and install any tools you think might be required, and please make notes about what you've done and all of the stupid places we've broken stuff.
The 6pn addresses for this are the [fdaa::] addresses on `eth0` in each vm. 
WireGuard should be running over 6pn, using the `fdaa` addresses. That's fine! It's what we want. We just don't want to use the 6pn addresses directly for HTTP; we want to route them over a WireGuard connection running over our 6pn addresses.

Since what we're actually trying to do here is pretty simple, you might be tempted to simply reconfigure the applications and redeploy them from scratch with a known-good configuration. Do not succumb to this temptation! You have to fix our dumb configuration. :)

Again: none of this is explicitly timed. If it’s getting out of hand for you, reach out to us. Ask lots of questions.

Have fun!
