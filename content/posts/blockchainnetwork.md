---
title: "How does a blockchain network work?"
date: 2025-08-15
draft: false
---

## Back Story

Hey, Are you curious? Curious enough to question things around you, Curious enough to dig deeper to it's core, Curious enough to learn more than other, Curious enough to break things and join them back.

My answer is I don't know, sometime I am and sometime I'm not and It is a story when I was one. I was looking to white paper of bitcoin and learning how is the network is self sustaining and reliable in itself. I was chatting with llm to know about it more what are the types of blockchain network architecture. This post is just about explain one of the architecture called mining pool setup.

## Introduction

In the mentioned architecture Blockchain network is divided in basically 4 parts and they are:

- Seed Server
- Full Node
- Miners
- User And Wallet

Below is the the diagram representation of it (please toggle theme to see the edges sorry for inconvenience)
![concept](/images/nodes.png)

Now let me explain each parts

- Seed Server:
  A seed server (sometimes called a DNS seed) is basically a special server that helps a new node find other peers when it first joins the network.So it keep track of node check for activeness.

- Full Node:
  Full node stores entire blockchain,it also maintain mempool of transactions that are to be mined.It is main component to participate in peertopeer networking,connects multiple peers shares tranactions broadcast if hash sent by miner.

- Miners
  Miner task is to find hash for transactions. Multiple miners compete for finding hash first after finding hash they broadcast it to connected nearby full node.

- User and Wallet
  It is a component to initiate transaction.

## Working

Let's talk about how these components works together:

- When a user initiate a transaction eg: A send 2btc to B,Wallet send that transaction to full node through http/websocket or tcp if wallet is also a node
- Now full node receives that transaction it validates it like if user A hash that btc or not and prepares transaction, it appends transaction to mempool and Gossips to all its' peers. Peers can be miners or other full nodes.
- If the receiving node is miner they go for finding the hash according to the difficulty of blockchain else if it is other full node it also appends transaction to its full node and regossips it to connected node this gossip is done in TCP.
- Let's assume miner1 found hash it immediately broadcast that hash to the connected near full node and full node rebroadcast to other connected nodes.
- Now on receiving hash full node verifies the hash.As it is **Hard to find,easy to verify**. Full node then appends that block with hash to rewards the miner with fee.

## Implementaion

I tried to implement this flow of every components as close I could get. Its' not anywhere close to perfection but I mean I tried my best. Golang is used for implemention. Every component is contenerized and every container have same externall network called blockchain-network so that each components can communicate with each other.

[![Repo Link](https://img.shields.io/badge/GitHub-Minor%20Project-blue?logo=github)](https://github.com/Aashish1-1-1/minor)
