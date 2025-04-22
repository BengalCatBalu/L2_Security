# Rollup (Layer 2) Architect Security Checklist

## 1. State Validation & Proofs
- [ ] Is the state posted to L1 always valid and verifiable?
- [ ] Can invalid states be challenged and reverted efficiently?
- [ ] Are Fraud/Validity proofs robust and cover all fraud scenarios?
- [ ] Are anchor roots based on recent enough L2 blocks (e.g., no older than 6 months L1 timestamp)?
- [ ] Are anchor roots only accepted from dispute games created within the correct registry context?
- [ ] Are all input invariants (timestamps, chainId ordering, block numbers, gas limits) explicitly validated?

## 2. Data Availability
- [ ] Are transaction data available independently from the sequencer?
- [ ] Can users reconstruct rollup state purely from data on L1?
- [ ] Are fallback data-availability mechanisms in place?

## 3. Sequencer Failure
- [ ] Is there a fallback mechanism if the primary sequencer fails?
- [ ] Can users bypass the sequencer and submit directly to L1?
- [ ] Is auto-switching between sequencers clearly defined?

## 4. Proposer Failure
- [ ] Does a backup proposer mechanism exist if the main proposer stops?
- [ ] Is there handling of prolonged proposer inactivity?
- [ ] Is the proposer role clearly separated from sequencer role?

## 5. Exit Window and Upgradeability
- [ ] Is there a sufficient delay (exit window) for users before upgrades are activated?
- [ ] Can users securely withdraw assets if disagreeing with an upgrade?
- [ ] Is governance process secure against malicious upgrades?
- [ ] Are upgrade procedures safe even in case of human error (e.g., misuse of `upgradeTo` without `call`)?
- [ ] Are input parameters strictly validated during deployments (e.g., consistency of game types)?
- [ ] Is the system upgrade-safe regarding withdrawal finality and anchor state migration?

## 6. Transaction Finality & Chain Re-org
- [ ] Are transaction finality levels (pre-finality, soft, hard) clearly defined?
- [ ] Is rollup resistant to Ethereum re-org events?
- [ ] Are users clearly informed about transaction finality timing?

## 7. L1 Withdrawal Latency
- [ ] How long does it take to withdraw assets to L1?
- [ ] Is withdrawal protected against griefing and censorship attacks?
- [ ] Is there an emergency withdrawal procedure?
- [ ] Are SuperRoot-based withdrawals correctly validated for timestamps and chainId ordering?
- [ ] Are donations or liquidity top-ups (e.g., via `donateETH`) immediately usable to unblock withdrawals?

## 8. RPC Latency and Reliability
- [ ] Have transaction propagation delays to RPC endpoints been minimized?
- [ ] Is there redundancy and fallback mechanisms for RPC nodes?
- [ ] Are RPC nodes geographically distributed?

## 9. Decentralization
- [ ] Is there a clear roadmap to sequencer decentralization?
- [ ] Are there protections against censorship by sequencers?
- [ ] Have centralization risks been analyzed and mitigated?

## 10. EVM Compatibility
- [ ] Can contracts be deployed unchanged from Ethereum?
- [ ] Are differences from Ethereum EVM clearly documented?
- [ ] Are there unsupported or modified EVM opcodes?

## 11. Ethereum Feature Drift
- [ ] Does the rollup track latest Ethereum EIPs?
- [ ] Is there a clear policy for adopting future EIPs?
- [ ] Are compatibility breaks clearly communicated?
- [ ] Are assumptions about contract address behavior (e.g., `code.length`) updated to reflect changes like EIP-7702?

## 12. Operational Cost
- [ ] Have infrastructure operational costs been transparently communicated?
- [ ] Is the fee model clearly documented?
- [ ] Are measures in place to avoid fee manipulation?

## 13. Transaction Costs
- [ ] Has typical transaction cost been analyzed and optimized?
- [ ] Are protections against economic attacks in place (spam, gas manipulation)?
- [ ] Is gas optimization actively maintained?

## 14. Production Deployments & Real-world Experience
- [ ] Has the rollup been tested thoroughly in production environments?
- [ ] How many production-grade deployments are using this stack?
- [ ] Have insights from real deployments been integrated?

## 15. MEV and Sequencing Risks
- [ ] Are protections against sequencing manipulation (MEV) implemented?
- [ ] Is transparency for transaction ordering guaranteed?
- [ ] Are sequencers economically incentivized against manipulation?

## 16. Emergency & Recovery
- [ ] Is there an emergency exit mechanism to L1?
- [ ] Is there an emergency operational mode?
- [ ] Are catastrophic failure scenarios documented with recovery steps?
- [ ] Can emergency mechanisms be applied selectively (e.g., per chainId)?
- [ ] Can access rights (e.g., to portals or lockboxes) be revoked dynamically?

## 17. Bridges
- [ ] Are bridges trust-minimized?
- [ ] Is replay attack protection implemented?
- [ ] Have double-spending and atomicity concerns been addressed?
- [ ] Are bridge targets validated against full set of current and future contract variants?

## 18. Oracle & External Dependencies
- [ ] Are oracle failures and manipulations protected against?
- [ ] Are backup oracle solutions implemented?
- [ ] Are oracle dependencies minimized?

## 19. Denial of Service & Resilience
- [ ] Is there protection from DoS attacks?
- [ ] Are spam and flooding transaction protections in place?
- [ ] Is there graceful degradation under overload scenarios?

## 20. Miscellaneous
- [ ] Are there no hardcoded values or addresses sensitive to network changes?
- [ ] Is logging and monitoring effectively implemented?
- [ ] Is comprehensive documentation available for all roles (users/devs)?
- [ ] Are all protocol-level constants (e.g., gas limits, fee bounds, timestamps) validated on-chain where necessary?
- [ ] Are fallback mechanisms and partially migrated states handled predictably?
