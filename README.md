# Multichain Auditor

Observations and tips for auditing protocols on multiple chains 🧐

#### ✍️ Open to Contributions

If you see some error, or want to add an observation, please create an issue or a PR. References are greatly appreciated. You can also contact me via [Twitter](https://twitter.com/0xJuancito).

#### 📜 Disclaimer

Take the observations in this repository as a guideline and kickstarter to your findings. Judge the actual impact independently, and please do not use them as a tool to spam audit contests. Do your own research.

## General Observations

### Support for the `push0` opcode

`push0` is an instruction which pushes the constant value 0 onto the stack. This opcode is still not supported by many chains, like [Arbitrum](https://developer.arbitrum.io/solidity-support#Differences%20from%20Solidity%20on%20Ethereum) and might be problematic for projects compiled with a version of Solidity `>= 0.8.20` (when it was introduced).

💡 Pay attention to projects using a Solidity version `>= 0.8.20` and check if it is supported on the deployed chains.

### Block time is not the same on different chains

Block time refers to the time separating blocks. The average block time in [Ethereum](https://ethereum.org/en/developers/docs/blocks/#block-time) is 12s, but this value is different on different chains.

Example: 

```solidity
// 1 block every 12 sec -> 5 blocks / min
uint256 auctionDuration = 7200; // Auction duration lasts for one day (5 * 60 * 24 = 7200)
```

💡 Look for harcoded time values dependent on the `block.number` that may only be valid on Mainnet.

### Block production may not be constant

For some chains, `block.number` is NOT a reliable source of timing information. Especially in L2 like [Optimism](https://community.optimism.io/docs/developers/build/differences/#block-production-is-not-constant) for example.

💡 Look for the use of `block.number` as a time reference, especially on L2.

### `transfer`, `send` and fixed gas operations

`transfer` and `send` forward a hardcoded amount of gas and are [discouraged as gas costs can change](https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/). On certain chains that cost can be higher than in Mainnet, and can result in issues, like in [zkSync Era](https://twitter.com/zksync/status/1644139364270878720).

💡 Look for fixed gas operations like `transfer` or `send`.

### Gas fees

Transactions on Ethereum mainnet are much more expensive than on other chains. Chains with very low fees may open the possibility to implement attacks that require a large amount of transactions, or where the cost-benefit of the attack would now be profitable.

Examples:

- DOS on unbound arrays
- DOS by filling bound arrays
- Spamming that can incur in extra processing costs for the protocol
- An attack that only drains smaller amounts of wei that wouldn't be profitable with high gas fees
- Frontrunning operations to prevent txs to be executed during a time frame (liquidations, complete auctions, etc.)
- Griefing attacks against the protocol

Although cheaper, each case should be analyzed to check if it is economically viable to actually be considered an attack.

💡 Analyze attack vectors that require low gas fees or where a considerable numbers of transactions have to be executed

### L2 Sequencer Uptime Feeds in Chainlink

From [Chainlink documentation](https://docs.chain.link/data-feeds/l2-sequencer-feeds):

> Optimistic rollup protocols have a sequencer that executes and rolls up the L2 transactions by batching multiple transactions into a single transaction.

> If a sequencer becomes unavailable, it is impossible to access read/write APIs that consumers are using and applications on the L2 network will be down for most users.

This means that if the project does not check if the sequencer is down, it can return stale results.

Mitigations can be found on [Handling Arbitrum outages](https://docs.chain.link/data-feeds/l2-sequencer-feeds#handling-arbitrum-outages) and [Handling outages on Optimism and Metis](https://docs.chain.link/data-feeds/l2-sequencer-feeds#handling-outages-on-optimism-and-metis).

💡 Check if the projects handles the scenarios where a sequencer is down on optimistic rollup protocols.

### Chainlink Price Feeds

> Chainlink Data Feeds provide data that is aggregated from many data sources by a decentralized set of independent node operators.

Chainlink provides more price feeds for some chains like [Ethereum](https://docs.chain.link/data-feeds/price-feeds/addresses/?network=ethereum) than others like [Base](https://docs.chain.link/data-feeds/price-feeds/addresses/?network=base) for example. On other chains, no feed may be supported. Also, the same feed like AAVE/USD may have one address on a chain like [Ethereum](https://etherscan.io/address/0x6Df09E975c830ECae5bd4eD9d90f3A95a4f88012), and another one on [Moonriver](https://moonriver.moonscan.io/address/0x37f35ef6735c594e6E803bC81577bAC759d8179C).

💡 Check that the price feed for the desired pair is supported on all of the deployed chains.

💡 Check that the correct addresses are set correctly for each chain if they are hardcoded.

### ERC20 decimals

Some ERC20 tokens have different `decimals` on different chains. Even some popular ones like USDT and USDC have 6 decimals on Ethereum, and 18 decimals on BSC for example:

- [USDT on Ethereum](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#readContract#F6) - 6 decimals
- [USDC on Ethereum](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48#readProxyContract#F11) - 6 decimals
- [USDT on BSC](https://bscscan.com/address/0x55d398326f99059ff775485246999027b3197955#readContract#F6) - 18 decimals
- [USDC on BSC](https://bscscan.com/address/0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d#readProxyContract#F3) - 18 decimals

💡 Check that the correct `decimals` are set for the deployed chains if the token values are hardcoded.

### Contract addresses

Projects sometimes deploy their contracts on the same addresses over different chains but that is not always the case.

Take USDC as an example, whose address in Ethereum is [0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48) and [0x2791bca1f2de4661ed88a30c99a7a9449aa84174](https://polygonscan.com/token/0x2791bca1f2de4661ed88a30c99a7a9449aa84174) on Polygon.

💡 Verify external contract addresses for the chains where the contracts are deployed

### AMM pools `token0` and `token1` order

In Uniswap and derived AMMs: `token0` is the token with the lower sort order, while `token1` is the token with the higher sort order, as described on [Uniswap documentation](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#token0). This is valid for both v2 and v3 pools.

The order is important because that determines which one is the base token, and which one is the quote token. In other words, if the price is WETH/USDC or USDC/WETH.

This is the case for example on Optimism, where the pair is [WETH/USDC](https://info.uniswap.org/#/optimism/pools/0x85149247691df622eaf1a8bd0cafd40bc45154a9) while on Polygon it is [USDC/WETH](https://info.uniswap.org/#/polygon/pools/0x45dda9cb7c25131df268515131f647d726f50608).

💡 Verify that the token orders hold for all chains.

### Signature replay

If a contract is deployed on multiple chains and uses signatures, it may be possible to reuse a signature used on one chain and execute the same transaction on another chain.

To prevent that, it is important that the signed data contains the chain id where it should be executed.

💡 Check that the data from the signed hash contains the chain id

### Frontrunning

Frontrunning is possible on chains that have a mempool or a way to read proposed transactions before they are executed.

It is possible on some chains like Ethereum, although expensive because of gas costs. It is possible at a cheaper cost on other chains like Polygon. 

But it may be [very difficult](https://help.optimism.io/hc/en-us/articles/4444375174299-Is-transaction-front-running-possible-on-Optimism-) on chains like Optimism where [the sequencer has no mempool](https://community.optimism.io/docs/protocol/2-rollup-protocol/#optimistic-rollups-tl-dr).

💡 Verify if a frontrunning attack is possible due to chain constraints or economic viability

---

## Differences from Ethereum

Some blockchains have articles explaining their differences with Ethereum or other EVM chains. Here's a list of official docs:

- [Arbitrum/Ethereum Differences](https://developer.arbitrum.io/arbitrum-ethereum-differences)
- [Differences between Ethereum and Optimism](https://community.optimism.io/docs/developers/build/differences/)
- [zkSync Era: Differences from Ethereum](https://era.zksync.io/docs/dev/building-on-zksync/contracts/differences-with-ethereum.html)
- [Differences Between Moonbeam and Ethereum](https://docs.moonbeam.network/learn/features/eth-compatibility/)
- [Differences between Ethereum and Base](https://docs.base.org/differences/)
- [BNB Smart Chain vs Polygon - Comparing the Differences](https://docs.bnbchain.org/docs/migration/evm-chains/chain-comparison)
- [BSC Token Standard Comparison](https://docs.bnbchain.org/docs/migration/evm-chains/token-comparison)
