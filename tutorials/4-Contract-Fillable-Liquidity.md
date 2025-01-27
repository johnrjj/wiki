One of the most powerful things that 0x has to offer is a large pool of liquidity with competitive pricing that can be used to implement “contract fillable liquidity”. Any developer that needs exchange functionality from within their own smart contract can use 0x to do so. Here are some examples of contract fillable liquidity:

-   A Dapp such as [CDP Saver](https://cdpsaver.com/) that aims to abstract the usage of MKR to interact with your CDP by performing swaps from ETH or DAI to MKR without any thought from the user.
-   A margin trading protocol, like [dYdX](https://medium.com/dydxderivatives/dai-usdc-market-launches-on-dydx-bde957c48e2e) that wants to give their users access to deep liquidity pools.
-   A simple script that sells ETH for DAI on a consistent basis over a week and deposits it into a lending protocol like Compound.

The following content will help explain 0x key concepts and describe how you can start implementing some of these examples, today. The walkthrough will be leveraging [asset-swapper](/docs/asset-swapper) to do most of the heavy lifting.

### Key Concepts

There are a few key concepts that will be good to understand in order to make best use of this guide:

#### 0x order

The 0x order is a message with a [specific structure](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#order-message-format). Conceptually, you can think of a 0x order as a promise from one user (the maker) to swap their token (the makerAsset) with another token (the takerAsset) at a specified rate. These orders can be fulfilled by interacting with the [0x exchange contract](https://etherscan.io/address/0x4f833a24e1f95d70f028921e27040ca56e09ab0b).

#### 0x exchange contract

The 0x exchange contract is a smart contract deployed on the [Ethereum mainnet](https://etherscan.io/address/0x4f833a24e1f95d70f028921e27040ca56e09ab0b) and most [testnets](https://0x.org/wiki#Deployed-Addresses). Users send transactions to this smart contract in order to fulfill 0x orders. There are many options for filling orders using the contract, they are listed [here](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#filling-orders).

#### 0x extension contracts

The 0x exchange contract is designed to be easily extendible through varying smart contracts that are referred to as extension contracts. Users can send their orders to an extension contract and expect augmented behavior that is not provided by the core 0x exchange contract. While any developer can develop and leverage extension contracts, the 0x core team has built, audited, and deployed a number of open source extension contracts. Learn more about them [here](https://0x.org/wiki#0x-Extensions).

### Enabling Contract Fillable Liquidity

Enabling contract fillable liquidity using 0x requires 3 steps:

1. Finding 0x orders
2. Passing 0x orders to a smart contract
3. Filling 0x orders with the 0x exchange contract

The library [asset-swapper](/docs/asset-swapper) is a convenience utility package that makes a number of these steps easier for developers. Begin by installing the library:

```
$ yarn add @0x/asset-swapper
```

#### Step 1: Finding 0x Orders

An easy way to find 0x orders is to query a 0x relayer via the [Standard Relayer REST API specification](http://sra-spec.s3-website-us-east-1.amazonaws.com/). Interactions with these APIs occur entirely off-chain. This means that Ethereum transactions are not involved yet. [Asset-swapper](/docs/asset-swapper) provides an easy to use interface that fetches the orders from a provided Standard Relayer REST API and provides a “price” quote for the swap.

```
import { SwapQuoter } from '@0x/asset-swapper';

const provider = web3.currentProvider;
const apiUrl = 'https://api.radarrelay.com/0x/v2/';

const swapQuoter = SwapQuoter.getSwapQuoterForStandardRelayerAPIUrl(provider, apiUrl);
```

With `SwapQuoter` you can easily request a ‘quote’ to sell X amount of a token or buy Y amount of a token:

```
const makerTokenAddress = '0x...';
const takerTokenAddress = '0x...';
const amount = new BigNumber(10);

// Requesting a quote for buying 10 units of makerToken
const quote = await swapQuoter.getMarketBuySwapQuoteAsync(
    makerTokenAddress,
    takerTokenAddress,
    amount
    );

// Requesting a quote to sell 10 units of takerToken for makerToken
const quote = await swapQuoter.getMarketSellSwapQuoteAsync(
    makerTokenAddress,
    takerTokenAddress,
    amount
    );

console.log(quote.orders);
```

#### Step 2: Passing 0x orders to a smart contract

Up to now, all interactions with the 0x protocol have been off-chain. To consume the output fetched off-chain in a smart contract, it needs to be transmitted on-chain.

The most intuitive way to pass 0x orders on-chain are via additional parameters passed along a smart contract call.

[Asset-swapper](/docs/asset-swapper) provides a `SwapQuoteConsumer` utility tool that converts a provided quote to a more intuitive format for passing on-chain via parameter or parameter(s).

A number of outputs can be provided by SwapQuoteConsumer:

-   calldata: with a provided quote, the consumer can generate executable calldata for 0x Exchange contracts. Simply pass the calldata on-chain through a `bytes calldata paramName` parameter in any function that requires 0x liquidity.
-   0x exchange parameter: For developers that want nuanced control of preparing 0x orders before transmitting them on chain, the consumer can provide all the parameters for a valid 0x exchange contract call. Developers can wrap these parameters in a tuple, convert it to a bytes array and into other data formats that work best for the smart contract interface.

Get executable `calldata` from a quote easily with `SwapQuoteConsumer`:

```
import { SwapQuoteConsumer } from '@0x/asset-swapper';

const provider = web3.currentProvider;

const swapQuoteConsumer = new SwapQuoteConsumer(provider);

const calldataInfo = await swapQuoteConsumer.getCalldataOrThrowAsync(quote);

const { calldataHexString } = calldataInfo;
```

Pass the `calldataHexString` on-chain via parameter like so:

```
const txHash = await yourSmartContract.methods
    .liquidityRequiringMethod(..., calldataHexString)
    .call(...);
```

#### Step 3: Filling 0x orders with the 0x exchange contract

In your smart contract, you can then call 0x exchange contracts to swap for tokens easily:

```
contract MyContract {

    public address zeroExExchangeAddress;

    function liquidityRequiringMethod(..., bytes calldata calldataHexString) {
        zeroExExchangeAddress.call(calldataHexString);
        ...
    }
}
```

_Note:_ Contract fillable liquidity integrations will be filling 0x orders “on behalf” of the user. Smart contract’s integrating 0x CFL would need to set an allowance for 0x exchange contracts before hand in the constructor or through an admin operated function.

There are many different ways to fill orders; developers in the 0x ecosystem have built a number of ways to interact with 0x protocol on-chain. For a complete list of options for filling 0x orders, check out the [0x protocol specification](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#filling-orders). A non-exhaustive list of great 0x integrations can be found here:

-   dYdX's exchange wrappers repo: https://github.com/dydxprotocol/exchange-wrappers
-   Set protocol’s exchange wrappers repo: https://github.com/SetProtocol/set-protocol-contracts/tree/master/contracts/core/exchange-wrappers
-   0x forwarder: https://github.com/0xProject/0x-monorepo/blob/development/contracts/exchange-forwarder/contracts/src/MixinForwarderCore.sol

### Comparing 0x to other on-chain liquidity providers

At a high level, [asset-swapper](/docs/asset-swapper) allows users to execute `marketBuy` or `marketSell` functions on the 0x protocol with relative ease. Analogously you can think of these functions as ‘swap’ functions in other liquidity provider’s interfaces:

0x:

```
(exchange || extension).marketBuy(...)
```

uniswap equivalent:

```
exchange.tokenToTokenSwapOutput(...)
```

kyber equivalent:

```
No direct equivalent
```

0x:

```
(exchange || extension).marketSell(...)
```

uniswap equivalent:

```
exchange.tokenToTokenSwapInput(...)
```

kyber equivalent:

```
kyberNetworkProxyContract.swapTokenToToken(...)
```

Kyber and Uniswap rely on purely on-chain mechanisms to provide liquidity; 0x, however, sources liquidity from off-chain entities called relayers or from 0x mesh, a peer-to-peer network. With this in mind, a number of off-chain computations must occur to prepare 0x liquidity for use in a smart contract.

[asset-swapper](/docs/asset-swapper) is a convenience javascript package that does most, if not all, of the off-chain work required for smart contract developers to easily leverage 0x protocol.

### Advanced Topics related to Asset-swapper

#### "Smart" SwapQuoteConsumer Nuances

`SwapQuoteConsumer` under the hood determines the best 0x smart contract interfaces to perform the swap specified.

The two main 0x smart contract interfaces are:

-   _Exchange_: The main 0x exchange protocol smart contract interface
    -   Specifically [asset-swapper](/docs/asset-swapper) interacts with `marketBuyNoThrow` and `marketSellNoThrow` functions
-   _Forwarder_: An extension contract that abstracts wrapping ether and a number of other computations; only supports WETH/ERCXX pairs. Read more about forwarder [here](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/forwarder-specification.md)
    -   Specifically [asset-swapper](/docs/asset-swapper) interacts with `marketBuyWithEth` and `marketSellWithEth` functions
-   (Future) _Coordinator_: An extension contract that enables the filling of coordinator orders; read more about coordinator 0x orders [here](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/coordinator-specification.md).

SwapQuoteConsumer has a router logic that, dependent on the takerAddress and asset trading pair, determine the best 0x interface to provide output for.

Generally, the router logic works as follows:

-   If the taker asset is not WETH, default to providing output for interacting directly with 0x exchange contracts
-   If the taker asset is WETH and the provided takerAddress custodies enough ether, consumer will provide output for interacting with _Forwarder_ extension contract.
-   Otherwise, default to output for 0x exchange contracts

Keep in mind that the provided output of SwapQuoteConsumer provides a toAddress parameter, this address can be used to verify which contract the provided output is for.

##### Ignoring router logic

If a `useConsumerType` is provided within the options of any call to `SwapQuoteConsumer`, the consumer instance will ignore the "router" logic that determines the best 0x interface and instead provide output for the interface specified by useConsumerType.

```
import { ConsumerType } from '@0x/asset-swapper';

const calldataInfo = await swapQuoteConsumer.getCalldataOrThrowAsync(quote,
    {
         useConsumerType: ConsumerType.Exchange,
         // or ConsumerType.Forwarder
    });
```

#### Interacting with fees

Dependent to where the orders are found, a fee, payed with ZRX, can be specified in an order that the taker would need to pay. The taker, for CFL integrations, is a smart contract. Developers will need to maintain a balance of ZRX in the smart contract to pay the fees for any order that contains a fee.

The fee can be offset to users of the smart contract with the provided quote from `SwapQuoter`:

```
const quote = await swapQuoter.getMarketBuySwapQuoteAsync(
    makerTokenAddress,
    takerTokenAddress,
    amount
    );
const feeOrders = quote.feeOrders;
const bestCaseQuoteInfo = quote.bestCaseQuoteInfo;
const feeTakerTokenAmount = bestCaseQuoteInfo.feeTakerTokenAmount;
const totalTakerTokenAmount = bestCaseQuoteInfo.totalTakerTokenAmount;
```

`FeeOrders` is an additional array of orders to buy ZRX with the takerToken specified when requesting the quote. The smart contract can fill the `feeOrders` to receive the necessary ZRX to fill the `orders` provided by the quote. The quote’s `quoteInfo` object also adjusts the `totalTakerTokenAmount` a user will need to pay to fulfill the swap.

The `quoteInfo` object found in quote can be used to extract the correct cost of filling the orders for a user.

##### Using SwapQuoteConsumer for orders with fees

As described by the **“Smart” SwapQuoteConsumer Nuances** section, `SwapQuoteConsumer` will prepare output for the most convenient 0x interface for CFL integrators. The current version of `SwapQuoteConsumer` prepares output from quotes with fees differently for each 0x interface:

-   _Exchange_: `feeOrders` are dropped and ignored. `SwapQuoteConsumer` will prepare output assuming developer will handle fee operations.
-   _Forwarder_: The `forwarder` extension contract conducts a number of fee related operations on behalf of the user; `SwapQuoteConsumer` will prepare output such that the forwarder will be consuming the `feeOrders`. The developer will not have to conduct any additional operations for fees.

##### Ignoring fees in SwapQuoter

Fetching, pruning, and selecting feeOrders can be a costly operation (time-wise). For time-sensitive CFL integrations that sources fee-less 0x orders or handles fees through custom means, an option is provided to ignore all fee operations conducted by `SwapQuoter`.

Simply pass the `shouldDisableRequestingFeeOrders` when requesting a quote from `SwapQuoter`.

```
const quote = await swapQuoter.getMarketBuySwapQuoteAsync(
    makerTokenAddress,
    takerTokenAddress,
    amount,
    {
        shouldDisableRequestingFeeOrders: true,
    });
```

#### Executing a SwapQuote with web3

`SwapQuoteConsumer` also provides a method to execute a `SwapQuote` with web3. This method powers products like [0x Instant](https://0x.org/instant) and helps build order execution experiences for exchanges like [0x Launchkit Frontend](https://0x.org/launch-kit).

Execute a `SwapQuote` in the following way:

```
const txHash = await swapQuoteConsumer.executeSwapQuoteOrThrowAsync(quote, {});
```

### Conclusion

That’s it, with a flexible suite of tools available, you too can find and fill 0x orders in order to provide your users with UX enhancing token swaps. Please reach out to us in the #questions channel under Dev Support in [Discord](https://discord.gg/d3FTX3M).

We also provide a range of other material and documentation to help you get started playing with 0x. Check them out:

-   Get familiar with other 0x key concepts and features by coding right in your browser: https://codesandbox.io/s/github/0xproject/0x-codesandbox
