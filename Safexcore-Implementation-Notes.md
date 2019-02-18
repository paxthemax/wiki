## Introduction

Safex is a open source blockchain project that is creating a purely peer-to-peer marketplace with an embedded cryptocurrency that maintains privacy, has an emission rate that is inclusive, is easy to obtain, and easy to use by a wide group of people. The Safex Team believes that this will enable mass adoption of decentralized Cryptocurrency.

More info about project:
* [Project website](https://safex.io/)
* [Project github](https://github.com/safex/safexcore)
* [Safex roadmap](https://safex.io/roadmap)
* [Safex blockchain explorer](http://explore.safex.io/)

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

![Block verification call stack](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/BlockVerification.png)

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

**Block batching**
If the database subclass, _BlockchainDB_, implements a batching method of caching blocks in RAM to be added to a backing store in groups, it should start a batch which will end either when _batch_num_blocks_ has been added or _batch_stop()_ has been called.  In either case, it should end the batch and write to its backing store.

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



### Alternative chains and chain switch

![Alternative chain check stack](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/AlternativeBlockVerification.png)

Often, blocks in the network make soft forks, due to competing mining. Goal of the nodes is to always be on the fork that has maximum difficulty (maximum of the mining work invested in them). 

For a new incoming block, this is the logic applied: if a block arrives and is to be added and its parent block is not the current main chain top block, then daemon needs to see if it knows about its parent block. If its parent block is part of a known forked chain, then daemon needs to see if that chain is long enough to become the main chain and reorganize accordingly if so.  If not, daemon needs to hang on to the block in case it becomes part of a long forked chain eventually.

If it is not applicable to the main chain, block is passed to the _Blockchain::handle_alternative_block_ function. There, following checks are performed:
* Block is checked for hard fork height, if it has expected block version and vote
* Search for block parent in alternative chain and in main chain
* Form alternative chain in form of the list, check its connection to main chain and complete timestamps vector list so that alternative chain timestamp check window is valid
* Check block timestamp
* Check if block is correct with respect to checkpoints
* Check block difficulty and work invested, with respect to _Blockchain::get_next_difficulty_for_alternative_chain()_
* Prevalidate miner transaction
* Add block to _Blockchain::m_alternative_chains_ list
* Check if alternative chain difficulty is bigger that the main chain. In case it is, switch to alternative chain with _Blockchain::switch_to_alternative_blockchain()_


List of alternative chain blocks is kept dynamically, in RAM, and is not saved to database permanent storage.



## Node RPC protocol

### ZMQ transport layer

ZeroMQ messaging system is partially integrated into safexd daemon. Currently, it is just server _ZmqServer_ implementing transport layer over TCP for existing RPC daemon handler. It is not yet implemented in client (simplewallet). ZMQ is meant to be utilized for faster communication between client(wallet) and safexd node, but it is yet to be implemented.

Classes _cryptonote::RpcHandler_ and _cryptonote::DaemonHandler_  are implementing handlers for RPC api exposed by _ZmqServer_.

## Node RPC API

![RPC request processing stack](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/RPCRequestHandle.png)

Node RCP functionality is implemented in _cryptonote::core_rpc_server_. Part of core rpc server is _Epee::net_utils::boosted_tcp_server_ which is started with _boosted_tcp_server<t_protocol_handler>::run_server _ command. It opens the worker threads listening for incoming RPC requests, and executes _net_utils::http::http_custom_handler_ for incoming http request. Handlers for particular http request are defined using handle http request map in _cryptonote::core_rpc_server_.

Structs (requests and responses) for rpc commands are defined in _core_rpc_server_commands_defs.h_ file. They are serializable using epee library serialization methods from _epee::serialization_ namespace. Each handler, upon receiving request and performing relevant checks, gets data from _cryptonote::core_ object and fills in response.

_Cryptonote::core_rpc_server_ defines multiple types of responses, using different macros for API endpoint definition: _MAP_URI_AUTO_BIN2_, _MAP_URI_AUTO_JON2_, _MAP_JON_RPC_. In binary case, _epee::serialization::store_t_to_binary_ is used for response body serialization, in case of JSON payload is serialized with  _epee::serialization::store_t_to_json_. Some of the API commands (e.g. _/getblocks_by_height.bin_) use binary representation, and some (_get_block_) use json representation.

Implementation of protocol buffer endpoints (to lower the communication overhead couple of times) is currently in progress.


## Node database

## Mining

## Hash function

## Difficulty

## Proof of work mining

Safex block production is based on Cryptonight v8 algorithm. Proof of work code/logic is mirroring the Monero original source code. The objective of the mining algorithm for the Safex Blockchain is to maintain access for mining by GPU and CPU and resisting ASIC (Application specific integrated circuit) production which limits the groups participation in the block production process. The mining of Safex Cash mission is to be as horizontal as possible and the mining algorithm will evolve to reflect that. Safex development team is planning to update mining algorithm with respect to Monero upstream changes.

## Miners


## Safex Cash and Safex Token

Cryptocurrencies based on the Cryptonote protocol normally have one native coin (e.g. Monero, Bytecoin...). The Safex Blockchain uses Safex Cash as coin that is a reward for block mining. It is used to pay transaction fees, and is currency used for commodity trading. On the other side there is Safex Token that is used as an instrument for special operations in the blockchain (creating accounts, title markets, could be locked to earn interest…). Both Safex Cash and Safex Token coexist in the blockchain, and one account has both a Safex Cash balance and a Safex Token balance.

Safex Cash replaces original Monero Coin and utilizes same source code/logic. Additional logic and support has been implemented for Safex Token operations.

Safex Token is migrated to the Safex Blockchain using migration transaction. After migration, it is kept as UTXO and is handled using the same logic as Safex cash. 

Comparison of Token and Cash

|   | Safex Cash | Safex Token | 
| ------------- | ------------- | ------------- |
| Number of Decimals  | 10  | 0 |
| Total supply  | 1 billion over 20 years using special curve 3 per block after 1 billion | 2,147,483,647 |
| Transaction output structure  |     struct txout_to_key { txout_to_key() { } txout_to_key(const crypto::public_key &_key) : key(_key) { } crypto::public_key key;};|     struct txout_token_to_key { txout_token_to_key() { } txout_token_to_key(const crypto::public_key &_key) : key(_key) { } crypto::public_key key; }; |
| Transaction types  | Coinbase transaction, Cash transaction, Migration transaction  | Migration transaction, Token Transaction |

Both Safex Cash and Safex Token are handled using 10 decimals, but Safex Token is undividable, e.g. token transactions could not send decimal amount of token.

To add support for tokens, the general strategy was to add parallel functions and structures, or attributes where applicable to the existing coin logic:
* Functions that implement token related operations and it was more practical to create new functions than to implement token logic in the current functions (e.g. _wallet::create_transactions_token_ in addition to _wallet::create_transactions_)
* Structures that handle tokens (e.g. _txin_token_to_key_ and _txout_token_to_key_ in addition to _txin_to_key_ and _txout_to_key_)
* Fields in data structures/objects that handle coins were usually extended with _uint64_t token_amount_, _bool token_transfer_ (or _token_transaction_) fields to implement support for tokens with minimum overhead and change of logic. If structure has amount field greater than zero, then it is Cash related, if it  has _token_amount>0_ and  _token_transfer==true_ then it is used for tokens
* Call stack of existing functions has often been extended with indicator about transaction class/purpose (_enum class tx_out_type_) to differentiate between tokens and cash transactions. (For same function that calculates total input/output amount returns sum of amounts or token_amounts based on indicator). More types will be added in the future as advanced concepts become introduced.
* Database keeps parallel bookkeeping of tokens in addition to coins (_m_cur_output_token_amounts_ cursor)



## Transactions

### Transaction version

Safex blockchain uses old Monero transaction with version 1 (pre RingCT, classic CryptoNote). Sub-addresses implementation works in command line wallet, and multisignatures are not supported.


### Genesis transaction

The Genesis transaction has two special tasks in Safex blockchain. First is to perform the airdrop of Safex cash, the second is to publish the migration verification keys as part of the transaction. It is generated with following function in _src/daemon/main.cpp_ file:

    void print_genesis_tx_hex(uint8_t nettype);

The function _print_genesis_tx_hex()_ generates 3 new accounts, saves their private keys to a file, it generates the initial Safex blockchain transaction with the airdrop amount to the first of those 3 accounts (migration account), and inserts public spend keys of those 3 accounts to the extradata of the transaction using the _add_migration_pub_keys_to_extra()_ function. The Safex genesis transaction performs an airdrop of 10 million Safex Cash coins (out of which 5 million assigned to the Safex team and 5 million which are distributed as part of the Bitcoin to Safex blockchain migration) to the migration account. The Migration account has the ability to sign migration transactions (described later in the document). Migration transactions could be verified by all nodes using the public migration key from the extra-data of the genesis transaction.


### Dynamic fee

Nominal fee is 0.01 Cash. Dynamic fee is calculated with formula:

    fee = (R/R0) * (M0/M) * F0

R: current base block reward
R0: reference base reward (60 Cash)
M: current median block size limit
M0: minimum block size limit (60000 from hard fork v2)
F0: 0.01 Cash


### Migration

Migrations is the most important feature of the milestone 0.1 . Current Safex OMNI Token  supply on the Bitcoin network will be migrated to the Safex blockchain.

Migration consist of 2 steps. First the user must send a special transaction to the Migration Burn Address which will set the Destination on the Safex Blockchain. Once the Safex Blockchain Address is set using the speciail transaction in step 1, then the user can send their Safe Exchange Coins which they want to migrate from the Bitcoin Blockchain. After the Address is set and the Coins are burned, the Address will be credited on the Safex Blockchain in exactly 1:1 Safe Exchange Coin for Safex Tokens and 440:1 Safex Cash.
Users are not actually converting their Safex Tokens but are receiving them in parallel.

A middleware will perform reading the Bitcoin blockchain for new burns for crediting users on the Safex blockchain. The Safex migration transaction will keep bitcoin Burn transaction tx hash as attribute and link to the previous Bitcoin token account, and will be signed with the special migration secret key. Other nodes can verify migration transaction inputs with the migration public key available in the genesis transaction. 

The Safex team has decided to perform migration in a centralized but transparent manner, as the simplest, shortest and most flexible way for token-holders. Special precautions and procedures must be taken to keep migration secret key safe. There are 3 possible migration verification public keys in the genesis transaction extradata (and matching secret keys), but only one is used and valid at any point in time. From the blockchain initiation, first migration key in the list will be used, and in unlikely case that it gets compromised, hard fork will be needed and the second migration verification key would become active.

Migration is currently in progress. As of February 2018, 650 million tokens is migrated to Safex blockchain.

### Migration transaction

Migration transaction is a special transaction generated by the migration account.

![Migration transaction](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/MigrationTransaction.png)

As part of the migration transaction, airdrop cash amount is also credited to the user address (airdrop token to cash rate is 0.00232830643 so that 5 million Safex Cash is distributed for all migrated tokens). Token input spends an imaginary output from Bitcoin blockchain burn transaction, and airdrop cash amount is transferred from the migration account balance.

Migration input key_image is used for the same purpose as cryptonote defined key_image, to prevent double spending. It has the value of the bitcoin burn transaction hash. That has implication that only one migration transaction could be generated per one bitcoin burn transaction.

Migration transaction creation is implemented/handled in the following functions (_src/wallet/wallet.cpp_):

    std::vector<wallet::pending_tx> wallet::create_transactions_migration(
       std::vector<cryptonote::tx_destination_entry> dsts,
       const crypto::hash bitcoin_transaction_hash, const uint64_t unlock_time,
       uint32_t priority, const std::vector<uint8_t> &extra, bool trusted_daemon);

    void wallet::transfer_migration(
       const std::vector<cryptonote::tx_destination_entry> &dsts, const crypto::hash bitcoin_transaction_hash,
       const size_t fake_outputs_count, const std::vector<size_t> &unused_transfers_indices,
       uint64_t unlock_time, uint64_t fee, const std::vector<uint8_t> &extra,
       cryptonote::transaction &tx, pending_tx &ptx, bool trusted_daemon)


Below is a JSON printout of example migration transaction with hash 811b499925ded19adb93cb011db7a4be68c9bec2d874e4422e44983de54f3827 that migrates 400 tokens and airdrops 0.92 Safex Cash to the Safex wallet SFXtzRzqWR2J3ytgxg1AxBfM8ZFgZmywoXHtqeqwsk3Gi63B2c3mvLNct35m268Pg2eGqHLmJubC7GPdvb1KxhTvHeVd4WKD9RQ. Owner of that wallet has burned his Bitcoin blockchain Safex OMNI tokens with bitcoin transaction 2d825e690c4cb904556285b74a6ce565f16ba9d2f09784a7e5be5f7cdb05ae1d. 

	{
	  "version": 1,
	  "unlock_time": 0,
	  "vin": [ {
		  "key": {
			"amount": 90000000000000000,
			"key_offsets": [ 0
			],
			"k_image": "dfe9314eb1d27c217a7f0a4c52e22b19de5da2761e6c0cfc56743833057d572d"
		  }
		}, {
		  "migration": {
			"token_amount": 4000000000000,
			"bitcoin_burn_transaction": "2d825e690c4cb904556285b74a6ce565f16ba9d2f09784a7e5be5f7cdb05ae1d",
			"k_image": "2d825e690c4cb904556285b74a6ce565f16ba9d2f09784a7e5be5f7cdb05ae1d"
		  }
		}
	  ],
	  "vout": [ {
		  "amount": 700000000,
		  "token_amount": 0,
		  "target": {
			"key": "cadef1027b599f1c985e3c829cd69eb3443e804ac6b62d1cac11d0b7ab01db66"
		  }
		}, {
		  "amount": 90000000000000,
		  "token_amount": 0,
		  "target": {
			"key": "118713832eb4b7a7cea994c128bfa0517fd0016533435cacd8bb099293b30c77"
		  }
		}, {
		  "amount": 0,
		  "token_amount": 4000000000000,
		  "target": {
			"key_token": "e832152c1f3f87f3f95e42782dff48487ef523c88251b8b993a14160294e833e"
		  }
		}, {
		  "amount": 9000000000000,
		  "token_amount": 0,
		  "target": {
			"key": "2a3d3fe5f02e6d2bd47db90cfa81ea3dd7a6aac0492bbb0e14dda884a4e06a2b"
		  }
		}, {
		  "amount": 80000000000000000,
		  "token_amount": 0,
		  "target": {
			"key": "9bd1890ef4f22d3032dd43180a31657cf05cbdf67d45f6bf878d2fb1363fda0a"
		  }
		}, {
		  "amount": 90000000000,
		  "token_amount": 0,
		  "target": {
			"key": "0e871f1c53a7f2f709c2ee5e3787175cec3a1e2ff5cd06d69abb5299c00f064e"
		  }
		}, {
		  "amount": 200000000,
		  "token_amount": 0,
		  "target": {
			"key": "267ea34ee8afcf7e80b9a3c466a4b5ad242f79765d2d2fc522d843fcababece2"
		  }
		}, {
		  "amount": 9000000000,
		  "token_amount": 0,
		  "target": {
			"key": "77e053281ff3e7f7ebee3bf626564040c8250067b098ccbe85186d4d48a740fe"
		  }
		}, {
		  "amount": 900000000000,
		  "token_amount": 0,
		  "target": {
			"key": "91b25d1078cb443ecd4071cc8b68dc24d38f5a90cae1d3116bdb16c14be1bc59"
		  }
		}, {
		  "amount": 9000000000000000,
		  "token_amount": 0,
		  "target": {
			"key": "e5f23de30b938b2b07819fe7de68ae3e94abe9a00d55c950548df37fc427d3c0"
		  }
		}, {
		  "amount": 900000000000000,
		  "token_amount": 0,
		  "target": {
			"key": "202e60428702f5784e8d1beb49c6bd1ea525ddec714a4aeb6184c5808eeb1997"
		  }
		}
	  ],
	  "extra": [ 1, 89, 118, 230, 239, 88, 64, 255, 186, 27, 177, 0, 252, 148, 198, 21, 142, 134, 211, 232, 142, 250, 199, 174, 169, 233, 134, 240, 123, 85, 255, 129, 117
	  ],
	  "signatures": [ "83e48ed2d867148299a3647873a2ffadc117e967e570ad7015e6a9d3f8c08201d2698bb8f315a9b2cc6b2f48421096e99ce0e766e7d9736ef3abb4ab09fb7307", "771af86a868bdd969c89979f00e933589865761ed14523eab55244c8e3dd3d0dffc23fa6705aa9758889aebf4973c0bf087daa7db6098ea33eaf4e7e074b1206"]
	}

### Token transaction

Token transaction moves tokens from one address to another. The sending account needs to have enough Safex Cash to pay for transfer fee.

![Token transaction](https://github.com/safex/wiki/blob/master/Safexcore_Implementation_Notes/TokenTransaction.png)

All rules, cryptography, signatures and logic that applies to the native Safex Coin transactions and UTXO outputs, also applies to token inputs and outputs. Separate bookkeeping of indexes of token outputs with same amount for ring formation/signatures is performed.

Token transaction creation is implemented in the following functions (_src/wallet/wallet.cpp_):
	std::vector<wallet::pending_tx> wallet::create_transactions_token(std::vector<cryptonote::tx_destination_entry> dsts, const size_t fake_outs_count, const uint64_t unlock_time, uint32_t priority,
	   const std::vector<uint8_t>& extra, uint32_t subaddr_account, std::set<uint32_t> subaddr_indices, bool trusted_daemon);

### Transaction verification

Transaction verification has been updated to handle new token related types of transactions. Node transaction verification functionality is implemented in the following functions (_src/cryptonote_core/blockchain.cpp_):

    bool Blockchain::check_tx_inputs(transaction& tx, tx_verification_context &tvc, uint64_t* pmax_used_block_height);

    void Blockchain::check_migration_signature(const crypto::hash &tx_prefix_hash, const crypto::signature &signature, uint64_t &result);

    bool Blockchain::check_tx_outputs(const transaction& tx, tx_verification_context &tvc);

### Transaction pool


## Wallets

### CLI Wallet

CLI wallet _safex-wallet-cli_ is the basic wallet implementation based on Monero CLI wallet. More advanced options inherited from Monero blockchain are disabled while others specific to Safex blockchain are enabled. _safex-advanced-wallet-cli_ which build is disabled by default for regular users has some additional functionality (migration transactions).

Menu options:

	[wallet SFXtzS]: help
	Commands:
	  account
		account new <label text with white spaces allowed>
		account switch <index>
		account label <index> <label text with white spaces allowed>
		account tag <tag_name> <account_index_1> [<account_index_2> ...]
		account untag <account_index_1> [<account_index_2> ...]
		account tag_description <tag_name> <description>
	  address [ new <label text with white spaces allowed> | all | <index_min> [<index_max>] | label <index> <label text with white spaces allowed>]
	  address_book [(add ((<address> [pid <id>])|<integrated address>) [<description possibly with whitespaces>])|(delete <index>)]
	  balance [detail]
	  balance_cash [detail]
	  balance_token [detail]
	  bc_height
	  check_tx_key <txid> <txkey> <address>
	  check_tx_proof <txid> <address> <signature_file> [<message>]
	  encrypted_seed
	  export_key_images <file>
	  export_outputs <file>
	  fee
	  get_description
	  get_tx_key <txid>
	  get_tx_note <txid>
	  get_tx_proof <txid> <address> [<message>]
	  help [<command>]
	  import_key_images <file>
	  import_outputs <file>
	  incoming_transfers [available|unavailable] [verbose] [index=<N1>[,<N2>[,...]]]
	  integrated_address [<payment_id> | <address>]
	  locked_transfer [index=<N1>[,<N2>,...]] [<priority>] [<ring_size>] <addr> <amount> <lockblocks> [<payment_id>]
	  password
	  payment_id
	  payments <PID_1> [<PID_2> ... <PID_N>]
	  print_ring <key_image> | <txid>
	  refresh
	  rescan_bc
	  rescan_spent
	  save
	  save_bc
	  save_watch_only
	  seed
	  set <option> [<value>]
	  set_daemon <host>[:<port>]
	  set_description [free text note]
	  set_log <level>|{+,-,}<categories>
	  set_tx_note <txid> [free text note]
	  show_transfer <txid>
	  show_transfers [in|out|pending|failed|pool] [index=<N1>[,<N2>,...]] [<min_height> [<max_height>]]
	  sign <file>
	  spendkey
	  status
	  sweep_unmixable [tokens]
	  transfer_cash [index=<N1>[,<N2>,...]] [<priority>] [<ring_size>] <address> <amount> [<payment_id>]
	  transfer_token [index=<N1>[,<N2>,...]] [<priority>] [<ring_size>] <address> <amount> [<payment_id>]
	  unspent_cash_outputs [index=<N1>[,<N2>,...]] [<min_amount> [<max_amount>]]
	  unspent_token_outputs [index=<N1>[,<N2>,...]] [<min_amount> [<max_amount>]]
	  verify <filename> <address> <signature>
	  version
	  viewkey
	  wallet_info


In first release of Safex blockchain, most significant changes in wallet compared to Monero CLI wallet are related to Safex token support (Addition/rename of _transfer_cash_, _transfer_token_, _unspent_cash_outputs_, _unspent_token_outputs_, _sweep_unmixable_ tokens, _balance_cash_, _balance_token_ commands).


### JSON Wallet

Safex JSON wallet rpc is based off of Monero JSON wallet rpc. The changes are related to the addition of Safex tokens. A new method has been added transfer_token. It behaves almost exactly like transfer, except it deals with tokens instead if cash. Token related information was added to responses to calls made to _get_balance_, _get_payments_, etc. 


