# Ethereum

## Blocks

- **Average Block Time**: [12 seconds](https://etherscan.io/chart/blocktime)

## Bounties

- [Projects on Immunefi](https://immunefi.com/explore/?filter=ecosystem%3DETH)

## Precompiles

| Address       | Name          | Description  |
|:-------------:|:-------------| :-----|
| `0x01`        | ecRecover | Elliptic curve digital signature algorithm (ECDSA) public key recovery function |
| `0x02`        | SHA2-256  | Hash function |
| `0x03`        | RIPEMD-160| Hash function |
| `0x04`        | Identity  | Returns the input |
| `0x05`        | modexp | 	Arbitrary-precision exponentiation under modulo |
| `0x06`        | ecAdd | Point addition (ADD) on the elliptic curve 'alt_bn128' |
| `0x07`        | ecMul | Scalar multiplication (MUL) on the elliptic curve 'alt_bn128' |
| `0x08`        | ecPairing | Bilinear function on groups on the elliptic curve 'alt_bn128' |
| `0x09`        | blake2f | Compression function F used in the BLAKE2 cryptographic hashing algorithm |
| `0x0A`        | zkPointEvaluation | Point evaluation for zk-rollups ([EIP-4844](https://eips.ethereum.org/EIPS/eip-4844#point-evaluation-precompile)) |

###### Sources

- [Geth Client](https://github.com/ethereum/go-ethereum/blob/63127f5443bbf4dd6c56fcb11236d35b1ecad848/core/vm/contracts.go#L98-L107)
- [RareSkills](https://www.rareskills.io/post/solidity-precompiles)
- [EVM Codes](https://www.evm.codes/precompiled?fork=shanghai)

## Tools

- [Block Explorer](https://etherscan.io/)
