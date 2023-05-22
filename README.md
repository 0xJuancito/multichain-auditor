# Multichain Auditing

Observations and tips for auditing protocols on different chains 🧐

## General Observations

### Block Time is Different on Different Chains

Block time refers to the time separating blocks. The average block time in Ethereum is 12s, but this value is different on different chains.

Example: 

```solidity
// 1 block every 12 sec -> 5 blocks / min
uint256 auctionDuration = 7200; // Auction duration lasts for one day (5 * 60 * 24 = 7200)
```

💡 Look for harcoded time values dependent on the `block.number` that may only be valid on Mainnet.

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
