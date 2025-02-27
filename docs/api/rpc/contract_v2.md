---
sidebarDepth: 4
---

# Contract

:::tip Maintainer
[viteLiz](https://github.com/viteLiz)
:::

## Create Contract

Basically, creating contract is sending a transaction (transaction type = 1) by specifying contract's binary code and input parameters. This process includes the following steps.

1. Call RPC method `contract_createContractToAddress` to generate a new contract address;
2. Encode input parameters according to contract constructor in ABI. This can be completed by calling `abi.encodeParameters` in **vite.js**. If constructor has no input parameter, skip this step;
3. Generate transaction data. The data content consists of a 14-byte fixed-length prefix (10-byte delegated consensus group id + 1 byte contract type + 1 byte response latency + 1 byte random degree + 1 byte quota multiplier), compiled contract code and encoded parameters (generated in Step 2).
4. Construct account block by calling RPC method `ledger_sendRawTransaction` to create contract. Here parameter `toAddress` is the contract address generated in Step 1. `data` is transaction data created in Step 3. `blockType` = 1, indicating creating contract transaction. `amount` and `tokenId` stand for the amount of token that are transferred to the contract upon creation. `fee` is a fixed amount of 10 VITE as contract deployment expense.

:::tip Tips
Above steps are implemented in method `builtinTxBlock.createContract` in **vite.js** 
:::

Parameter description:

* **Delegated consensus group id**: The response transaction blocks on contract chain are produced by the supernodes of delegated consensus group that the contract registers for. Public delegated consensus group has id `00000000000000000002`.
* **Contract type**: Use 1 to indicate Solidity++ contract.
* **Response latency**: The number of snapshot blocks by which request sent to the contract has been confirmed before responding to the transaction. The value range is 0-75. The value 0 means that there is no need to wait and the respond block will be produced immediately. If timestamp, snapshot block height, or random number is used in the contract, this parameter must be greater than zero. Larger response latency means slower contract response and each response transaction will consume more extra quota.
* **Random degree**: The number of snapshot blocks having random seed by which request sent to this contract is confirmed before responding to the transaction. The value range is 0-75. The value 0 indicates that there is no need to wait for the request transaction to be included in a snapshot block containing random number. If random number instruction is used in the contract, the value is required to be above 0. The larger the value, the more secure the random number. This parameter must be no less than response latency.
* **Quota multiplier**: This parameter defines how many times the quota is consumed for requested transaction when the contract is called. The quota charged by the contract response transaction is not affected. The value ranges from 10 to 100, which means that the quota consumed is 1-10 times. For example, a value of 15 means that the requested transaction to the contract is charged 1.5 times quota.

## Call Contract

Calling a smart contract is to send a special transaction to smart contract's address, specifying method name and passed-in parameters. 

1. Encode method name and passed-in parameter as ABI interface defines. This step can be done by calling `abi.encodeFunctionCall` method in **vite.js**
2. Build transaction block and call RPC method `ledger_sendRawTransaction` to send the transaction to contract. Herein `toAddress` is the contract address and `data` is transaction data created in Step 1. `blockType` is set to 2. `amount` and `tokenId` stand for the amount of token that are transferred to the contract. `fee` is 0, indicating no transaction fee.

:::tip Tips
Above steps are implemented in method `builtinTxBlock.callContract` in **vite.js** 
:::

## Read States Off-chain

Contract's states can be accessed off-chain through `getter` methods. The ABI definition of `getter` method and off-chain code are generated when the contract is compiled. 

1. Encode off-chain method name and passed-in parameter as ABI interface defines. This step can be done by calling `abi.encodeFunctionCall` method in **vite.js**
2. Call RPC method `contract_callOffChainMethod` to query contract states

## Call Built-in Contract

Calling built-in contract is to send a special transaction to built-in contract's address, specifying calling method name and passed-in parameters. 
Vite has provided 3 built-in smart contracts: Quota, Token Issuance and Consensus.

1. Encode method name and passed-in parameter as ABI interface defines. This step can be done by calling `abi.encodeFunctionCall` method in **vite.js**
2. Build transaction block and call RPC method `ledger_sendRawTransaction` to send the transaction to built-in contract.

:::tip Tips
Methods of built-in contracts are provided in `builtinTxBlock` in **vite.js** 
:::

## Quota Contract
### Contract Address
`vite_0000000000000000000000000000000000000003f6af7459b9`

### ABI
```json
[
  // Stake for quota
  {"type":"function","name":"Stake", "inputs":[{"name":"beneficiary","type":"address"}]},
  // Cancel staking
  {"type":"function","name":"CancelStake","inputs":[{"name":"beneficiary","type":"address"},{"name":"amount","type":"uint256"}]},
  // Stake for quota delegation
  {"type":"function","name":"DelegateStake", "inputs":[{"name":"stakeAddress","type":"address"},{"name":"beneficiary","type":"address"},{"name":"bid","type":"uint8"},{"name":"stakeHeight","type":"uint64"}]},
  // Cancel staking via delegation
  {"type":"function","name":"CancelDelegateStake","inputs":[{"name":"stakeAddress","type":"address"},{"name":"beneficiary","type":"address"},{"name":"amount","type":"uint256"},{"name":"bid","type":"uint8"}]},
]
```
Callback methods:
```json
[
  // Callback function for delegated staking
  {"type":"function","name":"DelegateStakeCallback","inputs":[{"name":"stakeAddress","type":"address"},{"name":"beneficiary","type":"address"},{"name":"amount","type":"uint256"},{"name":"bid","type":"uint8"},{"name":"success","type":"bool"}]},
  // Callback function for cancelling delegated staking
  {"type":"function","name":"CancelDelegateStakeCallback","inputs":[{"name":"stakeAddress","type":"address"},{"name":"beneficiary","type":"address"},{"name":"amount","type":"uint256"},{"name":"bid","type":"uint8"},{"name":"success","type":"bool"}]}       
]
```

#### Stake

Stake for quota. The minimum staking amount is 134 VITE, which can be retrieved after 259,200 snapshot blocks (about 3 days). The locking period will be reset if a new staking record is done for the same beneficiary. 

- **Parameters**: 
  * `beneficiary`: `string address` Address of staking beneficiary

#### CancelStake

Cancel an existing staking and retrieves staked tokens after the lock period expires. However, retrieving a part of staked tokens is also permitted. In that case, the original staking is still valid.  

- **Parameters**: 
  * `beneficiary`: `string address` Address of staking beneficiary
  * `amount`: `string bigint` Amount to retrieve. Cannot be less than 134 VITE. The remaining staking amount cannot be less than 134 VITE. 

#### DelegateStake

Stake for quota via delegation. The minimum staking amount is 134 VITE. Caller of this ABI method should be another contract which stakes on behalf of its users. 

- **Parameters**: 
  * `stakeAddress`: `string address` Address of original staking account
  * `beneficiary`: `string address` Address of staking beneficiary
  * `bid`: `uint8`   Business id. Staking records from the same original staking address and for the same beneficiary will be categorized into different groups by business id. The locking period of each group will be dependent on the last staking record among the individual business group. 
  * `stakeHeight`: `string uint64` Locking period in terms of number of snapshot blocks. No less than 259,200.

#### DelegateStakeCallback

Callback method for delegated staking. Basically, it is the caller contract's responsibility to implement the callback method to do refund(or other logic) upon staking failure.

- **Parameters**: 
  * `stakeAddress`: `string address` Address of original staking account
  * `beneficiary`: `string address` Address of staking beneficiary
  * `bid`: `uint8`   Business id. Staking records from the same original staking address and for the same beneficiary will be categorized into different groups by business id. The locking period of each group will be dependent on the last staking record among the individual business group. 
  * `stakeHeight`: `string uint64` Locking period in terms of number of snapshot blocks. No less than 259,200
  * `success`: `bool` If `false`, caller contract should return the relevant VITE tokens to the user

#### CancelDelegateStake

Cancel an existing delegated staking and retrieves staked tokens after the lock period expires. However, retrieving a part of staked tokens is also permitted. In that case, the original delegated staking is still valid.  

- **Parameters**: 
  * `stakeAddress`: `string address` Address of original staking account
  * `beneficiary`: `string address` Address of staking beneficiary
  * `amount`: `string bigint` Amount to retrieve. Cannot be less than 134 VITE. The remaining staking amount cannot be less than 134 VITE. 
  * `bid`: `uint8`   Business id. Staking records from the same original staking address and for the same beneficiary will be categorized into different groups by business id. The locking period of each group will be dependent on the last staking record among the individual business group. 

#### CancelDelegateStakeCallback

Callback method for cancelling delegated staking. Basically, it is the caller contract's responsibility to implement the callback method to do refund(or other logic) upon cancellation failure.

- **Parameters**: 
  * `stakeAddress`: `string address` Address of original staking account
  * `beneficiary`: `string address` Address of staking beneficiary
  * `amount`: `string bigint` Amount to retrieve. Cannot be less than 134 VITE. The remaining staking amount cannot be less than 134 VITE. 
  * `bid`: `uint8`   Business id. Staking records from the same original staking address and for the same beneficiary will be categorized into different groups by business id. The locking period of each group will be dependent on the last staking record among the individual business group. 
  * `success`: `bool` If `true`, caller contract should return the relevant VITE tokens to the user

## Consensus Contract
### Contract Address
`vite_0000000000000000000000000000000000000004d28108e76b`

### ABI
```json
[
  // Register block producer
  {"type":"function","name":"Register", "inputs":[{"name":"gid","type":"gid"},{"name":"sbpName","type":"string"},{"name":"blockProducingAddress","type":"address"}]},
  // Update block producing address
  {"type":"function","name":"UpdateBlockProducingAddress", "inputs":[{"name":"gid","type":"gid"},{"name":"sbpName","type":"string"},{"name":"newBlockProducingAddress","type":"address"}]},
  // Cancel block producer
  {"type":"function","name":"Revoke","inputs":[{"name":"gid","type":"gid"}, {"name":"sbpName","type":"string"}]},
  // Withdraw block producing reward
  {"type":"function","name":"WithdrawReward","inputs":[{"name":"gid","type":"gid"},{"name":"sbpName","type":"string"},{"name":"receiveAddress","type":"address"}]},
  // Vote for block producer
  {"type":"function","name":"Vote", "inputs":[{"name":"gid","type":"gid"},{"name":"sbpName","type":"string"}]},
  // Cancel voting
  {"type":"function","name":"CancelVote","inputs":[{"name":"gid","type":"gid"}]}
]
```

#### Register

Register new block producer in a consensus group. For snapshot consensus group, this requires staking 1m VITE for a locking period of 7,776,000 snapshot blocks (about 3 months). 

- **Parameters**: 
  * `gid`: `string gid` Consensus group id. For snapshot consensus group, fill in `00000000000000000001`
  * `sbpName`: `string` Distinct name of block producer
  * `blockProducingAddress`: `string address` Block producing address. Using a different address with registration address is recommended.

:::tip Tips
Have your gvite node fully synced before registration
:::

#### UpdateBlockProducingAddress

Update block producing address for existing producer

- **Parameters**: 
  * `gid`: `string gid` Consensus group id. For snapshot consensus group, fill in `00000000000000000001`
  * `sbpName`: `string` Distinct name of block producer
  * `newBlockProducingAddress`: `string address` New block producing address

#### Revoke

Cancel block producer after locking period expires. Cancelled producer is not able to produce blocks or receive reward.

- **Parameters**: 
  * `gid`: `string gid` Consensus group id. For snapshot consensus group, fill in `00000000000000000001`
  * `sbpName`: `string` Distinct name of block producer

#### WithdrawReward

Withdraw block producing reward. For snapshot consensus group, the top 100 block producers of the last round in each cycle can withdraw rewards in 1 hour after the cycle is over.

- **Parameters**: 
  * `gid`: `string gid` Consensus group id. For snapshot consensus group, fill in `00000000000000000001`
  * `sbpName`: `string` Distinct name of block producer
  * `receiveAddress`: `string address` Address to receive block producing rewards

#### Vote

Vote for a block producer. User's account balance is used as voting weight. For the same account, the latest voting record will count if multiple votes are made. 

- **Parameters**: 
  * `gid`: `string gid` Consensus group id. For snapshot consensus group, fill in `00000000000000000001`
  * `sbpName`: `string` Distinct name of block producer

#### CancelVote

Cancel existing voting

- **Parameters**: 
  * `gid`: `string gid` Consensus group id. For snapshot consensus group, fill in `00000000000000000001`

## Token Issuance Contract
### Contract Address
`vite_000000000000000000000000000000000000000595292d996d`

### ABI
```json
[
  // Issue new token
  {"type":"function","name":"IssueToken","inputs":[{"name":"isReIssuable","type":"bool"},{"name":"tokenName","type":"string"},{"name":"tokenSymbol","type":"string"},{"name":"totalSupply","type":"uint256"},{"name":"decimals","type":"uint8"},{"name":"maxSupply","type":"uint256"},{"name":"isOwnerBurnOnly","type":"bool"}]},
  // Re-issue. Mint additional tokens
  {"type":"function","name":"ReIssue","inputs":[{"name":"tokenId","type":"tokenId"},{"name":"amount","type":"uint256"},{"name":"receiveAddress","type":"address"}]},
  // Burn
  {"type":"function","name":"Burn","inputs":[]},
  // Transfer ownership of re-issuable token
  {"type":"function","name":"TransferOwnership","inputs":[{"name":"tokenId","type":"tokenId"},{"name":"newOwner","type":"address"}]},
  // Change token type from re-issuable to non-reissuable
  {"type":"function","name":"DisableReIssue","inputs":[{"name":"tokenId","type":"tokenId"}]},
  // Query token information
  {"type":"function","name":"GetTokenInfo","inputs":[{"name":"tokenId","type":"tokenId"},{"name":"bid","type":"uint8"}]},
  // Token issuance event
  {"type":"event","name":"issue","inputs":[{"name":"tokenId","type":"tokenId","indexed":true}]},
  // Token re-issuance event
  {"type":"event","name":"reIssue","inputs":[{"name":"tokenId","type":"tokenId","indexed":true}]},
  // Burn event
  {"type":"event","name":"burn","inputs":[{"name":"tokenId","type":"tokenId","indexed":true},{"name":"burnAddress","type":"address"},{"name":"amount","type":"uint256"}]},
  // Ownership transfer event
  {"type":"event","name":"transferOwnership","inputs":[{"name":"tokenId","type":"tokenId","indexed":true},{"name":"owner","type":"address"}]},
  // Token type change event
  {"type":"event","name":"disableReIssue","inputs":[{"name":"tokenId","type":"tokenId","indexed":true}]}
]
```
Callback method:
```json
[
  // Callback method for querying token information 
  {"type":"function","name":"GetTokenInfoCallback","inputs":[{"name":"tokenId","type":"tokenId"},{"name":"bid","type":"uint8"},{"name":"exist","type":"bool"},{"name":"decimals","type":"uint8"},{"name":"tokenSymbol","type":"string"},{"name":"index","type":"uint16"},{"name":"owner","type":"address"}]}
]
```

#### IssueToken

Issue a new token at a cost of 1,000 VITE. The issuing account will become token's owner and receive an amount of tokens equivalent to initial total supply. 

- **Parameters**: 
  * `isReIssuable`: `bool` If `true`, newly additional tokens can be minted after initial issuance
  * `tokenName`: `string` Name of token in 1-40 characters, including uppercase/lowercase letters, spaces and underscores. Cannot have consecutive spaces or start/end with space
  * `tokenSymbol`: `string` Symbol of token in 1-10 characters, including uppercase/lowercase letters, spaces and underscores. Cannot use `VITE`, `VCP` or `VX`
  * `totalSupply`: `string bigint` Total supply. Having $totalSupply \leq 2^{256}-1$. For re-issuable token, this is initial total supply
  * `decimals`: `uint8` Decimal places. Having $10^{decimals} \leq totalSupply$
  * `maxSupply`: `string bigint` Maximum supply. Mandatory for re-issuable token. Having $totalSupply \leq maxSupply \leq 2^{256}-1$. For non-reissuable token, fill in `0` 
  * `isOwnerBurnOnly`: `bool` If `true`, the token can only burned by owner. Mandatory for re-issuable token. For non-reissuable token, fill in `false`
  
#### ReIssue

Mint an amount of additional tokens after initial issuance. For token's owner only

- **Parameters**: 
  * `tokenId`: `string tokenId` Token id
  * `amount`: `string bigint` Issuing amount. Token's total supply will increase accordingly but cannot exceed max supply
  * `receiveAddress`: `string address` Address to receive newly issued tokens

#### Burn

Burn an amount of tokens by sending to token issuance contract. For re-issuable token only

#### TransferOwnership

Transfer token ownership. For re-issuable token only

- **Parameters**: 
  * `tokenId`: `string tokenId` Token id
  * `newOwner`: `string address` Address of new owner

#### DisableReIssue

Change re-issuable token to non-reissuable. Cannot change back once it is done

- **Parameters**: 
  * `tokenId`: `string tokenId` Token id

#### GetTokenInfo

Query token info

- **Parameters**: 
  * `tokenId`: `string tokenId` Token id
  * `bid`: `uint8` Business id, optional

#### GetTokenInfoCallback

Callback method for querying token info

- **Parameters**: 
  * `tokenId`: `string tokenId` Token id
  * `bid`: `uint8` Business id
  * `exist`: `bool` If `true`, the token exists
  * `tokenSymbol`: `string` Symbol of token
  * `index`: `uint16` Distinct token index beginning from 0 to 999. For tokens having the same symbol, sequential indexes will be allocated according to issuance time
  * `owner`: `string address` Address of token's owner

## contract_createContractAddress

Create a new contract address

- **Parameters**: 
  * `string address`: Address of creator
  * `string uint64`: Current height of account chain
  * `string hash`: Hash of previous account block

- **Returns**: 
  - `string address` New address

- **Example**:
::: demo
```json tab:Request
{  
   "jsonrpc":"2.0",
   "id":1,
   "method":"contract_createContractAddress",
   "params":[
      "vite_a5a7f08011c2f0e40ccd41b5b79afbfb818d565f566002d3c6", 
      "2", 
      "3a56babeb0a8140b12ac55e91d2e05c41f908ebe99767b0e4aa5cd7af22d6de7"]
}
```
```json tab:Response
{  
   "jsonrpc":"2.0",
   "id":1,
   "result": "vite_96a7911037179451bada2ab05ee070ba83dcfa2fac2ad6d770"
}
```
:::

## contract_getContractInfo

Return contract information by address

- **Parameters**: 
  * `string address`: Address of contract
  
- **Returns**: 
  - `ContractInfo`
    - `code`: `string base64`  Code of contract
    - `gid`: `string gid`  Associated consensus group id
    - `responseLatency`: `uint8`  Response latency
    - `randomDegree`: `uint8` Random degree
    - `quotaMultiplier`: `uint8` Quota multiplier

- **Example**:
::: demo
```json tab:Request
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "contract_getContractInfo",
    "params": ["vite_22f4f195b6b0f899ea263241a377dbcb86befb8075f93eeac8"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "code": "AWCAYEBSYAQ2EGEAQVdgADV8AQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACQBGP/////FoBjkabLSxRhAEZXW2AAgP1bYQCJYASANgNgIIEQFWEAXFdgAID9W4EBkICANXT///////////////////////////8WkGAgAZCSkZBQUFBhAItWWwBbgHT///////////////////////////8WRmn/////////////FjRgQFFgQFGAggOQg4WH8VBQUFCAdP///////////////////////////xZ/qmUoH130tL08cfK6JZBbkHIF/OCAmoFu+OBLTUlqhbs0YEBRgIKBUmAgAZFQUGBAUYCRA5CiUFb+oWVienpyMFgg5BEYZploBADsJutGp1y0+UwegyI5VjOkuA+v2lg7JFoAKQ==",
        "gid": "00000000000000000002",
        "responseLatency": 2,
        "randomDegree": 1,
        "quotaMultiplier": 10
    }
}
```
:::

## contract_callOffChainMethod

Call contract's offchain method 

- **Parameters**: 
  * `Params`:
    * `address`:`string address` Address of contract
    * `code`:`string base64` Binary code for offchain query. This is the value of "OffChain Binary" section generated when compiling the contract with `--bin`
    * `data`:`string base64` Encoded passed-in parameters
    
- **Returns**: 
  - `string base64` Encoded return value. Use decode methods of vitejs to get decoded value

- **Example**:
::: demo
```json tab:Request
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "contract_callOffChainMethod",
    "params": [{
      "address":"vite_22f4f195b6b0f899ea263241a377dbcb86befb8075f93eeac8",
      "code":"YIBgQFJgBDYQYEJXYAA1fAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAkARj/////xaAY8GjSGUUYERXYEJWWwBbYEpgYFZbYEBRgIKBUmAgAZFQUGBAUYCRA5DzW2AAYABgAFBUkFBgblZbkFb+oWVienpyMFggSaCBXUGf/Mh5lfHDLvGQt9g3K+aLjE2PrRxcLb6RSWQAKQ==",
      "data":"waNIZQ=="
    }]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
}
```
:::

## contract_getContractStorage

Query contract's state

- **Parameters**: 
  * `string address` Address of contract
  * `string` Hex key or prefix of hex key of contract state field
    
- **Returns**: 
  - `map<string,string>` Map of key-value pairs in hex

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 17,
	"method": "contract_getContractStorage",
	"params": ["vite_22f4f195b6b0f899ea263241a377dbcb86befb8075f93eeac8","0000000000000000000000000000000000000000000000000000000000000001"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 17,
    "result": {
        "0000000000000000000000000000000000000000000000000000000000000001": "01"
    }
}
```
:::

## contract_getQuotaByAccount
Return quota balance by account

- **Parameters**: 
  * `string address`: Address of account

- **Returns**: 
  - `QuotaInfo`
    - `currentQuota`: `string uint64` Current available quota
    - `maxQuota`: `string uint64` Max quota, equivalent to UTPE
    - `stakeAmount`: `string bigint` Amount of staking
  
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getQuotaByAccount",
	"params": ["vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "currentQuota": "1554000",
        "maxQuota": "1575000",
        "stakeAmount": "10000000000000000000000"
    }
}
```
:::

## contract_getStakeList
Return staking records by account, ordered by target unlocking height in descending order

- **Parameters**: 
  * `string address`: Address of staking account
  * `int`: Page index, starting from 0
  * `int`: Page size

- **Returns**: 
  - `StakeListInfo`
    - `totalStakeAmount`: `string bigint`  The total staking amount of the account
    - `totalStakeCount`: `int`  The total number of staking records
    - `stakeList`: `Array<StakeInfo>` Staking record list
      - `stakeAddress`: `string address`  Address of staking account
      - `stakeAmount`: `string bigint`  Amount of staking
      - `expirationHeight`: `string uint64`  Target unlocking height
      - `beneficiary`: `string address`  Address of staking beneficiary
      - `expirationTime`: `int64`  Estimated target unlocking time
      - `isDelegated`: `bool`  If `true`, this is a delegated staking record  
      - `delegateAddress`: `string address`  Address of delegate account. For non-delegated staking, 0 is filled
      - `bid`: `uint8`  Business id. For non-delegated staking, 0 is filled
    
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "1.0",
	"id": 1,
	"method": "contract_getStakeList",
	"params": ["vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a",0,10]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "totalStakeAmount": "1000000000000000000000",
        "totalStakeCount": 1,
        "stakeList": [
            {
                "stakeAmount": "1000000000000000000000",
                "beneficiary": "vite_bd756f144d6aba40262c0d3f282b521779378f329198b591c3",
                "expirationHeight": "1360",
                "expirationTime": 1567490923,
                "isDelegated": false,
                "delegateAddress": "vite_0000000000000000000000000000000000000000a4f3a0cb58",
                "stakeAddress": "vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a",
                "bid": 0
            }
        ]
    }
}
```
:::

## contract_getRequiredStakeAmount
Return the minimum required amount of staking in order to obtain the given quota

- **Parameters**: 
  * `string uint64`: Quotas accumulated per second. For example, an amount of 21,000 quota are consumed to send a transaction with no comment, in this case, to satisfy the minimum 280 (21000/75) quota accumulation per second in an epoch, a staking amount of 134 VITE is required

- **Returns**: 
  - `string bigint`: The minimum amount of staking
    
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getRequiredStakeAmount",
	"params": ["280"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": "134000000000000000000"
}
```
:::

