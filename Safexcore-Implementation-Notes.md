## Introduction

Safex is a open source blockchain project that is creating a purely peer-to-peer marketplace with an embedded cryptocurrency that maintains privacy, has an emission rate that is inclusive, is easy to obtain, and easy to use by a wide group of people. The Safex Team believes that this will enable mass adoption of decentralized Cryptocurrency.

More info about project:
* [Project website](https://safex.io/)
* [Project github](https://github.com/safex/safexcore)
* [Safex roadmap](https://safex.io/roadmap)

The purpose of this document is to organize the current Safexcore implementation and knowledge for the purpose of reference and review by the open source development community. The Safex Team would welcome any productive feedback from the community related to security/integrity/potential bugs and errors in the current implementation.

As of February 2019, Safex main network is on version 1.1.2, and hard fork version 3.

## Codebase

Safex has forked Monero codebase in April 2018. (between Monero releases v0.12.0.0 and v0.12.1.0) on commit 8fdf645397654956b74d6ddcd79f94bfa7bf2c5f. After fork, the Safex Team has implemented numerous changes (for first release about 20k lines of source code changes/insertions) to set up the foundation for the Safex Blockchain. We will outline the specific changes the team has made that should be reviewed by the open source development community.

## Safexcore Node Architecture
![Safexcore Arhitecture Diagram](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/CoreDiagram.png)


* _cryptonote::core_ - Safexcore node core implementation
* _cryptonote::t_cryptonote_protocol_handler_ - implement command handlers for blockchain p2p gossip protocol, commands like new_block, new_transaction, request_chan, then bc_protocol with functions relay_block, relay_transactions
* _nodetool::node_server_ - Implementation of the p2p endpoint with payload handler set as _t_cryptonote_protocol_handler<cryptonote::core>_
* _cryptonote::core_rpc_server_ - Implements http rpc endpoint for wallet communication with nodes




