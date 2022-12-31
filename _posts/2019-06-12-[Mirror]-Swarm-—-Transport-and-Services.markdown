---
layout: post
title:  "[Mirror] Swarm — Transport and Services"
date:   2019-12-31 10:36:43
categories: jekyll update
---

This post was originally published [here](https://medium.com/fair-data-society/swarm-services-and-protocols-892b4e72cf74)

In this post, I will talk about how Swarm’s base transport protocol is started and how other Swarm based sub-protocols are multiplexed on top of that.

![image](https://user-images.githubusercontent.com/940575/210152131-f6c66f63-30b6-426f-8f7a-7e030d73a753.png)

Note 1: To know more about the Swarm Architecture (introduction to components in the above picture) look at my old blog [here](https://jmozah.github.io/jekyll/update/2019/06/26/Mirror-Swarm-Architecture-a-view-from-above.html).


### Swarm and p2p
Swarm is a decentralized network, It means that a Swarm node does not speak with one centralized server, but speaks instead with a subset of peers which belong to the same network Id (NID). Every Swarm node itself has a unique node Id (SID) which is generated just before the node is started for the first time. This is nothing but a normal ethereum account. A swarm node can be described with a URL scheme called enode, similar to Ethereum. This is described in more detail [here](https://github.com/ethereum/wiki/wiki/enode-url-format).

In this blog post, I am going to introduce the Swarm transport protocol stack and other protocols that are arranged as services in the Swarm code.

### Transport Base Protocol (devp2p)
Before we go in to the other protocols that run on Swarm (including the peer discovery), I would like to briefly touch upon the transport architecture of Swarm. Today, Swarm uses [devp2p](https://github.com/ethereum/devp2p/blob/master/discv4.md) as the Transport layer; in the future, this may change to [libp2p](https://github.com/libp2p).

The entire Swarm node runs as an Ethereum [Node Service](https://github.com/ethereum/go-ethereum/blob/master/node/node.go) which takes care of [starting](https://github.com/ethereum/go-ethereum/blob/3bb6815fc1ade61e85b7039cf2e08a0871bb47e4/node/node.go#L162) and [stopping](https://github.com/ethereum/go-ethereum/blob/3bb6815fc1ade61e85b7039cf2e08a0871bb47e4/node/node.go#L429) of the service, its transport server and protocols running on top of them, and other menial things like its RPC, IPC, WS and HTTP Servers, etc.

The [Transport Server](https://github.com/ethereum/go-ethereum/blob/master/p2p/server.go) is responsible for managing peers. It accepts incoming connections from peers, and does the RLPx handshake (Diffie–Hellman) and the capability handshake. Once these handshakes are over, the channel is encrypted and devp2p takes over. Once the capabilities are known, a new go-routine spawns out for that peer and it manages its own lifecycle. Although the Transport Server manages the peer groups and takes care of common functions like RLPx encode/decode and message passing up, it does nothing more than that.

After the devp2p is established between two Swarm peers, they can start other Swarm protocols on top of that. These protocols are registered for every peer. The [Peer](https://github.com/ethereum/go-ethereum/blob/master/p2p/peer.go) once spawned does a few things:

- Sends out a Ping message every 15 seconds with its peer and will expect a Pong message
- Handle Incoming Messages: There are two kinds of messages that are handled. One responds to the regular Ping messages from its peers. The other is to receive the RLPX message from the peers, decodes it and passes it on to the respective protocol that is running on top of devp2p

Below is a summary of what was explained above. These are the messages that are exchanged as part of the devp2p transport protocol that is used by Swarm.

![image](https://user-images.githubusercontent.com/940575/210152325-36da7816-3992-4698-9462-ca63eb3c69e1.png)

![image](https://user-images.githubusercontent.com/940575/210152327-8684aff2-0e1d-4f4e-ad61-f568c9065746.png)


In future versions of devp2p, more messages are planned to be added, especially ENRRequest and ENRResponse. This is to negotiate a node’s requirements in the devp2p layer itself. Read more about that [here](https://github.com/ethereum/devp2p/blob/master/discv4.md#enrresponse-packet-0x06)

### Swarm Services
When the swarm node starts, it utilizes the swarm’s service provider interface to start the required services. Swarm is started as a node, that gets an explicit Start(), Stop() method in which Swarm can initialize itself. One of the important things that the Swarm service does is to start the required protocol services with its specifications (messages and its structure). Whenever a service is spawned, it registers itself with the devp2p layer so that any valid RLPx decoded incoming message there is passed over to its handler in the layer above it in the protocol stack. Below is a list of protocols that Swarm currently runs on top of devp2p.

![image](https://user-images.githubusercontent.com/940575/210152368-b6d9387c-303d-453e-afd7-f296bac9510b.png)


In the next few blogs, I will try to describe each protocol in the order shown in the above diagram. Stay tuned.

