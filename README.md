# Multichain Auditor

Observations and tips for auditing protocols on multiple chains 🧐

#### ✍️ Open to Contributions

If you see some error, or want to add an observation, please create an issue or a PR. References are greatly appreciated. You can also contact me on Twitter at [@0xJuancito](https://twitter.com/0xJuancito).

#### 📜 Disclaimer

Take the observations in this repository as a guideline and kickstarter to your findings. Judge the actual impact independently, and please **do not use them as a tool to spam audit contests**. Do your own research.

## Index

- [General Observations](#general-observations)
  - [Block time is not the same on different chains](#block-time-is-not-the-same-on-different-chains)
  - [Block production may not be constant](#block-production-may-not-be-constant)
  - [L2 Sequencer Uptime Feeds in Chainlink](#l2-sequencer-uptime-feeds-in-chainlink)
  - [Chainlink Price Feeds](#chainlink-price-feeds)
  - [AMM pools token0 and token1 order](#amm-pools-token0-and-token1-order)
  - [Modified Opcodes](#modified-opcodes)
  - [Support for the push0 opcode](#support-for-the-push0-opcode)
  - [Address Aliasing - tx.origin / msg.sender](#address-aliasing---txorigin--msgsender)
  - [tx.origin == msg.sender](#txorigin--msgsender)
  - [transfer, send and fixed gas operations](#txorigin--msgsender)
  - [Gas fees](#gas-fees)
  - [Frontrunning](#frontrunning)
  - [Signature replay](#signature-replay)
  - [Hardcoded Contract Addresses](#hardcoded-contract-addresses)
  - [ERC20 decimals](#erc20-decimals)
  - [Contracts Interface](#contracts-interface)
  - [Contracts Upgradability](#contracts-upgradability)
  - [Contracts may behave differently](#contracts-may-behave-differently)
  - [Precompiles](#precompiles)
  - [zkSync Era](#zksync-era)
- [Differences from Ethereum](#differences-from-ethereum)

## General Observations

### Block time is not the same on different chains

Block time refers to the time separating blocks. The average block time in [Ethereum](https://ethereum.org/en/developers/docs/blocks/#block-time) is 12s, but this value is different on different chains.

Example: 

```solidity
// 1 block every 12 sec -> 5 blocks / min
uint256 auctionDuration = 7200; // Auction duration lasts for one day (5 * 60 * 24 = 7200)
```

Example: [OZ Wizard](https://wizard.openzeppelin.com/#governor)

💡 Look for hardcoded time values dependent on the `block.number` that may only be valid on Mainnet.

### Block production may not be constant

`block.number` is NOT a reliable source of timing information for short terms.

On [Arbitrum](https://docs.arbitrum.io/time#example) it reflects the L1 block number, which is updated once per minute

💡 Look for the use of `block.number` as a time reference, especially on L2.
💡 Block time may change on the same chain [over time](https://etherscan.io/chart/blocktime).

📝 [1](https://github.com/code-423n4/2022-12-tigris-findings/issues/419) [2](https://github.com/code-423n4/2022-12-tigris-findings/issues/67)

### L2 Sequencer Uptime Feeds in Chainlink

From [Chainlink documentation](https://docs.chain.link/data-feeds/l2-sequencer-feeds):

> Optimistic rollup protocols have a sequencer that executes and rolls up the L2 transactions by batching multiple transactions into a single transaction.

> If a sequencer becomes unavailable, it is impossible to access read/write APIs that consumers are using and applications on the L2 network will be down for most users.

This means that if the project does not check if the sequencer is down, it can return stale results.

[Optimism Goerli Uptime Feed](https://goerli-optimism.etherscan.io/address/0x4C4814aa04433e0FB31310379a4D6946D5e1D353#readContract#F10)

Mitigations can be found on [Handling Arbitrum outages](https://docs.chain.link/data-feeds/l2-sequencer-feeds#handling-arbitrum-outages) and [Handling outages on Optimism and Metis](https://docs.chain.link/data-feeds/l2-sequencer-feeds#handling-outages-on-optimism-and-metis).

Example:

```solidity
function getPrice(address token) external view override returns (uint) {
    if (!isSequencerActive()) revert Errors.L2SequencerUnavailable();
    ...
}

function isSequencerActive() internal view returns (bool) {
    (, int256 answer, uint256 startedAt,,) = sequencer.latestRoundData();
    if (block.timestamp - startedAt <= GRACE_PERIOD_TIME || answer == 1)
        return false;
    return true;
}
```

💡 Check if the projects handles the scenarios where a sequencer is down on optimistic rollup protocols.

📝 [1](https://github.com/sherlock-audit/2023-02-bond-judging/issues/1) [2](https://github.com/sherlock-audit/2023-01-sentiment-judging/issues/16) [3](https://github.com/sherlock-audit/2022-11-sentiment-judging/issues/3) [4](https://github.com/sherlock-audit/2023-04-jojo-judging/issues/101) [5](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/278)

### Chainlink Price Feeds

> Chainlink Data Feeds provide data that is aggregated from many data sources by a decentralized set of independent node operators.

Chainlink provides more price feeds for some chains like [Ethereum](https://docs.chain.link/data-feeds/price-feeds/addresses/?network=ethereum) than others like [Base](https://docs.chain.link/data-feeds/price-feeds/addresses/?network=base) for example. On other chains, no feed may be supported. Also, the same feed like AAVE/USD may have one address on a chain like [Ethereum](https://etherscan.io/address/0x6Df09E975c830ECae5bd4eD9d90f3A95a4f88012), and another one on [Moonriver](https://moonriver.moonscan.io/address/0x37f35ef6735c594e6E803bC81577bAC759d8179C).

💡 Check that the price feed for the desired pair is supported on all of the deployed chains.

💡 Check that the correct addresses are set correctly for each chain if they are hardcoded.

### AMM pools `token0` and `token1` order

In Uniswap and derived AMMs: `token0` is the token with the lower sort order, while `token1` is the token with the higher sort order, as described on [Uniswap documentation](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#token0). This is valid for both v2 and v3 pools.

The order is important because that determines which one is the base token, and which one is the quote token. In other words, if the price is WETH/USDC or USDC/WETH.

As contracts may have different addresses on different chains, the token order can change. That is the case for example on Arbitrum, where the pair is [WETH/USDC](https://arbiscan.io/address/0xc31e54c7a869b9fcbecc14363cf510d1c41fa443#readContract#F16) while on Polygon it is [USDC/WETH](https://polygonscan.com/address/0x45dda9cb7c25131df268515131f647d726f50608#readContract#F16).

💡 Verify that the token orders are taken into account, and it is not assumed to be the same on all chains.

### Modified Opcodes

Some chains implement opcodes with some modification compared to Ethereum, or are not supported.

Optimism for example, [has a different implementation](https://community.optimism.io/docs/developers/build/differences/#modified-opcodes) of opcodes like `block.coinbase`, `block.difficulty`, `block.basefee`. `tx.origin` may also behave different if the it is an L1 => L2 transaction. It also implements some new opcodes like `L1BLOCKNUMBER`.

Arbitrum also [has some differences](https://developer.arbitrum.io/solidity-support) in some operations/opcodes like: `blockhash(x)`, `block.coinbase`, `block.difficulty`, `block.number`. `msg.sender` may also behave different for L1 => L2 "retryable ticket" transactions.

💡 Verify that the EVM opcodes and operations used by the protocol are compatible on all chains

### Support for the `push0` opcode

`push0` is an instruction which pushes the constant value 0 onto the stack. This opcode is still not supported by many chains and might be problematic for projects compiled with a version of Solidity `>= 0.8.20` (when it was introduced).

💡 Pay attention to projects using a Solidity version `>= 0.8.20` and check if it is supported on the deployed chains.

ℹ️ Arbitrum added support in [ArbOS 11](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/solidity-support) and Optimism introduced support for it on the [Canyon Upgrade](https://blog.oplabs.co/canyon-hardfork/) .

### Address Aliasing - `tx.origin` / `msg.sender`

On some chains like [Optimism](https://community.optimism.io/docs/developers/build/differences/#using-eth-in-contracts), because of the behaviour of the CREATE opcode, it is possible for a user to create a contract on L1 and on L2 that share the same address but have different bytecode.

This can break trust assumptions, because one contract may be trusted and another be untrusted. To prevent this problem the behaviour of the ORIGIN and CALLER opcodes (tx.origin and msg.sender) differs slightly between L1 and L2.

💡 Verify that the expected behaviour of `tx.origin` and `msg.sender` holds on all deployed chains

### `tx.origin == msg.sender`

From [Optimism documentation](https://community.optimism.io/docs/developers/build/differences/#pre-eip-155-support):

> On L1 Ethereum tx.origin is equal to msg.sender only when the smart contract was called directly from an externally owned account (EOA). However, on Optimism tx.origin is the origin on Optimism. It could be an EOA. However, in the case of messages from L1, it is possible for a message from a smart contract on L1 to appear on L2 with tx.origin == msg.sender. This is unlikely to make a significant difference, because an L1 smart contract cannot directly manipulate the L2 state. However, there could be edge cases we did not think about where this matters.

💡 Verify that the expected behavior of `tx.origin` and `msg.sender` holds on all deployed chains

### Cross-chain message vulnerabilities

Some protocols work by sending cross-chain messages to their counterpart contracts on the other chains. This can lead to vulnerabilities like authorization issues, or issues with relayers.

💡 Look for cross-chain messages implementations and verify the correct permissions and functionality considering all the actors involved

📝 [1](https://github.com/code-423n4/2022-12-pooltogether-findings/issues/60) [2](https://github.com/sherlock-audit/2023-01-derby-judging/issues/309) [3](https://github.com/sherlock-audit/2023-01-derby-judging/issues/325)

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
- Frontrunning operations to prevent txns to be executed during a time frame (liquidations, complete auctions, etc.)
- Griefing attacks against the protocol

Although cheaper, each case should be analyzed to check if it is economically viable to actually be considered an attack.

💡 Analyze attack vectors that require low gas fees or where a considerable numbers of transactions have to be executed

📝 [1](https://github.com/sherlock-audit/2023-02-surge-judging/issues/109)

### Frontrunning

Frontrunning is possible on chains that have a mempool or a way to read proposed transactions before they are executed.

It is possible on some chains like Ethereum, although expensive because of gas costs. It is possible at a cheaper cost on other chains like Polygon. 

But it may be [very difficult](https://help.optimism.io/hc/en-us/articles/4444375174299-Is-transaction-front-running-possible-on-Optimism-) on chains like Optimism [with a private mempool](https://community.optimism.io/docs/developers/bedrock/differences/#mempool)

💡 Verify if a frontrunning attack is possible due to chain constraints or economic viability

### Signature replay across chains

If a contract is deployed on multiple chains and uses signatures, it may be possible to reuse a signature used on one chain and execute the same transaction on another chain.

To prevent that, it is important that the signed data contains the chain id where it should be executed.

Example from [UniswapV2](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2ERC20.sol#L34):

```solidity
constructor() public {
    uint chainId;
    assembly {
        chainId := chainid
    }
    DOMAIN_SEPARATOR = keccak256(
        abi.encode(
            keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
            keccak256(bytes(name)),
            keccak256(bytes('1')),
@>          chainId,                   // @audit
            address(this)
        )
    );
}

function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external {
    require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');
    bytes32 digest = keccak256(
        abi.encodePacked(
            '\x19\x01',
@>          DOMAIN_SEPARATOR, // @audit
            keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))
        )
    );
    address recoveredAddress = ecrecover(digest, v, r, s);
    require(recoveredAddress != address(0) && recoveredAddress == owner, 'UniswapV2: INVALID_SIGNATURE');
    _approve(owner, spender, value);
}
```

💡 Check that the data from the signed hash contains the chain id

📝 [1](https://github.com/code-423n4/2022-06-connext-findings/issues/144) [2](https://solodit.xyz/issues/7234) [3](https://solodit.xyz/issues/16276)

### Hardcoded Contract Addresses

Projects sometimes deploy their contracts on the same addresses over different chains but that is not always the case.

Take WETH as an example. Its address on Ethereum is [0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2](https://etherscan.io/token/0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2), but [0x7ceb23fd6bc0add59e62ac25578270cff1b9f619](https://polygonscan.com/token/0x7ceb23fd6bc0add59e62ac25578270cff1b9f619) on Polygon.

💡 Verify external contract addresses for the chains where the contracts are deployed

📝 [1](https://github.com/sherlock-audit/2023-01-derby-judging/issues/308)

### ERC20 decimals

Some ERC20 tokens have different `decimals` on different chains. Even some popular ones like USDT and USDC have 6 decimals on Ethereum, and 18 decimals on BSC for example:

- [USDT on Ethereum](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#readContract#F6) - 6 decimals
- [USDC on Ethereum](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48#readProxyContract#F11) - 6 decimals
- [USDT on BSC](https://bscscan.com/address/0x55d398326f99059ff775485246999027b3197955#readContract#F6) - 18 decimals
- [USDC on BSC](https://bscscan.com/address/0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d#readProxyContract#F3) - 18 decimals

A more exhaustive list can be found in the [tokens-decimals](https://github.com/magnetto90/tokens-decimals) repository by [@magnetto90](https://github.com/magnetto90).

💡 Check that the correct `decimals` are set for the deployed chains if the token values are hardcoded.

### Contracts Interface

Some contracts have a slightly different interface on different chains, which may break compatibility. 

USDT for example is missing its return value on Ethereum as the ERC20 specification suggests, but it is compliant on that aspect on Polygon. This may [lead to some vulnerabilities](https://github.com/d-xo/weird-erc20#missing-return-values) on some chains, while not on others.

[USDT on Ethereum](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code):

```solidity
  function transfer(address _to, uint _value) public whenNotPaused {
```

[USDT Implementation](https://polygonscan.com/address/0x7ffb3d637014488b63fb9858e279385685afc1e2#code) | [USDT Proxy](https://polygonscan.com/token/0xc2132d05d31c914a87c6611c10748aeb04b58e8f#readProxyContract) on Polygon:

```solidity
  function transfer(address recipient, uint256 amount) public virtual override returns (bool) {
```

[ERC20 transfer specification](https://eips.ethereum.org/EIPS/eip-20):

```solidity
  function transfer(address _to, uint256 _value) public returns (bool success)
```

💡 Verify that the contracts respect the same interface on different chains, or that sufficient mitigations are taken.

### Contracts Upgradability

Some contracts are immutable on a chain but upgradeable on others, like [USDT in Ethereum](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code) vs [USDT in Polygon](https://polygonscan.com/token/0xc2132d05d31c914a87c6611c10748aeb04b58e8f#code).

💡 Double-check the upgradability of contracts on different chains and evaluate their implications.

### Contracts may behave differently

Contracts deployed on different chains may behave differently.

On the XDai chain, USDC, WBTC, and WETH contained post-transfer callback procedures, as opposed to their traditional ERC20 implementations on other chains with no callback.

That enabled the possibility of a re-entrancy attack that was exploited and ultimately [derived on the fork of the chain](https://forum.gnosis.io/t/gip-31-should-gnosis-chain-perform-a-hardfork-to-upgrade-the-token-contract-vulnerable-to-the-reentrancy-attack/4134).

💡 Check that implementations of contracts match on different chains, or that their differences won't incur on any new vulnerability.

### Precompiles

Chains have precompiled contracts on different addresses like [Arbitrum](https://developer.arbitrum.io/arbos/precompiles) or [Optimism](https://github.com/ethereum-optimism/optimism/blob/develop/specs/predeploys.md). Care has to be taken if some is used that is not available, works differently or is on a different address.

💡 Double-check the use of precompiled contracts, their addresses, and their compatibility

### zkSync Era

zkSync Era has many differences from Ethereum on EVM instructions like `CREATE`, `CREATE2`, `CALL`, `STATICCALL`, `DELEGATECALL`, `MSTORE`, `MLOAD`, `CALLDATALOAD,` `CALLDATACOPY`, etc.

💡 Double-check the compatibility of the contracts when being deployed to zkSync Era

---

## Differences from Ethereum

Some blockchains have articles explaining their differences with Ethereum or other EVM chains. Here's a list of official docs:

- [Arbitrum vs Ethereum](https://developer.arbitrum.io/arbitrum-ethereum-differences)
- [Optimism vs Ethereum](https://docs.optimism.io/chain/differences)
- [zkSync Era vs Ethereum](https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions)
- [Linea vs Ethereum](https://docs.linea.build/build-on-linea/ethereum-differences)
- [Moonbeam vs Ethereum](https://docs.moonbeam.network/learn/features/eth-compatibility/)
- [Base vs Ethereum](https://docs.base.org/differences/)
- [Celo vs Ethereum](https://docs.celo.org/developer/migrate/from-ethereum#:~:text=Key%20differences%20between%20Celo%20and%20Ethereum%E2%80%8B&text=Celo%20allows%20users%20to%20pay,pay%20transaction%20fees%20in%20Ether)
- [opBNB vs Base](https://docs.bnbchain.org/bnb-opbnb/faq/protocol-faqs/?h=difference#what-is-the-difference-between-opbnb-and-other-optimism-based-layer-2-solution-like-base)
- [Filecoin vs Ethereum](https://docs.filecoin.io/smart-contracts/filecoin-evm-runtime/differences-with-ethereum/)
- [Gnosis vs Ethereum](https://docs.gnosischain.com/about/specs/hard-forks/dencun#differences-with-ethereum-mainnet)
- [Tron vs Ethereum](https://developers.tron.network/v4.4.0/docs/vm-vs-evm#Differences%20from%20EVM)

## EVM Compatible Chains Diff

Check [evm-diff](https://github.com/mds1/evm-diff) repository and the website [evmdiff.com](https://evmdiff.com) to diff EVM-compatible chains in a friendly format. It's an amazing tool created by [@mds1](https://github.com/mds1)