## contract_getDelegatedStakeInfo
Return delegated staking information

- **Parameters**: 
  * `Params`
    * `stakeAddress`:`string address`  Address of original staking account
    * `delegateAddress`:`string address`  Address of delegate account
    * `beneficiary`:`string address`  Address of staking beneficiary
    * `bid`: `uint8`   Business id. Staking records from the same original staking address and for the same beneficiary will be categorized into different groups by business id. The locking period of each group will be dependent on the last staking record among the individual business group. 

- **Returns**: 
  - `StakeInfo`
    - `stakeAddress`: `string address`  Address of staking account
    - `stakeAmount`: `string bigint`  Amount of staking
    - `expirationHeight`: `string uint64`  Target unlocking height
    - `beneficiary`: `string address`  Address of staking beneficiary
    - `expirationTime`: `int64`  Estimated target unlocking time
    - `isDelegated`: `bool`  If `true`, this is a delegated staking record  
    - `delegateAddress`: `string address`  Address of delegate account. For non-delegated staking, 0 is filled
    - `bid`: `uint8`  Business id. For non-delegated staking, 0 is filled
    
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getDelegatedStakeInfo",
	"params": [
		{
			"stakeAddress":"vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906", 
			"delegateAddress":"vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a",
			"beneficiary":"vite_098dfae02679a4ca05a4c8bf5dd00a8757f0c622bfccce7d68",
			"bid":2
		}
	]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "stakeAmount": "1000000000000000000000",
        "beneficiary": "vite_098dfae02679a4ca05a4c8bf5dd00a8757f0c622bfccce7d68",
        "expirationHeight": "543",
        "expirationTime": 1567490406,
        "isDelegated": true,
        "delegateAddress": "vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a",
        "stakeAddress": "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906",
        "bid": 2
    }
}
```
:::

## contract_getSBPList
Return registered SBP list, including historical SBP nodes, ordered by target unlocking height in descending order

- **Parameters**: 
  * `string address`: Address of registration account

- **Returns**: 
  - `Array<SBPInfo>`
    - `name`: `string`  Name of SBP
    - `blockProducingAddress`: `string address`  Block producing address
    - `stakeAddress`: `string address`  Address of registration account
    - `stakeAmount`: `string bigint`  Amount of staking
    - `expirationHeight`: `string uint64`  Target unlocking height. Registered SBP node can be cancelled after the locking period expires. 
    - `expirationTime`: `int64`  Estimated target unlocking time
    - `revokeTime`: `int64`  Time of cancellation. For non-cancelled SBP nodes, 0 is filled

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getSBPList",
	"params": ["vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": [
        {
            "name": "s1",
            "blockProducingAddress": "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906",
            "stakeAddress": "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906",
            "stakeAmount": "1000000000000000000000000",
            "expirationHeight": "7776000",
            "expirationTime": 1575266076,
            "revokeTime": 0
        }
    ]
}
```
:::

