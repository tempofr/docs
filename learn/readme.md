---
title: Learn Overview
---
# The Stellar Ecosystem

![Stellar Ecosystem](https://www.stellar.org/wp-content/uploads/2015/08/ecosystem-overview-2.png)

The Stellar ecosystem is made up of several pieces of software. Depending on what you're doing with Stellar, you'll interact with or run different pieces of the ecosystem.

## [Stellar Core](https://github.com/stellar/stellar-core)
Stellar Core, or stellar-core, is the backbone of the Stellar network. It maintains a local copy of the ledger, communicating and staying in sync with other instances of stellar-core on the network. The Stellar network is made up of a collection of stellar-cores run by various individuals and entities all connected to each other. Stellar-core carries out the Stellar Consensus Protocol (SCP) and comes to consensus about the state of the network.

stellar-core accepts a limited number of [commands](https://github.com/stellar/stellar-core/blob/master/docs/learn/commands.md).

stellar-core writes its state out to a SQL DB that other applications can read to follow changes to the global [ledger](./concepts/ledger.md).

It can also be configured to send historical data to a `history store`. Every stellar-core needs to use *some* history store in order to catch up to the network, but not every stellar-core needs to write out its own history store.

## [Horizon](../horizon/learn/)
Horizon is the client-facing API server for the Stellar ecosystem. As the interface between stellar-core and applications that want to access the Stellar network, Horizon allows you to submit transactions to the network, check the status of accounts, subscribe to event streams, etc.

Horizon provides a RESTful API to allow client applications to interact with the Stellar network. You can communicate with Horizon using cURL or just your web browser. However, if you're building a client application, you'll likely want to use a [Stellar SDK](https://www.stellar.org/developers/horizon/learn/#libraries) in the language of your client.

## [SDKs](../horizon/learn/#libraries)
The SDKs facilitate communication between Horizon and a client application that is interacting with the Stellar network. They are responsible for crafting and signing transactions, submitting requests to Horizon, processing the responses, etc.

`stellar-core` <-> `horizon`  <-> `stellar-sdk` <-> `your client application`
