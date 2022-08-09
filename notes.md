Server.py
This is the starting point of the devnet. It initializes a Flask app and uses meinheld web server to serve the http requests made to the devnet. The flask app is just a wrapper around the Starknet state object from the Starknet testing framework.

This is the file that can be modified if you need any customisation while starting up the server.
Use this script to run without meinheld (https://github.com/Shard-Labs/starknet-devnet/blob/master/scripts/starknet-devnet-debug.sh)

For speeding up use the lite-mode. The devnet gives an error if you send too many transactions simultaneously since the state object updates cannot be handled quickly and there are conflicting updates. To get around this, the trick I used is to run the devnet with just the flask webserver. Initialise with lite mode - you need to be ok with multiple transactions getting the same transaction hash though.

Starknetwrapper.py

This is the most important section of the code. StarknetWrapper is an object that wraps a Starknet state object internally. It stores the data that needs to be returned by the server like contract states, storage, transactions etc. 

Deploy Contract
The wrapper deploy function expects a Deploy type Transaction object as an argument (whereas the internal Starknet state object's deploy function expects the contract_class, constructor calldata and contract address salt). This deploy transaction is provided by the add_transaction function inside this file https://github.com/Shard-Labs/starknet-devnet/blob/master/starknet_devnet/blueprints/gateway.py - this is where the transaction data is parsed and loaded into a Transaction object (actually done inside shared.py). 

The wrapper deploy function first creates an instance of InternalDeploy (from starkware.starknet.business_logic.internal_transaction) Transaction object. This object represents an internal transaction in the starknet system. This object stores, amongst other fields, the contract class hash (which is calculated based on the contract class / contract definition available from the Transaction object that is passed as an argument to this deploy function), the deployed contract address (it is calculated deterministically based on salt, class hash, constructor calldata and deployer address (0)), transaction hash (this is again calculated deterministically based on deployed version, contract address, constructor call data and chain id).

The wrapper deploy function then calls the deploy function on the internal starknet object. The starknet contract object returned by this call is stored in the DevnetContracts field inside the wrapper object.
The [DevnetContracts](https://github.com/Shard-Labs/starknet-devnet/blob/master/starknet_devnet/contracts.py) class stores the deployed contracts of the devnet.

After calling the deploy function on the internal starknet state object, the internal __update_state function is called. This generates the block info, does the state update calculation and block state_root calculation (unless using lite-mode). This update state function calls on the block_info_generator to get the next block info (BlockInfo object). This BlockInfo object's constructor requies - gas price, block number, block timestamp, sequencer address and cairo lang version. The blocktime stamp is not increased for a new block unless done so explicitly.

The instance of the DevnetTransaction is stored by the wrapper object using the DevnetTransactions class. The __store_transaction function stores the transaction and also generates a new block using the generate function of the DevnetBlocks class. This generate function inside blocks.py actually generates the block hash (unless using lite mode) and then creates a StarknetBlock object using StarknetBlock.create - this StarknetBlock object is actually returned by the generate function. This StarknetBlock.create function requires the following parameters - block_hash, block_number, state_root, list of transactions (Internal Tx objects), timestamp, transaction receipts, status, gas price, sequencer address, parent block hash and starknet version.

Declare Contract
The wrapper declare function expects a Declare type transaction object. It then creates an InternalDeclare type tx using the Declare type transaction object received as argument and the general config. It then calls the declare function on the internal starknet state object which expects an argument of type ContractClass. This object is obtained from the Declare type transaction object originally received as argument.

The declared class hash and the contract class are stored in the contracts field of this wrapper class. Also the instance of the DevnetTransaction object is stored using the DevnetTransactions class.