## contract_getSBPRewardPendingWithdrawal
Return un-retrieved SBP rewards by SBP name

- **Parameters**: 
  * `string`: Name of SBP

- **Returns**: 
  - `RewardInfo`
    - `totalReward`: `string bigint`  The total rewards that have not been retrieved
    - `blockProducingReward`: `string bigint`  Un-retrieved block creation rewards
    - `votingReward`: `string bigint`  Un-retrieved candidate additional rewards(voting rewards)
    - `allRewardWithdrawed`: `bool` If `true`, the SBP node has been cancelled and all rewards have been withdrawn

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getSBPRewardPendingWithdrawal",
	"params": ["s1"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "blockProducingReward": "1499714611872146118517",
        "votingReward": "746306845207209076970",
        "totalReward": "2246021457079355195487",
        "producedBlocks": "0",
        "targetBlocks": "0",
        "allRewardWithdrawed": false
    }
}
```
:::

## contract_getSBPRewardByTimestamp
Return SBP rewards in 24h by timestamp. Rewards of all SBP nodes in the cycle that the given timestamp belongs will be returned. 

- **Parameters**: 
  * `int64`: Timestamp in seconds

- **Returns**: 
  - `RewardByDayInfo` 
    - `rewardMap`: `map<string,RewardInfo>`
      - `totalReward`: `string bigint`  The total rewards that have not been retrieved
      - `blockProducingReward`: `string bigint`  Un-retrieved block creation rewards
      - `votingReward`: `string bigint`  Un-retrieved candidate additional rewards(voting rewards)
      - `producedBlocks`: `string bigint`  Actual blocks produced
      - `targetBlocks`: `string bigint`  Target blocks should be produced
    - `startTime`: `int64` Cycle start time
    - `endTime`: `int64` Cycle end time
    - `cycle`: `string uint64` Index of cycle

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getSBPRewardByTimestamp",
	"params": [1567440000]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "rewardMap": {
            "s1": {
                "blockProducingReward": "1499714611872146118517",
                "votingReward": "746306845207209076970",
                "totalReward": "2246021457079355195487",
                "producedBlocks": "3153",
                "targetBlocks": "3168",
                "allRewardWithdrawed": false
            },
            "s2": {
                "blockProducingReward": "0",
                "votingReward": "0",
                "totalReward": "0",
                "producedBlocks": "0",
                "targetBlocks": "3168",
                "allRewardWithdrawed": false
            }
        },
        "startTime": 1567396800,
        "endTime": 1567483200,
        "cycle": "104"
    }
}
```
:::

