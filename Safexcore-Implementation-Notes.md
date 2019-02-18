## Introduction

Safex is a open source blockchain project that is creating a purely peer-to-peer marketplace with an embedded cryptocurrency that maintains privacy, has an emission rate that is inclusive, is easy to obtain, and easy to use by a wide group of people. The Safex Team believes that this will enable mass adoption of decentralized Cryptocurrency.

More info about project:
* [Project website](https://safex.io/)
* [Project github](https://github.com/safex/safexcore)
* [Safex roadmap](https://safex.io/roadmap)

The purpose of this document is to organize the current Safexcore implementation and knowledge for the purpose of reference and review by the open source development community. The Safex Team would welcome any productive feedback from the community related to security/integrity/potential bugs and errors in the current implementation.

As of February 2019, Safex main network is on version 1.1.2, and hard fork version 3.

### Codebase

Safex has forked Monero codebase in April 2018. (between Monero releases v0.12.0.0 and v0.12.1.0) on commit 8fdf645397654956b74d6ddcd79f94bfa7bf2c5f. After fork, the Safex Team has implemented numerous changes (for first release about 20k lines of source code changes/insertions) to set up the foundation for the Safex Blockchain. We will outline the specific changes the team has made that should be reviewed by the open source development community.

## Safexcore Node Architecture
![Safexcore Arhitecture Diagram](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/CoreDiagram.png)


* _cryptonote::core_ - Safexcore node core implementation
* _cryptonote::t_cryptonote_protocol_handler_ - implement command handlers for blockchain p2p gossip protocol, commands like new_block, new_transaction, request_chan, then bc_protocol with functions relay_block, relay_transactions
* _nodetool::node_server_ - Implementation of the p2p endpoint with payload handler set as _t_cryptonote_protocol_handler<cryptonote::core>_
* _cryptonote::core_rpc_server_ - Implements http rpc endpoint for wallet communication with nodes


### Cryptonote::core

_Cryptonote::core_ class handles core cryptonote protocol functionality, it is defined in _cryptonote_core.h_ file. This class coordinates cryptonote functionality including communication among the Blockchain, transaction pool, any miners and the network. 


## Node p2p protocol

### Levin protocol


More low level Levin protocol discussion is available on [https://github.com/xmr-rs/xmr/blob/master/doc/levin.md](https://github.com/xmr-rs/xmr/blob/master/doc/levin.md), some info here is taken from that page.

The Levin network protocol is an implementation of peer-to-peer (P2P) communications found in a large number of cryptocurrencies, including all of the currencies that are forked from the CryptoNote project. The creator is a Russian programmer called Andrey N. Sabelnikov. A few different implementations of Levin are in existence. Safex uses Epee library implementation. This tcp protocol used to communicate with other nodes. It acts as both a server and a client.
Here are the fundamental concepts in Levin:
1. **Command**: a command used to talk to another peers, commands are identified by an unsigned integer (uint32_t), this usually starts at 1.000 (e:g: 1000 + 1).
2. **Invoke**: used to craft commands that need a response, you can think of it as a method, for example you can call a command called ADD with to integers as the input and receive the result.
3. **Notify**: used for commands that don't need a response, you can create a HELLO command without receiving a response from the peer.

Levin protocol is communicated using TCP. Packet starts with bucket header, that serves to identify Levin command from other stream data:

    struct bucket_head2
     {
         uint64_t m_signature;
         uint64_t m_cb;
         bool     m_have_to_return_data;
         uint32_t m_command;
         int32_t  m_return_code;
         uint32_t m_flags;
         uint32_t m_protocol_version;
    };

Where _m_signature_ is binary signature, _m_cb_ is size of message payload after the bucket head, _m_have_to_return_data_ is true for invoke commands and false for notify commands, _m_command_ is id of the command, _m_return_code_ is the response return code from the peer to the command request, _m_flags_ carries information if this message is request or response, and currently used _m_protocol_version_ protocol version is _LEVIN_PROTOCOL_VER_1_. For commands, format used Portable Storage is used and is able to serialize complex data structures. Portable Storage is implemented in _epee/include/storages/portable_storage.h_. Portable storage consist of **Sections**, which represent structure (like a key in a hash map), and **Storage Entry** which represent information itself.


## Transaction/block verifications

### Verification of incoming transactions

![Transaction verification call stack](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/TransactionVerification.png)

Upon receiving of _NOTIFY_NEW_TRANSACTIONS_ request message from other node using p2p message overlay,P2p payload handler _t_cryptonote_protocol_handler::handle_notify_new_transactions_ callback is executed. Following verifications are performed:
* Handler checks if p2p connection with the remote peer is in _cryptonote_connection_context::state_normal_
* local t_cryptonote_protocol_handler status is synchronized
* continue with processing binary transaction blob list from _NOTIFY_NEW_TRANSACTIONS_ request. One by one, transactions are forwarded to _cryptonote::core::handle_incoming_tx_ with verification context as return value. If verification of new transaction fails, connection with the remote peer is dropped, as peer node seems to be unreliable. _Handle_notify_new_transactions_ also keeps list of transactions that should be relayed. In case that some of the transactions from the incoming _NOTIFY_NEW_TRANSACTIONS_ list should be relayed further, at the end of the handler _t_cryptonote_protocol_handler<t_core>::relay_transactions_ is called. _Cryptonote::core_ transaction pool (class _cryptonote::tx_memory_pool_) keeps info if transaction from the txpool is relayed.

_Cryptonote::core::handle_incoming_tx_, _core::handle_incoming_txs_ implements handling process of incoming transactions. To increase speed of transaction verification, _core::handle_incoming_txs_ uses _tools::threadpool_ to process incoming transactions in parallel, calling in separate thread _core::handle_incoming_tx_pre_ to check each transaction: 
* transaction blob is checked for size
* blob is parsed with verification of integrity
* transaction is checked for transaction version
* is transaction on blacklist of bad transactions in _core::bad_semanticds_txes_
* check if transaction hash is already in the database of the transaction pool, current database of transactions in blocks
After all transactions are prechecked, it is, and again tread-pool is activated with _core::handle_incoming_tx_post_ parallel calls:
* check for syntax (currently trivially)
* Semantic check in core::check_tx_semantic
Transaction semantic check:
* check of number of inputs (must not be 0)
* check if input types are supported (must be _cryptonote::txin_to_key_, _cryptonote::txin_token_migration_ or _cryptonote::txin_token_to_key_)
* check if output types are valid (must be _cryptonote::txout_to_key_ or _cryptonote::txout_token_to_key_)
* checks for inputs and outputs overflow
* checks that sum of inputs is less or equal the sum of outputs (for both cash and tokens)
* check if transaction size is bigger that current block limit (in that case in is too big to enter the block)
* check if there are not duplicated key image in the utxo inputs of the transaction
* check for input keyimages domain (related to elliptic curve)
If all this checks are passed, transaction if forwarded to _core::add_new_tx_ call:
* additional check that transaction hash is not already in the transaction pool and transaction database
From there, _tx_memory_pool::add_tx_ is called:
* additional checks for type of tx inputs
* is transaction on the list of timed-out transactions
* is transaction fee larger than zero
* is fee properly calculated (_Blockchain::check_fee_)
* is transaction larger than transaction size limit
* are key images already in the txpool as spent
* check tx outputs (_Blockchain::check_tx_outputs_)
  * Is token whole amount
  * Is valid transaction output type (must be _cryptonote::txout_to_key_ or _cryptonote::txout_token_to_key_)
  * Check validity of out public key
  * don’t allow bulletproofs
* Check transaction inputs with _Blockchain::check_tx_inputs_
  * Check if minimum mixin is required (currently not used)
  * Check if key images in input are sorted
  * Check if all transaction input types are valid
  * Check if token amount is whole amount
  * Check that there is no more that maximum number of migrated tokens
  * Make sure that txin has valid key offsets
  * Check if input has key image as spent
  * Check every tx input with particular checking function for that type in _Blockchain::check_tx_input _(_Blockchain::check_tx_input_generic_, _Blockchain::check_tx_input_migration_)
  * Start thread pool to check ring signature or migration signature for relevant inputs with _Blockchain::check_ring_signature_ and _Blockchain::check_migration_signature_
  * Check if max_used_block_height is smaller then database height
* Special handling of transaction that are kept_by_block. This needs to be investigated more
If transaction has passed all this check, then it is included in the transaction pool database with _Blockchain::add_txpool_tx_.

    //**TODO**: add kept_by_block, max_used_block_height  explanation


### Verification of incoming blocks

[Incomings_blocks_verification](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/BlockVerification.png)

Upon receiving of _NOTIFY_NEW_BLOCK_ request message from other node using p2p message overlay, p2p payload handler t_cryptonote_protocol_handler::handle_notify_new_block callback is executed. NOTIFY_NEW_BLOCK request payload contains block blob and list of transaction blobs in the block. Following verifications are performed:

* Handler checks if p2p connection with the remote peer is in _cryptonote_connection_context::state_normal_
* local _t_cryptonote_protocol_handler_ status is synchronized
* Mining is paused
* Core is prepared to receive incoming blocks by taking lock for incoming transactions and calling _Blockchain::prepare_handle_incoming_blocks_ (check). 
  * Pool and blockchain locks are acquired.
  * Blocks are processed using threadpool and multiple batches
  * Check if block hash is already in db, alt chains or pool
  * Block is parsed and validated from blob
  * Thread with _Blockchain::block_longhash_worker_ is called
    * Block id is calculated with _cryptonote::get_block_hash_ and proof of work is calculated with _cryptonote::get_block_longhash_

* _Cryptonote::get_block_longhash_ is the function where cryptonight variant is selected (e.g. V7 (cryptonight version 1), V8 (cryptonight version 2). _Blockchain::m_blocks_longhash_table_ map keeps the list of block ids and their corresponding proof of works:
  * All transactions in the block are parsed for outputs, inputs, amounts
    * Check if tx prefix hash is already in _Blokchchain::m_scan_table_
    * Check for duplicated key image in one transaction
    * Parse amounts and makes list of input amounts
    * Keeps, sorts and filters list of absolut offsets from tx inputs
* Call _core::handle_incoming_tx_ for all incoming transactions in the block
* Call _core::handle_incoming_block_ for incoming block
  * Parse and validate block from blob
  * Call _core::add_new_block_ for block and then _Blockchain::add_new_block_ where we check we already know about block in database, alternative chains or invalid block list and then we handle block to be added to the main chain _Blockchain::handle_block_to_main_chain_  or alternative chain _Blockchain::handle_alternative_block_. Criteria for deciding if new block belongs to the main chain is previous block id, it should be same as last known main block id.
  * If new block is added, miner block template is also updated with _core::update_miner_block_template_

_Blockchain::handle_block_to_main_chain_ is important function that validates block and adds it to the end of the chain:
* Block _major_version_ is checked for current expected version and warning is issued if client is not updated but block of higher version has come
* Block is checked to be of current hard fork version
* Block timestamp is checked with _Blockchain::check_block_timestamp_. Block timestamp must not be larger than local time + block future time limit which is 500 seconds. Then, timestamps of last 12 blocks are collected and their median is found, and new block timestamp must not smaller than that median.
* Block proof of work is checked. New block proof of work calculated with _cryptonote::get_block_longhash_ is compared to expected difficulty calculated with _Blockchain::get_difficulty_for_next_block_
* Checkpoint for the block is checked, if in the checkpoint zone
* Prevalidate miner transaction for  one input of type _txin_gen_, height set to block height, correct unlock time, non-overflowing tx amount
* Check all other transactions:
  * Check if transaction is already in the database (_BlockchainDB->tx_exists()_)
  * Take transaction from transaction pool if it is available there. If it is not, fail verification and _Blockchain::handle_block_to_main_chain_
  * Call _Blockchain:check_tx_inputs_ for every transaction (function described in verification of incoming transactions chapter)
* Fully validate miner transaction with _Blockchain::validate_miner_transaction_
  * Check if miner tx outputs are decomposed correctly
  * Check if block size is bigger than 2*median with _cryptonote::get_block_reward_
  * Check if coinbase outputs are in total block reward + fee
  * Allow miner to take less reward if he wants too (to ignore dust outputs)
* If block verification has not failed, call _BlockchainDB::add_block_ with updated _already_generated_coins_ and _already_migrated_tokens_
  * This function checks if key images are already in the database and throw exception _KEY_IMAGE_EXISTS_
  * Add all block and tx related info to database

Blockchain class has list of invalid blocks that fail notification, and keeps dynamic list of them _Blockchain::m_invalid_blocks_.

Block batching: if the database subclass, _BlockchainDB_, implements a batching method of caching blocks in RAM to be added to a backing store in groups, it should start a batch which will end either when _batch_num_blocks_ has been added or _batch_stop()_ has been called.  In either case, it should end the batch and write to its backing store.

**Fluffy blocks**

Fluffy block is a concept where only block header with list of transactions is sent to the node,  and then node asks remote node to send all missing transactions, because many of them are already in the tx pool. Event is NOTIFY_NEW_FLUFFY_BLOCK, and handler is _cryptonote_protocol_handler::handle_notify_new_fluffy_block_. There, there is a logic that checks how many transactions node already knows off, and the rest of the transactions from the fluffy block list are requested from the remote node with _NOTIFY_REQUEST_FLUFFY_MISSING_TX_ request. Afterwards, processing of block works the same as for _NOTIFY_NEW_BLOCK_.



### Transaction Outputs 

Transaction outputs are kept in the database in the table with cursor _m_cur_output_amounts_. Output amounts are stored by value, a then kept in the list with incrementing global indexes (for the outputs with the same amount). They are indexed from the start of the blockchain history. When node or wallet operate with list of output indexes, only first value is absolute index, the rest indexes are relative difference (steps) to the previous index e.g. they're stored as offsets from the previous one (the first one from 0). This will result in smaller values for indexes, which can often result in a yet smaller amount of data, since those numbers are written out in a variable length output.

So, when referencing outputs to spend in the transaction:

    "vin": [ {
      "key": {
        "amount": 0, 
        "key_offsets": [ 17927, 10436, 804, 32, 3817
        ],
        "k_image": "67e33ecb9fc4e697248ef57ca88aa626fe670ce1551598f9cbc1565089d43c41"
       }}]

_Key_offsets_ are giving indexes (first one absolute, rest relative) that are easily referenced in the daemon database, as daemon node keeps all the output references from the beginning of the blockchain.































