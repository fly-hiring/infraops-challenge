# InfraOps Engineer Hiring Project

Hello! This is a hiring project for our [InfraOps Engineer position](https://fly.io/jobs/change-me/). If you apply, we’ll ask you to do this project so we can see how you work through a contrived (yet shockingly accurate) example of the type of things you’d do at Fly.io. Checkout the job post for the nitty gritty.

There are two parts to this challenge. One is to deploy a bite-sized version of several services we use into a Fly.io app, and the other is debugging a network problem between two VMs.

### Setup your Fly.io Account

You need a Fly.io account to apply for this position since you'll be deploying apps for these challenges. You don't need to be an expert to pass them, but knowing the product is a _massive_ advantage through our hiring process.

You will NOT have to spend money to do these challenges. We'll email you a magic link that'll let you signup without a credit card and shovel credits to your account so you have plenty left to spare after the challenges. Just use them for good, not evil. If for some reason your account doesn't have credits or requires a credit card, email us and we'll sort it out.

The [Fly.io speed run](https://fly.io/docs/speedrun/) is a good way to get started. Our docs and blog are also great ways to learn more.

## Challenge 1: Deployment

We’d like you to run your own [Hashicorp Nomad](https://nomadproject.io) instance inside of Fly.io.

Specifically, we want you to do 4 things:
1. Build a Dockerized deployment of a single-node development Nomad “cluster” (we don’t need multiple Nodes; the Nomad documentation tells you how to do this for dev/testing Nomad).
2. Run something in your Nomad cluster that generates logs. We don’t care what; a shell script is fine. 
3. Nomad and your test process generate logs, and we want you to collect them with an ElasticSearch instance alongside Nomad.
4. Run Kibana, too, so we can actually see the logs.

Remember: it's only a work sample, it doesn't have to be "good", it just needs to work. We know you wouldn't _actually_ run a bunch of single-node production services in the same VM, right?!? 

Along with your code, include a NOTES.md that goes over:
- A short summary of what you built, how it works, and the decisions you made to get there
- What you would do or explore if you were actually deploying this to production

## Challenge 2: Network Debugging

We spend a lot of time in Linux network plumbing. Linux networking has gotten deceptively complicated. We want to make sure you’re comfortable diving into a misconfiguration and figuring out what’s gone wrong. You should be comfortable with iproute2, network sysctls, relatively low-level TCP/IP, and WireGuard. 

For this, we've built a system that'll be ran in two apps: a simple HTTP server and a client that makes requests to it. The network is broken. Unbreak it! 

### Setup

Two apps need to be deployed to your Fly.io account using the `flyio/sre-network-challenge:latest` Docker image. One is named `sre-network-server-[some_name]` and the other is `sre-network-client-[some_name]`. The test script expects the prefix, so don't get clever there. 

The setup for these applications is much more annoying than a typical Fly app, because part of the challenge is a misconfigured image we're providing, and we want you to figure out how it's broken by spelunking in a running instance and not by reading the code in our repo. We'll walk you through installing them here, and you should feel free to ask us questions. Just know that this *isn't* how normal Fly users boot up applications!

They both need to be installed from the same docker image - `flyio/sre-network-challenge:latest`. 

`sre-network-server-*` needs to have a secret or ENV variable set for `TYPE=SERVER` to configure it correctly.

Example `flyctl` commands to setup:

```
UNIQUE_NAME=myname   # You can just use your name, or similar here.
flyctl apps create --org personal sre-network-client-${UNIQUE_NAME} 
flyctl apps create --org personal sre-network-server-${UNIQUE_NAME} 
flyctl deploy -i flyio/sre-network-challenge:latest -a sre-network-client-${UNIQUE_NAME}
flyctl deploy -i flyio/sre-network-challenge:latest -a sre-network-server-${UNIQUE_NAME} -e TYPE=SERVER
```

This will leave you with two apps running, `sre-network-client-${UNIQUE_NAME}` and `sre-network-server-${UNIQUE_NAME}`.

Next, configure your computer as a Wireguard peer for the Fly.io private network ([docs](https://fly.io/docs/reference/private-networking/#private-network-vpn)) and log in with SSH. You can use `flyctl wireguard create` to create a "real" WireGuard connection, and `flyctl ssh issue --agent` to add credentials to your agent to log in with normal SSH. The server is `sre-network-server-${UNIQUE_NAME}.internal` and the client is `sre-network-client-${UNIQUE_NAME}.internal`.

### Fixing the Glitch

At this point you have two apps that are configured incorrectly. The client is talking directly to the server over the [Fly.io private network](https://fly.io/docs/reference/private-networking/). This is what you'd normally want to do on Fly.io, but this is a hiring challenge so we're going to do it the weird way. What we want is for the client to talk to the server using a WireGuard tunnel, which is set up (badly) on both instances.

There's a `test-connection.sh` script that tries to connect to the server from the client first over 6pn, and then over wireguard. The end result should be that connections over 6pn should be blocked, and connections over wireguard should work.  The opposite is the case right now.

You are free to modify and install any tools you think might be required, and please make notes about what you've done and all of the stupid places we've broken stuff.
The 6pn addresses for this are the [fdaa::] addresses on `eth0` in each vm. 
WireGuard should be running over 6pn, using the `fdaa` addresses. That's fine! It's what we want. We just don't want to use the 6pn addresses directly for HTTP; we want to route them over a WireGuard connection running over our 6pn addresses.

Since what we're actually trying to do here is pretty simple, you might be tempted to simply reconfigure the applications and redeploy them from scratch with a known-good configuration. Do not succumb to this temptation! You have to fix our dumb configuration in the running VM.  

Once you cleaned up the mess we left you, write up a list of what you fixed in your notes.

# Submitting your work

We’ll invite you to a private GitHub repo based on this template. Do all of your work in the main branch. We only care about the end result. Don’t bother with PRs, branches, or spend time on tidy commits — we have software to help us review. Don’t force push over the initial commit -- we get cranky if we can't see a diff of your changes. When you’re ready, let us know and we’ll schedule it for review. We review submissions once a week. You’ll hear back from us no matter what by the end of the /following/ week, possibly sooner if you submit early in the week.

Remember, we are not timing you and there is no deadline. Work at your own pace. Reach out if you're stuck or have any questions. We want you to succeed and are here to help!