## contract_getSBPRewardByCycle
Return SBP rewards of all SBP nodes by cycle

- **Parameters**: 
  * `string uint64`: Index of cycle

- **Returns**: 
  - `RewardByDayInfo` 
    - `rewardMap`: `map<string,RewardInfo>`
      - `totalReward`: `string bigint`  The total rewards that have not been retrieved
      - `blockProducingReward`: `string bigint`  Un-retrieved block creation rewards
      - `votingReward`: `string bigint`  Un-retrieved candidate additional rewards(voting rewards)
      - `producedBlocks`: `string bigint`  Actual blocks produced
      - `targetBlocks`: `string bigint`  Target blocks should be produced
    - `startTime`: `int64` Cycle start time
    - `endTime`: `int64` Cycle end time
    - `cycle`: `string uint64` Index of cycle

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getSBPRewardByCycle",
	"params": ["104"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "rewardMap": {
            "s1": {
                "blockProducingReward": "1499714611872146118517",
                "votingReward": "746306845207209076970",
                "totalReward": "2246021457079355195487",
                "producedBlocks": "3153",
                "targetBlocks": "3168",
                "allRewardWithdrawed": false
            },
            "s2": {
                "blockProducingReward": "0",
                "votingReward": "0",
                "totalReward": "0",
                "producedBlocks": "0",
                "targetBlocks": "3168",
                "allRewardWithdrawed": false
            }
        },
        "startTime": 1567396800,
        "endTime": 1567483200,
        "cycle": "104"
    }
}
```
:::

## contract_getSBP
Return SBP node information

- **Parameters**: 
  * `string`: Name of SBP

- **Returns**: 
  - `SBPInfo`
    - `name`: `string`  Name of SBP
    - `blockProducingAddress`: `string address`  Block producing address
    - `stakeAddress`: `string address`  Address of registration account
    - `stakeAmount`: `string bigint`  Amount of staking
    - `expirationHeight`: `string uint64`  Target unlocking height. Registered SBP node can be cancelled after the locking period expires. 
    - `expirationTime`: `int64`  Estimated target unlocking time
    - `revokeTime`: `int64`  Time of cancellation. For non-cancelled SBP nodes, 0 is filled

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getSBP",
	"params": ["s1"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "name": "s1",
        "blockProducingAddress": "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906",
        "stakeAddress": "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906",
        "stakeAmount": "500000000000000000000000",
        "expirationHeight": "7776000",
        "expirationTime": 1575268146,
        "revokeTime": 0
    }
}
```
:::

