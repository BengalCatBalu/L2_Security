# 🛠️ DApp Developer Security Checklist for Rollup Integration

## 1. Chain & Opcode Compatibility

- [ ] Are hardcoded chain IDs used consistently across contracts?
- [ ] Is `tx.origin` and `msg.sender` behavior consistent across all Rollups?
  - 🚨 *Issue example*: Unexpected contract permissions due to differences in `tx.origin`.
- [ ] Are all EVM opcodes used compatible across Optimism, Arbitrum, zkSync, and Ethereum mainnet?
  - 🚨 *Issue example*: Opcodes like `PUSH0` not supported universally (e.g., zkSync).

## 2. Block.number & Block.timestamp Dependency

- [ ] Is the contract logic independent from hardcoded `block.number` or fixed intervals?
  - 🚨 *Issue example*: Optimism has ~2s blocks, Ethereum ~12s. Timings can diverge significantly.
- [ ] Are timestamp-dependent contracts tolerant of different average block times?
- [ ] Have Arbitrum-specific multi-tx block implications been considered?

## 3. Token Standards & ERC20 Decimals

- [ ] Is there consistency in ERC20 decimals across deployed Rollups?
  - 🚨 *Issue example*: USDT or USDC decimal discrepancies across chains.
- [ ] Does the contract logic adapt correctly to different token decimals?

## 4. Cross-chain Messaging & Bridges

- [ ] Have messaging implementations (bridges) been reviewed thoroughly?
- [ ] Are replay protection mechanisms (e.g., chain-specific nonce) in place?
  - 🚨 *Issue example*: Replay attacks using signed messages across Rollups.
- [ ] Is there a whitelist of verified and compatible chains?

## 5. Upgradeability Across Chains

- [ ] Is contract upgradeability consistent across deployed chains?
- [ ] Have upgradeability governance processes been checked across all Rollups?
  - 🚨 *Issue example*: Different upgrade permissions causing divergence in state or behavior.

## 6. Signature Replay Vulnerabilities

- [ ] Are signatures strictly validated with chain-specific data included?
  - 🚨 *Issue example*: An attacker can reuse signatures intended for another chain.
- [ ] Is `chain_id` always explicitly verified in signed payloads?

## 7. Low Gas Fee Exploits & DoS

- [ ] Is there protection against mass transactions exploiting low fees on certain Rollups?
  - 🚨 *Issue example*: Spam attacks economically feasible due to cheaper fees on zkSync or Optimism.
- [ ] Are contracts resilient to transaction spam causing DoS?

## 8. Hardcoded Addresses & Parameters

- [ ] Have all hardcoded addresses (e.g., routers like WETH) been verified per chain?
  - 🚨 *Issue example*: WETH router address varies significantly by Rollup.
- [ ] Are critical parameters dynamic and adjustable per deployment?

## 9. AMM Pools token0/token1 Order

- [ ] Is token ordering logic (Uniswap-style `token0/token1`) stable across chains?
  - 🚨 *Issue example*: Different address ordering causing AMM pairs mismatch between chains.

## 10. RPC Latency & State Consistency

- [ ] Does frontend/RPC integration correctly handle varied RPC response latencies?
- [ ] Are state updates verified consistently across RPC endpoints?

## 11. L2-specific MEV Considerations

- [ ] Has the DApp accounted for MEV risk unique to Optimistic and ZK-rollups?
  - 🚨 *Issue example*: Sequencer front-running transactions on Optimism due to centralized sequencing.
- [ ] Are protections (such as private mempool integrations or encrypted mempool) considered?

## 12. L2 Sequencer Uptime & Reliability (e.g., Chainlink feeds)

- [ ] Do contracts verify sequencer uptime/availability?
  - 🚨 *Issue example*: Funds frozen if sequencer goes down without fallback mechanisms.
- [ ] Is a fallback mechanism in place if Chainlink sequencer uptime feed is unavailable?

## 13. Ethereum Feature Drift & Solidity Versions

- [ ] Is Solidity compiler version consistent and supported across all Rollups?
  - 🚨 *Issue example*: zkSync may lag behind mainline Solidity version support.

## 14. Production Deployments & Testing

- [ ] Has the contract been tested comprehensively in actual rollup environments?
- [ ] Are real-world edge cases from existing multichain deployments integrated into testing?

## 15. Documentation & User Guidance

- [ ] Are rollup-specific behaviors and risks clearly documented for end-users?
- [ ] Have deployment instructions and best practices been specified per rollup?

## 16. Miscellaneous & Additional Checks

- [ ] Is the contract resilient against sudden changes or hard forks in underlying L2 chains?
- [ ] Has contract logic been audited specifically with a focus on rollup differences?

