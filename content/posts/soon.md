---
title: "How to run an Ethereum full node on AWS (post-merge)"
date: 2022-09-02T22:12:03.284Z
type: posts
draft: false
slug: "eth-node-post-merge"
category: "5 mins read"
tags: ["crypto"]
description: "How to run an Ethereum full node on AWS (post-merge)"
---

In preparation for the [merge](https://blog.ethereum.org/2022/08/24/mainnet-merge-announcement), which is coming up in just under two weeks, I thought it would be good to do a lil update how to run an Ethereum full node post-merge. If you run an existing full node and want to make sure the merge goes smoothly, **make sure to upgrade your clients (which I'll describe below) before September 6**. Even though the merge is happening in two weeks, this happens over the course of two phases: first the Bellatrix upgrade (which is a network upgrade), and second (Paris upgrade) the transition of the execution layer's consensus mechanism from proof-of-work to proof-of-stake. The Bellatrix upgrade, as you can check in the merge announcement, is scheduled for epoch 144896 which is ~ 11am UTC on September 6, so you should upgrade your clients before then!

### What's different in running a full node before and after the merge?

The main difference between running a full node in proof-of-work and proof-of-stake is that previously, we would only need to run one client - for example geth, erigon etc. After the merge, we will now need to run **both an execution client (geth, erigon etc.) which is responsible for executing transactions, storing state, validation & broadcasting transactions etc. and a consensus client (which runs Ethereum's proof of stake algorithm)**. Tl;dr, you need to run two pieces of software together, you can no longer run an execution engine like geth or erigon etc. on its own. You must run it with a consensus client of your choice (there are five different ones you can choose from) to keep track of the head of the chain (more [details here](https://ethereum.org/en/developers/docs/nodes-and-clients/)).

### Instructions

### Step 1

I'm going to assume you're already running an execution engine (like geth or erigon), if you don't already do so, you should start by following [these instructions](https://amirbolous.com/posts/eth-node/) in order to catch up to speed. Note that even if you do run an execution engine, you **must upgrade to the latest merge-ready versions if you don't already run those versions**. At the time of writing, that's `1.10.23` and higher for geth, check your version is up to date with the latest version and if not, update it. When you run your execution engine, it should display something along the lines of "configured for merge" (or at least it does this for geth, I can't speak for the other engines but I would be surprised if they do something differently) at the start if you're running a merge-compatible version of your execution client.

### Step 1b

Before you go any further, you should verify that a default JWT token was created by your execution engine. The execution and consensus client communicate securely with a JWT token that you need to pass in, you can create a [custom one](https://geth.ethereum.org/docs/interface/consensus-clients) if you'd like, but if you don't want to, starting the latest version of geth should have created a JWT token at `<datadir>/geth/jwtsecret` where `<datadir>` is the directory you're using for storing all of geth's data. If you're following from step 1, `<datadir>` was `/mnt/ebs/ether/` for us, so verify that there is a jwtsecret or if you open `vim /mnt/ebs/ether/geth/jwtsecret`, it has a token and is not empty. If the file does not exist, double check you're running the latest version of geth and try restarting it. Note if you didn't use my previous post to run your geth node, this may live at a different directory for you.

### Step 2

Assuming you have a fully running and synced execution engine (that is up to date with the head of the chain), you now need to choose a consensus client of your choice. There are five different ones to choose from:

-   [Lighthouse](https://github.com/sigp/lighthouse/releases/tag/v3.1.0)
-   [Lodestar](https://chainsafe.github.io/lodestar/install/source/)
-   [Nimbus](https://github.com/status-im/nimbus-eth2/releases/tag/v22.8.2)
-   [Prysm](https://github.com/prysmaticlabs/prysm/releases/tag/v3.0.0)
-   [Teku](https://github.com/ConsenSys/teku/releases)

In order to push for [client diversity](https://twitter.com/sproulM_/status/1564882712120291328?s=20&t=MrLhEaOwvlIR7PQMTRjt_A), I'd recommend running one of the less popular ones if you can. I will be using Lighthouse for the remainder of this tutorial but running it with a different consensus client will be near identical.

### Step 3

Once you've selected your client of choice, download the latest version. If you're running a linux server (you likely are), make sure to download the x86 architecture version (or whatever architecture configured your server with if it was different).

```
$ wget https://github.com/sigp/lighthouse/releases/download/v3.1.0/lighthouse-v3.1.0-x86_64-unknown-linux-gnu.tar.gz
$ tar -xzf lighthouse-v3.1.0-x86_64-unknown-linux-gnu.tar.gz
```

Lighthouse in particular extracts to a single binary, some of the other clients (like Nimbus) are slightly different and you may have to run them via a script provided in the extracted folder. Refer to your client of choice's docs and README in this case.

### Step 4

Now we need to run our consensus client. This is not a guide for running a consensus client if you're trying to stake, the steps for that are slightly different. Most consensus clients provide two main points of functionality - one for running a beacon node, which connects to the P2P network and verifies blocks (this is the only one you want if you're not staking), and one for running a validator client, which actually manages validators using data obtained from the beacon node.

Since I'm not staking with this setup, I'll only be running the beacon node. There are various ways to keep the client running reliably, in the last blog post I wrote kept geth running with `nohup`. If you want to do the same, the command for that is:

`nohup ./lighthouse beacon_node --http --network mainnet --execution-endpoint http://localhost:8551 --execution-jwt <datadir>/geth/jwtsecret --datadir <datadir>/lighthouse/`

So there are a couple of things going on here, so let's break it down real quick

-   we run the command with admin privileges (`sudo`)
-   we provide `mainnet` as our network since this is what we're running our consensus client for
-   we prove the `--http` flag so that we can locally query the beacon node via the API when we need to
-   we provide the `execution-endpoint` which is the endpoint our execution engine exposes for communication with the consensus client, this will typically be port `8551`. Note **do not provide port 8545** for this, that is your execution engine's port for RPC communication
-   we then need to provide our JWT secret, again if you're following along from the previous tutorial `<datadir>` is `/mnt/ebs/ether/`
-   finally we need to provide the data directory for our consensus client to actually store the data it uses. Note that if you don't provide this command, lighthouse defaults to storing the data at `~/.lighthouse/`. Since our setup (if you're following along from the previous tutorial), is storing all of the data in a separate EBS volume, we need to provide the path to that volume, which is `/mnt/ebs/` for our case

A word of warning, I wasn't able to run this process and have it redirect `stdout` to a custom log file (without the process dying after the ssh connection disconnects), so if you're running your execution engine client with `nohup`, this will put both outputs of the processes in one muddled file, which is not ideal.

An alternative and better way to keep the client running is using a `systemd` service, which is much more robust because you can declaratively specify various custom configs and it will auto-restart every time it fails. In terms of practically how to make this work, I

-   made a bash script that was an executable that started lighthouse, the contents of this `startLighthouse` file being:

```
#!/bin/bash

# note make sure you provide the path to where your lighthouse executable at /path/to

sudo /path/to/lighthouse beacon_node --network mainnet --execution-endpoint http://localhost:8551 --execution-jwt <datadir>/geth/jwtsecret --datadir <datadir>/lighthouse/ --http

```

(note make sure you make this an executable by running `sudo chmod +x startLighthouse`)

-   created a `lighthouse.service` file in `/etc/systemd/system/lighthouse.service`, I've provided the contents of my file below for reference, you may want to customize this further or change some of the paths based on where your executables are located

```
[Unit]
Description=lighthouse

[Service]
Type=simple
Restart=always
RestartSec=5s

# change /home/ec2-user/ to what directory you're working from
WorkingDirectory=/home/ec2-user/
ExecStart=/bin/sudo /home/ec2-user/startLighthouse

# make sure log directory exists and owned by syslog
PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /var/log/lighthouse
ExecStartPre=/bin/chmod 755 /var/log/lighthouse
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=lighthouse

[Install]
WantedBy=multi-user.target
```

-   to start this service, you'll then want to run

```
$ sudo systemctl daemon-reload
$ sudo systemctl start lighthouse
```

-   you can check the service is running with

```
$ sudo systemctl status lighthouse
```

-   if it says `active` then things are running and are looking good, if it says `exited` or `ended` then the process exited for one reason or another, double check all the file contents and paths you specified are correct.

-   Once you've ran the service, you can then tail the log file in `<datadir>/lighthouse/beacon/logs/beacon.log` to see what's going on when you need to.

Regardless of whether you use `nohup` or `systemd` to run the client, if you don't want to wait for the consensus client to fully sync from the start, you can do a sync from a trusted checkpoint, more info [here](https://lighthouse-book.sigmaprime.io/checkpoint-sync.html#automatic-checkpoint-sync)

And that's it!

If any of the instructions were not clear or something did not work, please [reach](https://twitter.com/amirbolous) out to me so I can fix it!