## contract_getSBPVoteList
Return current number of votes of all SBP nodes

- **Parameters**: 

- **Returns**: 
  - `Array<SBPVoteInfo>`
    - `sbpName`: `string` Name of SBP
    - `blockProducingAddress`: `string address` Block producing addresss
    - `votes`: `string bigint` Number of votes

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getSBPVoteList",
	"params": []
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": [
        {
            "sbpName": "s1",
            "blockProducingAddress": "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906",
            "votes": "100000000000000000000"
        },
        {
            "sbpName": "s2",
            "blockProducingAddress": "vite_0acbb1335822c8df4488f3eea6e9000eabb0f19d8802f57c87",
            "votes": "50000000000000000000"
        }
    ]
}
```
:::

## contract_getVotedSBP
Return voting information by account

- **Parameters**: 
  * `string address`: Address of voting account

- **Returns**: 
  - `VoteInfo`
    - `blockProducerName`: `string` Name of SBP
    - `status`: `uint8` Status of registration. For cancelled SBP node, `2` is filled, otherwise `1` for valid SBPs
    - `votes`: `string bigint` Number of votes, equivalent to account balance
  
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "1.0",
	"id": 1,
	"method": "contract_getVotedSBP",
	"params": [
		"vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a"
	]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "blockProducerName": "s1",
        "status": 1,
        "votes": "599960989999999999999999997"
    }
}
```
:::

