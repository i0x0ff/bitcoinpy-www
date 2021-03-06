---
title: "Overview"
weight: 1
---

These docs consists of a list of maintained and up-to-date Bitcoin Python libraries. Goal is to
provide a solid starting point for Python developers willing to work and interact with Bitcoin
protocol. Weather it's constructing transactions or fetching data from a Bitcoin RPC server.

### Running Bitcoin

Bitcoin node can be ran on several networks (mainet, testnet, signet, regtest). Under this section
you'll learn how to setup and run a Bitcoin node which you can then use as a "portal" into the
Bitcoin protocol. [Read more](/docs/running-bitcoin/)

### Libraries

#### python-bitcoinlib

Python3 library providing an interface to Bitcoin data structures and protocol. [Read more](/docs/python-bitcoinlib/)

Library supports **mainnet**, **testnet** and **regtest**. A PR for [signet](https://github.com/petertodd/python-bitcoinlib/pull/266)
has been merged but the version that supports it hasn't been released yet.

#### buidl

Python3 library with no dependencies. Provides an interface to Bitcoin data structures. There are
more functionalities included into this library which are for more advanced use and will be
described on this page at later stage. [Read more](/docs/buidl/)

Library only supports **mainnet**, **testnet** and **signet** and requires internet to work. 

#### bip_utils

This package allows generating mnemonics, seeds, private/public keys and addresses.

This unfortunatelly isn't a Bitcoin only library and includes supports for other cryptocurrencies.

[Read more](/docs/bip-utils/)