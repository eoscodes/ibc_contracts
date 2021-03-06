
IBC System Deployment and Test
------------------------------

This article will detail how to build, deploy and use the IBC system, and take the deployment between `BOS testnet`
and `Kylin testnet` as an example. If you want to deploy IBC system between EOSIO blockchains, 
such as between `EOS mainnet` and `Kylin or Jungle testnet` or between other side/parallel chains 
whose consensus algorithem didn't modified, you need not `ibc_plugin_bos`, 
only `ibc_plugin_eos` is needed; if you want to deploy between EOSIO chains 
whose underlying source code especially the consensus algorithem has been modified,
it's need to determine whether those chains can use the `ibc_contracts` and `ibc_plugin_eos` according to the actual situation.

Now IBC Plugin release version include:

- [BOS IBC Plugin](https://github.com/boscore/ibc_plugin_bos)
- [EOS IBC Plugin](https://github.com/boscore/ibc_plugin_eos)
- [WAX IBC Plugin](https://github.com/boscore/ibc_plugin_wax) Tag: [ibc-v3.0.0-rc](https://github.com/boscore/ibc_plugin_wax/releases/tag/ibc-v3.0.0-rc) 


Notes: 
- Current BOS mainnt's consensus algorithem is 
[batch-pbft](https://github.com/boscore/Documentation/blob/master/LIB/Algorithm_for_improving_EOSIO_consensus_speed_based_on_Batch-PBFT.md), 
which is different with EOS mainnet's and Kylin or Jungle testnet's consensus algorithem 
[pipeline-pbft](https://github.com/EOSIO/welcome/blob/master/docs/04_protocol/01_consensus_protocol.md#31-layer-1-native-consensus-abft).

Contents
--------
 * [Build](#build)
 * [Create Accounts](#create-accounts)
 * [Configure and Start Relay Nodes](#configure-and-start-relay-nodes)
 * [Deploy and Init IBC Contracts](#deploy-and-init-ibc-contracts)
 * [Register Token](#register-token)
 * [Send IBC Transactions](#send-ibc-transactions)


### Build
- Build contracts  
  Please refer to [ibc_contracts](https://github.com/boscore/ibc_contracts) and the [pre-build](https://github.com/boscore/bos.contract-prebuild/tree/master/bosibc) for IBC contracts.

  
- Build ibc_plugin_eos and ibc_plugin_bos  
  Please refer to [ibc_plugin_eos](https://github.com/boscore/ibc_plugin_eos) and [ibc_plugin_bos](https://github.com/boscore/ibc_plugin_bos). 
  And there are Docker images you can find in the release page.

  
### Create Accounts
Five accounts need to be created on each blockchain:
- one relayer account for ibc_plugin nodeos, 
- one account to deploy `ibc.token` contract
- one account to deploy `ibc.chain` contract
- one account to be the administrator account for peg token which can adjust the parameters for their token, for example `min_once_transfer`
- one account to test the IBC function free for transaction fee
   
For deploying IBC contracts, and one charge free account used for ibc system monitoring. let's assume that the four accounts on both chains are:   

**ibc3relay333**: used by ibc_plugin to push ibc related transactions, and need a certain amount of `cpu resources`, typically require 10 minutes or more, and require a certain amount of `net resources`.   
**ibc3chain333**: used to deploy ibc.chain contract, and need a certain amount of ram for tables data, typically require 1.3MB or more.  
**ibc3token333**: used to deploy ibc.token contract, and need a certain amount of ram for tables data, typically require 2.5MB or more.  
**freeaccount1**: ibc transactions to and from this account is **transaction fee free** and is used for fault monitoring of the IBC system 
in production environment，throw sending ibc transactions regularly and check the results, avoid the need to constantly recharge this account.
**ibc3admin333**: used to manage the configures for peg token. 
  
Note:

  The relayer account is not a must in theory, there is a `check_relay_auth` switch in code. 
  In the future, relayer account can be disable or an optional configure.


### Configure and Start Relay Nodes
Suppose the IP address and port of the relay node of Kylin testnet are 192.168.1.101:5678, 
and node of BOS testnet are 192.168.1.102:5678

Some key configurations and ibc-related configurations are listed below. 

- configure relay node of Kylin testnet  
``` 
max-transaction-time = 1000
contracts-console = true

# ------------ ibc related configures ------------ #
plugin = eosio::ibc::ibc_plugin

ibc-listen-endpoint = 0.0.0.0:5678
#ibc-peer-address = 192.168.1.102:5678 

ibc-token-contract = ibc3token333
ibc-chain-contract = ibc3chain333
ibc-relay-name = ibc3relay333
ibc-relay-private-key = <the pulic key of ibc3relay333>=KEY:<the private key of ibc3relay333>

# ------------------- p2p peers -------------------#
p2p-peer-address = <Kylin testnet p2p endpoint>
```

- configure relay node of BOS testnet  
``` 
max-transaction-time = 1000
contracts-console = true

# ------------ ibc related configures ------------ #
plugin = eosio::ibc::ibc_plugin

ibc-listen-endpoint = 0.0.0.0:5678
ibc-peer-address = 192.168.1.101:5678 

ibc-token-contract = ibc3token333
ibc-chain-contract = ibc3chain333
ibc-relay-name = ibc3relay333
ibc-relay-private-key = <the pulic key of ibc3relay333>=KEY:<the private key of ibc3relay333>

# ------------------- p2p peers -------------------#
p2p-peer-address = <BOS testnet p2p endpoint>
```

- relay nodes's log  
When the relay nodes of two chains started, they try to connect to each other, 
and check the IBC contracts' status on their respective chains, and print the logs according to these status.


### Deploy and Init IBC Contracts

#### Choose the version of ibc.token

For example, if we take BOS Testnet as the HUB chain, we need to deploy `HUB ibc.chain` contract which in [*_hub_ibc_token](https://github.com/boscore/bos.contract-prebuild/tree/master/bosibc/).  
For the parallel chains, the normal [ibc.token](https://github.com/boscore/bos.contract-prebuild/tree/master/bosibc/ibc.token) should be deployed.   
[ibc.chain](https://github.com/boscore/bos.contract-prebuild/tree/master/bosibc/ibc.token) is same for HUB chain and parallel chains.  

- ibc.chain and ibc.token contracts on Kylin testnet  
```bash
cleos_kylin='cleos -u <Kylin testnet http endpoint>'
contract_chain=ibc3chain333
contract_token=ibc3token333
contract_token_pubkey='<public key of contract_token>' # public active key is ok

$cleos_kylin set contract ${contract_chain} <contract_chain_folder> -x 1000 -p ${contract_chain}
$cleos_kylin set contract ${contract_token} <contract_token_folder> -x 1000 -p ${contract_token}

# if check_relay_auth is set to true, relay account should be registered to allown ibc_plugin use it to push transactions.
relay_account=ibc3relay333
$cleos_kylin push action ${contract_chain} relay  '["add",'${relay_account}']' -p ${contract_chain}
$cleos_kylin push action ${contract_chain} setglobal '["bostest","<bos testnet chain_id>","batch"]' -p ${contract_chain}

$cleos_kylin set account permission ${contract_token} active '{"threshold": 1, "keys":[{"key":"'${contract_token_pubkey}'", "weight":1}], "accounts":[ {"permission":{"actor":"'${contract_token}'","permission":"eosio.code"},"weight":1}], "waits":[] }' owner -p ${contract_token}
$cleos_kylin push action ${contract_token} setglobal '["kylin",true]' -p ${contract_token}
$cleos_kylin push action ${contract_token} regpeerchain '["bostest","https://bos-test.bloks.io","ibc3token333","ibc3chain333","freeaccount1",5,1000,1000,true]' -p ${contract_token}
```

- ibc.chain and ibc.token contracts on BOS testnet  
```bash
cleos_bos='cleos -u <BOS testnet http endpoint>'
contract_chain=ibc3chain333
contract_token=ibc3token333
contract_token_pubkey='<public key of contract_token>' # public active key is ok

$cleos_bos set contract ${contract_chain} <contract_chain_folder> -x 1000 -p ${contract_chain}
$cleos_bos set contract ${contract_token} <contract_token_folder> -x 1000 -p ${contract_token}

# if check_relay_auth is set to true, relay account should be registered to allown ibc_plugin use it to push transactions.
relay_account=ibc3relay333
$cleos_bos push action ${contract_chain} relay  '["add",'${relay_account}']' -p ${contract_chain}

$cleos_bos push action ${contract_chain} setglobal '["kylin","<Kylin testnet chain_id>","pipeline"]' -p ${contract_chain}

$cleos_bos set account permission ${contract_token} active '{"threshold": 1, "keys":[{"key":"'${contract_token_pubkey}'", "weight":1}], "accounts":[ {"permission":{"actor":"'${contract_token}'","permission":"eosio.code"},"weight":1}], "waits":[] }' owner -p ${contract_token}
$cleos_bos push action ${contract_token} setglobal '["bostest",true]' -p ${contract_token}
$cleos_bos push action ${contract_token} regpeerchain '["kylin","https://www.cryptokylin.io","ibc3token333","ibc3chain333","freeaccount1",5,1000,1000,true]' -p ${contract_token}
```

Note:
  - The document for [ibc.token](https://github.com/boscore/ibc_contracts/tree/master/ibc.token)
  - The document for [ibc.chain](https://github.com/boscore/ibc_contracts/tree/master/ibc.chain)

#### IBC HUB chain init

The HUB chain initialization [here](https://github.com/boscore/ibc_contracts/blob/master/docs/IBC_Hub_Protocol.md#hub-init)

After initializing the two contracts, the relay nodes of two chains will start synchronizing a first block header of 
each other's peer chain as a `genesis block header` of ibc.chain contract, then synchronize a batch of block headers.


### Register Token
Please refer to detailed content [Token_Registration_and_Management](./Token_Registration_and_Management.md)

- register `EOS Token` (eosio.token contract on Kylin testnet) on `ibc.token contracts on both Kylin and BOS testnet` to open a `Token channel` for it.
```bash
eos_admin=ibc3admin333

$cleos_kylin push action ${contract_token} regacpttoken \
    '["eosio.token","1000000000.0000 EOS","1.0000 EOS","100000.0000 EOS",
    "1000000.0000 EOS",1000,"cryptokylin","https://www.cryptokylin.io","ibc3admin333","fixed","0.1000 EOS",0.01,"0.1000 EOS",true]' -p ${contract_token}
        
$cleos_bos push action ${contract_token} regpegtoken \
    '["kylin","eosio.token","1000000000.0000 EOS","1.0000 EOS","100000.0000 EOS",
    "1000000.0000 EOS",1000,"ibc3admin333","0.1000 EOS",true]' -p ${contract_token} 
```

- register `BOS Token` (eosio.token contract on BOS testnet) on `ibc.token contracts on both Kylin and BOS testnet` to open a `Token channel` for it.  
```bash
bos_admin=ibc3admin333

$cleos_bos push action ${contract_token} regacpttoken \
    '["eosio.token","1000000000.0000 BOS","1.0000 BOS","100000.0000 BOS",
    "1000000.0000 BOS",1000,"bos organization","https://boscore.io","ibc3admin333","fixed","0.1000 BOS",0.01,"0.1000 BOS",true]' -p ${contract_token}
    
$cleos_kylin push action ${contract_token} regpegtoken \
    '["bostest","eosio.token","1000000000.0000 BOS","1.0000 BOS","10000.0000 BOS",
    "1000000.0000 BOS",1000,"ibc3admin333","0.1000 BOS",true]' -p ${contract_token}
```

#### HUB chain registration

For the HUB chain, there is a new function `regpegtoken2` to call one time to finsih the registration. Other parallel chains, still need to call `regpegtoken` and `regpegtoken`. Check the [detail]().


After performing the above steps, you need to wait for a period of time, preferably more than 6 minutes, and watch the logs of the two relay nodes.
You can see a log like the one below. It is better to wait for the last part of the following log to change from `,false]` to `,true]` for the first time before sending inber-blockchain transactions.

For now, it's needed to call `regpegtoken` in every parallel chains, in the future, IBC can do it through broadcast protocol.

``` 
ibc_plugin.cpp:3168           start_ibc_heartbeat_ ] origtrxs_table_id_range [0,0] cashtrxs_table_seq_num_range [0,0] new_producers_block_num 0, lwcls_range [75468428,75469424,true]
```

### Send IBC Transactions

- create some accounts on both chains  
```bash
kylin_acnt1=eosaccount11  # account on Kylin testnet
kylin_acnt2=eosaccount12  # account on Kylin testnet
bos_acnt1=bosaccount11  # account on BOS testnet
bos_acnt2=bosaccount12  # account on BOS testnet
```

- send EOS form Kylin testnet ${kylin_acnt1} to BOS testnet ${bos_acnt1}  
```bash
$cleos_kylin transfer ${kylin_acnt1} ${contract_token} "1.0000 EOS" ${bos_acnt1}"@bostest have a nice day!" -p ${kylin_acnt1}
```

- withdraw EOS(peg token) form BOS testnet ${bos_acnt1} to Kylin testnet ${kylin_acnt2}  
```bash
$cleos_bos push action ${contract_token} transfer '["'${bos_acnt1}'","'${contract_token}'","1.0000 EOS" "'${kylin_acnt2}'@kylin"]' -p ${bos_acnt1}
```

- send BOS form BOS testnet ${bos_acnt1} to Kylin testnet ${kylin_acnt1}  
```bash
$cleos_bos transfer ${bos_acnt1} ${contract_token} "1.0000 BOS" ${kylin_acnt1}"@kylin have a nice day!" -p ${bos_acnt1}
```

- withdraw BOS(peg token) form Kylin testnet ${kylin_acnt1} to BOS testnet ${bos_acnt2}  
```bash
$cleos_kylin push action ${contract_token} transfer '["'${kylin_acnt1}'","'${contract_token}'","1.0000 BOS" "'${bos_acnt2}'@bostest"]' -p ${kylin_acnt1}
```
After executing above commands, you need to wait for a period of time to receive your token on the peer chain.
It takes about six minutes to transfer from Kylin to BOS, and transfer from BOS to Kylin takes about 10 seconds.

IBC transactions can be processed concurrently, that is, new ibc transactions can be sent without waiting for the previous ibc transactions to be completed.