## vote_getVoteDetails
Return voting details of all SBP nodes by cycle

- **Parameters**: 
  * `string uint64`: Index of cycle

- **Returns**: 
  - `Array<SBPVoteDetail>`
    - `blockProducerName`: `string` Name of SBP
    - `totalVotes`: `string bigint` The total number of votes
    - `blockProducingAddress`: `string address` Block producing address
    - `historyProducingAddresses`: `Array<string address>` Historical block producing address list
    - `addressVoteMap`: `map<string address, string bigint>` Voting details

- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "1.0",
	"id": 1,
	"method": "contract_getSBPVoteDetailsByCycle",
	"params": ["104"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": [
        {
            "blockProducerName": "s1",
            "totalVotes": "100000000000000000000",
            "blockProducingAddress": "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906",
            "historyProducingAddresses": [
                "vite_e41be57d38c796984952fad618a9bc91637329b5255cb18906"
            ],
            "addressVoteMap": {
                "vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a": "100000000000000000000"
            }
        },
        {
            "blockProducerName": "s2",
            "totalVotes": "50000000000000000000",
            "blockProducingAddress": "vite_0acbb1335822c8df4488f3eea6e9000eabb0f19d8802f57c87",
            "historyProducingAddresses": [
                "vite_0acbb1335822c8df4488f3eea6e9000eabb0f19d8802f57c87"
            ],
            "addressVoteMap": {
                "vite_56fd05b23ff26cd7b0a40957fb77bde60c9fd6ebc35f809c23": "50000000000000000000"
            }
        }
    ]
}
```
:::

## contract_getTokenInfoList
Return a list of all tokens issued

- **Parameters**: 
  * `int`: Page index, starting from 0
  * `int`: Page size

- **Returns**: 
  - `TokenListInfo`
    - `totalCount`: `int` The total number of tokens
    - `tokenInfoList`: `Array<TokenInfo>`
      - `tokenName`: `string`  Token name
      - `tokenSymbol`: `string`  Token symbol
      - `totalSupply`: `big.Int` Total supply
      - `decimals`: `uint8` Decimal places
      - `owner`: `Address` Token owner
      - `isReIssuable`: `bool`  Whether the token can be re-issued
      - `maxSupply`: `big.Int`  Maximum supply
      - `ownBurnOnly`: `bool`  Whether the token can be burned by the owner only
      - `tokenId`: `TokenId` Token ID
      - `index`: `uint16` Token index between 0-999. For token having the same symbol, sequential indexes will be allocated according to when the token is issued.
  
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getTokenInfoList",
	"params": [0, 10]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "totalCount": 1,
        "tokenInfoList": [
            {
                "tokenName": "Vite Token",
                "tokenSymbol": "VITE",
                "totalSupply": "1000000000000000000000000000",
                "decimals": 18,
                "owner": "vite_0000000000000000000000000000000000000004d28108e76b",
                "tokenId": "tti_5649544520544f4b454e6e40",
                "maxSupply": "115792089237316195423570985008687907853269984665640564039457584007913129639935",
                "isReIssuable": true,
                "index": 0,
                "isOwnerBurnOnly": false
            }
        ]
    }
}
```
:::

