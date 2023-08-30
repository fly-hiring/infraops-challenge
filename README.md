# InfraOps Engineer Hiring Project

Hello! This is a hiring project for our [InfraOps Engineer position](https://fly.io/jobs/infrastructure-ops-engineering/). If you apply, we’ll ask you to do this project so we can see how you work through a contrived (yet shockingly accurate) example of the type of things you’d do at Fly.io. Checkout the job post for the nitty gritty.

There are two parts to this challenge. One is debugging a network problem between two VMs, the other is debugging and writing a small script to fix a storage problem.

### Setup your Fly.io Account

You need a Fly.io account to apply for this position since you'll be deploying apps for these challenges. You don't need to be an expert to pass them, but knowing the product is a _massive_ advantage through our hiring process.

You will NOT have to spend money to do these challenges. We'll email you a magic link that'll let you signup without a credit card and shovel credits to your account so you have plenty left to spare after the challenges. Just use them for good, not evil. If for some reason your account doesn't have credits or requires a credit card, email us and we'll sort it out.

The [Fly.io speed run](https://fly.io/docs/speedrun/) is a good way to get started. Our docs and blog are also great ways to learn more.

## Challenge 1: Network Debugging

We spend a lot of time in Linux network plumbing. Linux networking has gotten deceptively complicated. We want to make sure you’re comfortable diving into a misconfiguration and figuring out what’s gone wrong. You should be comfortable with iproute2, network sysctls, relatively low-level TCP/IP, and WireGuard. 

For this, we've built a system that'll be ran in two apps: a simple HTTP server and a client that makes requests to it. The network is broken. Unbreak it! 

### Setup

Two apps need to be deployed to your Fly.io account using the `flyio/infra-network-challenge:latest` Docker image. One is named `infra-network-server-[some_name]` and the other is `infra-network-client-[some_name]`. The test script expects the prefix, so don't get clever there. 

The setup for these applications is much more annoying than a typical Fly app, because part of the challenge is a misconfigured image we're providing, and we want you to figure out how it's broken by spelunking in a running instance and not by reading the code in our repo. We'll walk you through installing them here, and you should feel free to ask us questions. Just know that this *isn't* how normal Fly users boot up applications!

They both need to be installed from the same docker image - `flyio/infra-network-challenge:latest`. 

`infra-network-server-*` needs to have a secret or ENV variable set for `TYPE=SERVER` to configure it correctly.

Example `flyctl` commands to setup:

```
UNIQUE_NAME=myname   # You can just use your name, or similar here.
flyctl apps create --no-config --org personal infra-network-client-${UNIQUE_NAME} 
flyctl apps create --no-config --org personal infra-network-server-${UNIQUE_NAME} 
flyctl secrets set -a infra-network-server-${UNIQUE_NAME} TYPE=SERVER
flyctl machines run flyio/infra-network-challenge:latest -a infra-network-client-${UNIQUE_NAME}
flyctl machines run flyio/infra-network-challenge:latest -a infra-network-server-${UNIQUE_NAME}
```

This will leave you with two apps running, `infra-network-client-${UNIQUE_NAME}` and `infra-network-server-${UNIQUE_NAME}`.

Next, configure your computer as a Wireguard peer for the Fly.io private network ([docs](https://fly.io/docs/reference/private-networking/#private-network-vpn)) and log in with SSH. You can use `flyctl wireguard create` to create a "real" WireGuard connection, and `flyctl ssh issue --agent` to add credentials to your agent to log in with normal SSH. The server is `infra-network-server-${UNIQUE_NAME}.internal` and the client is `infra-network-client-${UNIQUE_NAME}.internal`.

### Fixing the Glitch

At this point you have two apps that are configured incorrectly. The client is talking directly to the server over the [Fly.io private network (6pn)](https://fly.io/docs/reference/private-networking/). This is what you'd normally want to do on Fly.io, but this is a hiring challenge so we're going to do it the weird way. What we want is for the client to talk to the server using a WireGuard tunnel, which is set up (badly) on both instances.

There's a `test-connection.sh` script that tries to connect to the server from the client first over 6pn, and then over wireguard. The end result should be that connections over 6pn should be blocked, and connections over wireguard should work. The opposite is the case right now.

You are free to modify and install any tools you think might be required, and please make notes about what you've done and all of the stupid places we've broken stuff.
The 6pn addresses for this are the [fdaa::] addresses on `eth0` in each vm. 
WireGuard should be running over 6pn, using the `fdaa` addresses. That's fine! It's what we want. We just don't want to use the 6pn addresses directly for HTTP; we want to route them over a WireGuard connection running over our 6pn addresses.

Since what we're actually trying to do here is pretty simple, you might be tempted to simply reconfigure the applications and redeploy them from scratch with a known-good configuration. Do not succumb to this temptation! You have to fix our dumb configuration in the running VM.

There are at least 7 things wrong with these VMs. See how many issues you can find.

Once you cleaned up the mess we left you, write up a list of what you fixed in your notes.

## Challenge 2: Storage Debugging

Operating a large fleet of servers and VMs is part of the job and we provide block storage for our users. We want you to be familiar with LVM, device mapper, filesystems, and linux storage systems so we put together a little challenge to help us fix a hypothetical storage problem on one of our servers.

### Setup

You'll want to create an app for this in your account. We think `infra-storage-[some_name]` is pretty good so we're going to refer to things with that name in this doc, but you can name it whatever you'd like.

The setup is pretty straightforward using fly machines:

```
UNIQUE_NAME=myname    # You can just use your name, or similar here.
fly apps create --org personal infra-storage-${UNIQUE_NAME}
fly machines run flyio/infra-storage-challenge:latest -a infra-storage-${UNIQUE_NAME}
```

This will leave you with one machine running and you can connect using:

```
fly ssh console -a infra-storage-${UNIQUE_NAME}
```

### Challenge Instructions

The machine runs a really simple web service that serves objects out of a directory. Feel free to check it out by running something like `curl http://localhost:8000`, but the objective is really just to approximate a production service that relies on some disk storage.

We're pretending we have real disks in our LVM pool by using loopback devices on top of files in the `/media` directory. We're just creating them with [`fallocate -l 1800M /media/diskN.img`](https://man7.org/linux/man-pages/man1/fallocate.1.html) as 1800MB images.

```
root@6e82dd90c22398:/# ls -lh /media/disk0.img
-rw-r--r--  1 root root 1887436800 Aug 23 17:05 disk0.img
```

We ran out of space so we created a few more "disks" and added them to the pool so now there are 3, but things still aren't working quite right. Then we asked ChatGPT for help and are pretty certain we've managed to break things even more. So we need your help getting things healthy again, and helping ensure we don't run out of disk space as more data is created.

The objective:

1. Fix up the existing `/data` volume so it's usable.
2. Set up the `vg0` VG with four 1800MB disks, using loopback images.
3. Expand the `data0` LV to 4GB.
4. Write a tiny bit of software to automatically expand the volume by 500M any time it gets close to full.

On the host is a script that will help you test things as you make progress. You can run `test-storage.sh` and it will run some simple tests to ensure things are working.

Feel free to install tools you think might be required and change things around, but please keep notes about what you've done. Also write down any silly things you find that are broken. If you manage to break things irreparably, then you can just stop the machine, delete it, and create a new one. Keep a note if that happens so we know where things went sideways. We've broken things in production plenty of times and are more interested in how you solve problems.

What we're looking for at the end of this is:
- the notes you made along the way,
- the output from the test script,
- and a copy of your volume expanding script or config.

Try to pretend like this is a production system and minimize any downtime on the HTTP object service. The notes you're keeping are the sort of raw context we'd want to have if we were doing a retrospective after an outage.

Have fun!

# Submitting your work

We’ll invite you to a private GitHub repo based on this template. Do all of your work in the main branch. We only care about the end result. Don’t bother with PRs, branches, or spend time on tidy commits — we have software to help us review. Don’t force push over the initial commit -- we get cranky if we can't see a diff of your changes. When you’re ready, let us know and we’ll schedule it for review. We review submissions once a week. You’ll hear back from us no matter what by the end of the /following/ week, possibly sooner if you submit early in the week.

Remember, we are not timing you and there is no deadline. Work at your own pace. Reach out if you're stuck or have any questions. We want you to succeed and are here to help!
