# Multichain Auditor

Observations and tips for auditing protocols on different chains 🧐

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

---

## Ethereum

## Arbitrum

### References

- [Arbitrum/Ethereum Differences](https://developer.arbitrum.io/arbitrum-ethereum-differences)

## Optimism

### References

- [Differences between Ethereum and Optimism](https://community.optimism.io/docs/developers/build/differences/)

## zkSync Era

### References

- [Differences from Ethereum](https://era.zksync.io/docs/dev/building-on-zksync/contracts/differences-with-ethereum.html)

## MoonBeam

### References

- [Differences Between Moonbeam and Ethereum](https://docs.moonbeam.network/learn/features/eth-compatibility/)

## BSC

- Decimals 
  - [USDT](https://bscscan.com/address/0x55d398326f99059ff775485246999027b3197955#readContract#F6)
  - [USDC](https://bscscan.com/address/0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d#readProxyContract#F3)

### References

- [BNB Smart Chain vs Polygon - Comparing the Differences](https://docs.bnbchain.org/docs/migration/evm-chains/chain-comparison)
- [Token Standard Comparison](https://docs.bnbchain.org/docs/migration/evm-chains/token-comparison)

## Polygon

## Polygon zkEVM

## Base

### References

- [Differences between Ethereum and Base](https://docs.base.org/differences/)