## contract_getTokenInfoById
Return token information

- **Parameters**: 
  * `string tokenId`: Token id

- **Returns**: 
  - `TokenInfo`
    - `tokenName`: `string`  Token name
    - `tokenSymbol`: `string`  Token symbol
    - `totalSupply`: `big.Int` Total supply
    - `decimals`: `uint8` Decimal places
    - `owner`: `Address` Token owner
    - `isReIssuable`: `bool`  Whether the token can be re-issued
    - `maxSupply`: `big.Int`  Maximum supply
    - `ownBurnOnly`: `bool`  Whether the token can be burned by the owner only
    - `tokenId`: `TokenId` Token ID
    - `index`: `uint16` Token index between 0-999. For token having the same symbol, sequential indexes will be allocated according to when the token is issued.
  
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getTokenInfoById",
	"params": ["tti_5649544520544f4b454e6e40"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "tokenName": "VITE",
        "tokenSymbol": "VITE",
        "totalSupply": "999369292029736282857580488",
        "decimals": 18,
        "owner": "vite_0000000000000000000000000000000000000004d28108e76b",
        "tokenId": "tti_5649544520544f4b454e6e40",
        "maxSupply": "115792089237316195423570985008687907853269984665640564039457584007913129639935",
        "isReIssuable": true,
        "index": 0,
        "isOwnerBurnOnly": false
    }
}
```
:::

## contract_getTokenInfoListByOwner
Return a list of tokens issued by the given owner

- **Parameters**: 
  * `string address`: Address of token owner

- **Returns**: 
  - `Array<TokenInfo>` 
    - `tokenName`: `string`  Token name
    - `tokenSymbol`: `string`  Token symbol
    - `totalSupply`: `big.Int` Total supply
    - `decimals`: `uint8` Decimal places
    - `owner`: `Address` Token owner
    - `isReIssuable`: `bool`  Whether the token can be re-issued
    - `maxSupply`: `big.Int`  Maximum supply
    - `ownBurnOnly`: `bool`  Whether the token can be burned by the owner only
    - `tokenId`: `TokenId` Token ID
    - `index`: `uint16` Token index between 0-999. For token having the same symbol, sequential indexes will be allocated according to when the token is issued.
  
- **Example**:
::: demo
```json tab:Request
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "contract_getTokenInfoListByOwner",
	"params": ["vite_0000000000000000000000000000000000000004d28108e76b"]
}
```
```json tab:Response
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": [
        {
            "tokenName": "VITE",
            "tokenSymbol": "VITE",
            "totalSupply": "999411106171319027184734227",
            "decimals": 18,
            "owner": "vite_0000000000000000000000000000000000000004d28108e76b",
            "tokenId": "tti_5649544520544f4b454e6e40",
            "maxSupply": "115792089237316195423570985008687907853269984665640564039457584007913129639935",
            "isReIssuable": true,
            "index": 0,
            "isOwnerBurnOnly": false
        }
    ]
}
```
:::
