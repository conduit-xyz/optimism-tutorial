# Bridging ETH with the Conduit AND Optimism SDK

[![Discord](https://img.shields.io/discord/667044843901681675.svg?color=768AD4&label=discord&logo=https%3A%2F%2Fdiscordapp.com%2Fassets%2F8c9701b98ad4372b58f13fd9f65f966e.svg)](https://discord-gateway.optimism.io)
[![Twitter Follow](https://img.shields.io/twitter/follow/optimismFND.svg?label=optimismFND&style=social)](https://twitter.com/optimismFND)

This tutorial teaches you how to use the [Optimism SDK](https://sdk.optimism.io/) to transfer ETH between Layer 1 (Ethereum) and Layer 2 (Optimism).


## Setup

1. Ensure your computer has:
   - [`git`](https://git-scm.com/downloads)
   - [`node`](https://nodejs.org/en/)
   - [`yarn`](https://classic.yarnpkg.com/lang/en/docs/install/#mac-stable)

1. Clone this repository and enter it.

   ```sh
   git clone https://github.com/ethereum-optimism/optimism-tutorial.git
   cd optimism-tutorial/cross-dom-bridge-eth
   ```

1. Install the necessary packages.

   ```sh
   yarn
   ```

1. Go to https://app.conduit.xyz/published/view/conduit-opstack-demo-nhl9xsg0wg to view the information for the OP-stack devnet


## Run the sample code

The sample code is in `index.js`, execute it.

`PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 node index.js`

### Expected output

```
Deposit ETH
On L1:904625697166532776746648320380374280103671755200316906557262200592 Gwei    On L2:904625697166532776746648320380374280103671755200316906559262375061 Gwei
Transaction hash (on L1): 0x52de853aa246f606f15f54b971ed17c4a837981bbda5b64bb516aeeb6d46f6b2
Waiting for status to change to RELAYED
Time so far 8.938 seconds
On L1:904625697166532776746648320380374280103671755200316906556262053171 Gwei    On L2:904625697166532776746648320380374280103671755200316906560262375061 Gwei
depositETH took 9.633 seconds
```

## How does it work?


```js
#! /usr/local/bin/node

// Transfers between L1 and L2 using the Optimism SDK

const ethers = require("ethers")
const optimismSDK = require("@eth-optimism/sdk")
const conduitSDK = require('@conduitxyz/sdk');
require('dotenv').config()

```

The libraries we need: [`ethers`](https://docs.ethers.io/v5/), [`dotenv`](https://www.npmjs.com/package/dotenv) and the Conduit and Optimism SDK themselves.

```js
// Your settlement layer rpc url here
const l1Url = `https://l1-conduit-opstack-demo-nhl9xsg0wg.t.conduit.xyz`
// Your conduit rpc url here
const l2Url = `https://l2-conduit-opstack-demo-nhl9xsg0wg.t.conduit.xyz`
const privateKey = process.env.PRIVATE_KEY
```

Configuration, read from `.env`.


```js
// Global variable because we need them almost everywhere
let crossChainMessenger
let addr    // Our address
```


The configuration parameters required for transfers.

### `getSigners`

This function returns the two signers (one for each layer). 

```js
const getSigners = async () => {
    const l1RpcProvider = new ethers.providers.JsonRpcProvider(l1Url)    
    const l2RpcProvider = new ethers.providers.JsonRpcProvider(l2Url)
```

The first step is to create the two providers, each connected to an endpoint in the appropriate layer.

```js
    const l1RpcProvider = new ethers.providers.JsonRpcProvider(l1Url)
    const l2RpcProvider = new ethers.providers.JsonRpcProvider(l2Url)
    const l1Wallet = new ethers.Wallet(privateKey, l1RpcProvider)
    const l2Wallet = new ethers.Wallet(privateKey, l2RpcProvider)
```    

Finally, create and return the wallets.
We need to use wallets, rather than providers, because we need to sign transactions.


### `setup`

This function sets up the parameters we need for transfers.

```js
const setup = async() => {
  const [l1Signer, l2Signer] = await getSigners()
  addr = l1Signer.address
```

Get the signers we need, and our address.

```js
  // The network slug is available in the Network Information tab here: https://app.conduit.xyz/published/view/conduit-opstack-demo-nhl9xsg0wg
  let config = await conduitSDK.getOptimismConfiguration('conduit:conduit-opstack-demo-nhl9xsg0wg');
  config.l1SignerOrProvider = l1Signer
  config.l2SignerOrProvider = l2Signer
    
  crossChainMessenger = new optimismSDK.CrossChainMessenger(config)
```

Create the [`CrossChainMessenger`](https://sdk.optimism.io/classes/crosschainmessenger) object that we use to transfer assets. Here we generate a config at runtime given the `slug` of the Conduit chain. This queries Conduit's servers for the addresses of the rollups contracts and other metadata necessary for the CrossChainMessenger.


### Variables that make it easier to convert between WEI and ETH

Both ETH and DAI are denominated in units that are 10^18 of their basic unit.
These variables simplify the conversion.

```js
const gwei = 1000000000n
const eth = gwei * gwei   // 10^18
const centieth = eth/100n
```

### `reportBalances`

This function reports the ETH balances of the address on both layers.

```js
const reportBalances = async () => {
  const l1Balance = (await crossChainMessenger.l1Signer.getBalance()).toString().slice(0,-9)
  const l2Balance = (await crossChainMessenger.l2Signer.getBalance()).toString().slice(0,-9)

  console.log(`On L1:${l1Balance} Gwei    On L2:${l2Balance} Gwei`)
}    // reportBalances
```



### `depositETH`

This function shows how to deposit ETH from Ethereum to Optimism.

```js
const depositETH = async () => {

  console.log("Deposit ETH")
  await reportBalances()
```

To show that the deposit actually happened we show before and after balances.

```js  
  const start = new Date()

  const response = await crossChainMessenger.depositETH(gwei)  
```

[`crossChainMessenger.depositETH()`](https://sdk.optimism.io/classes/crosschainmessenger#depositETH-2) creates and sends the deposit transaction on L1.

```js
  console.log(`Transaction hash (on L1): ${response.hash}`)
  await response.wait()
```

Of course, it takes time for the transaction to actually be processed on L1.

```js
  console.log("Waiting for status to change to RELAYED")
  console.log(`Time so far ${(new Date()-start)/1000} seconds`)
  await crossChainMessenger.waitForMessageStatus(response.hash, 
                                                  optimismSDK.MessageStatus.RELAYED) 
```

After the transaction is processed on L1 it needs to be picked up by an off-chain service and relayed to L2. 
To show that the deposit actually happened we need to wait until the message is relayed. 
The [`waitForMessageStatus`](https://sdk.optimism.io/classes/crosschainmessenger#waitForMessageStatus) function does this for us.
[Here are the statuses we can specify](https://sdk.optimism.io/enums/messagestatus).

The third parameter (which is optional) is a hashed array of options:
- `pollIntervalMs`: The poll interval
- `timeoutMs`: Maximum time to wait

```js
  await reportBalances()    
  console.log(`depositETH took ${(new Date()-start)/1000} seconds\n\n`)
}     // depositETH()
```

Once the message is relayed the balance change on Optimism is practically instantaneous.
We can just report the balances and see that the L2 balance rose by 1 gwei.

### `withdrawETH`

This function shows how to withdraw ETH from Optimism to Ethereum.

```js
const withdrawETH = async () => { 
  
  console.log("Withdraw ETH")
  const start = new Date()  
  await reportBalances()

  const response = await crossChainMessenger.withdrawETH(centieth)
```

For deposits it was enough to transfer 1 gwei to show that the L2 balance increases.
However, in the case of withdrawals the withdrawing account needs to be pay for finalizing the message, which costs more than that.

By sending 0.01 ETH it is guaranteed that the withdrawal will actually increase the L1 ETH balance instead of decreasing it.

```js
  console.log(`Transaction hash (on L2): ${response.hash}`)
  await response.wait()

  console.log("Waiting for status to change to IN_CHALLENGE_PERIOD")
```

There are two wait periods for a withdrawal:

1. Until the status root is written to L1. 
1. The challenge period.

You can read more about this [here](https://community.optimism.io/docs/developers/bridge/messaging/#for-optimism-l2-to-ethereum-l1-transactions).

```js
  console.log(`Time so far ${(new Date()-start)/1000} seconds`)  
  await crossChainMessenger.waitForMessageStatus(response.hash, 
    optimismSDK.MessageStatus.IN_CHALLENGE_PERIOD)
  console.log("In the challenge period, waiting for status READY_FOR_RELAY") 
  console.log(`Time so far ${(new Date()-start)/1000} seconds`)  
  await crossChainMessenger.waitForMessageStatus(response.hash, 
                                                optimismSDK.MessageStatus.READY_FOR_RELAY)
```

Wait until the state that includes the transaction gets past the challenge period, at which time we can finalize (also known as claim) the transaction.

```js
                                                
  console.log("Ready for relay, finalizing message now")
  console.log(`Time so far ${(new Date()-start)/1000} seconds`)  
  await crossChainMessenger.finalizeMessage(response)
```

Finalizing the message also takes a bit of time.

```js
  console.log("Waiting for status to change to RELAYED")
  console.log(`Time so far ${(new Date()-start)/1000} seconds`)  
  await crossChainMessenger.waitForMessageStatus(response, 
    optimismSDK.MessageStatus.RELAYED) 
  await reportBalances()   
  console.log(`withdrawETH took ${(new Date()-start)/1000} seconds\n\n\n`)  
}     // withdrawETH()
```


### `main`

A `main` to run the setup followed by both operations.

```js
const main = async () => {    
    await setup()
    await depositETH()
    await withdrawETH() 
}  // main



main().then(() => process.exit(0))
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })
```

## Conclusion

You should now be able to write applications that use Conduit's and Optimism's SDK and bridge to transfer ETH between layer 1 and layer 2. 
