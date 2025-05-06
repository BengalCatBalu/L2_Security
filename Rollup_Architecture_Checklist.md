# Rollup (Layer 2) Architect Security Checklist

## 1. State Validation & Proofs

- [ ] Is the state posted to L1 always valid and verifiable?
- [ ] Can invalid states be challenged and reverted efficiently?
- [ ] Are Fraud/Validity proofs robust and cover all fraud scenarios?
- [ ] Are anchor roots based on recent enough L2 blocks (e.g., no older than 6 months L1 timestamp)?
- [ ] Are anchor roots only accepted from dispute games created within the correct registry context?
- [ ] Are all input invariants (timestamps, chainId ordering, block numbers, gas limits) explicitly validated?

```solidity
// Is the state posted to L1 always valid and verifiable?
// Example: Publishing L2 state root to L1
contract L1StatePublisher {
    event StatePublished(bytes32 indexed outputRoot, uint256 l2Block, uint256 timestamp);

    function publishState(bytes32 outputRoot, uint256 l2Block) external {
        emit StatePublished(outputRoot, l2Block, block.timestamp);
    }
}

// Can invalid states be challenged and reverted efficiently?
// Example: Dispute resolution mechanism
contract DisputeGame {
    address public challenger;
    bytes32 public disputedState;

    function challenge(bytes32 _disputedState) external {
        disputedState = _disputedState;
        challenger = msg.sender;
    }

    function resolveChallenge(bool fraudDetected) external {
        require(msg.sender == address(0xValidator));
        if (fraudDetected) {
            emit StateReverted(disputedState);
        }
    }

    event StateReverted(bytes32 invalidRoot);
}

// Are Fraud/Validity proofs robust and cover all fraud scenarios?
// Example: ZK proof verification
contract ZKVerifier {
    function verifyProof(
        uint256[8] calldata proof,
        uint256[1] calldata publicSignals
    ) external pure returns (bool) {
        return Groth16.verify(proof, publicSignals);
    }
}

// Are anchor roots based on recent enough L2 blocks (e.g., no older than 6 months L1 timestamp)?
// Example: Recency check for anchor roots
function isFreshAnchor(uint256 l1Timestamp, uint256 anchorTimestamp) public pure returns (bool) {
    return l1Timestamp - anchorTimestamp <= 180 days;
}

// Are anchor roots only accepted from dispute games created within the correct registry context?
// Example: Registry-based validation of anchor roots
mapping(address => bool) public isValidRegistry;

function acceptAnchor(address registry, bytes32 anchor) external {
    require(isValidRegistry[registry], "Invalid registry source");
}

// Are all input invariants (timestamps, chainId ordering, block numbers, gas limits) explicitly validated?
// Example: Input invariant checks
function validateInputs(
    uint256 timestamp,
    uint256 blockNumber,
    uint256 chainId,
    uint256 gasLimit
) public pure {
    require(timestamp > 0, "Invalid timestamp");
    require(blockNumber > 0, "Invalid block number");
    require(chainId == 1 || chainId == 10, "Unsupported chainId");
    require(gasLimit <= 30_000_000, "Gas limit too high");
}
```


## 2. Data Availability

- [ ] Are transaction data available independently from the sequencer?
- [ ] Can users reconstruct rollup state purely from data on L1?
- [ ] Are fallback data-availability mechanisms in place?

```solidity
// Are transaction data available independently from the sequencer?
// Example: Rollup publishes calldata to L1 for independent access
contract RollupTxPublisher {
    event L2Transaction(bytes data);

    function publishTx(bytes calldata txData) external {
        emit L2Transaction(txData); // Stored on-chain in L1 event logs
    }
}

// Can users reconstruct rollup state purely from data on L1?
// Example: Mapping L2 state from on-chain logs or calldata
contract StateCommitmentChain {
    bytes32[] public stateRoots;

    function appendState(bytes32 stateRoot) external {
        stateRoots.push(stateRoot);
    }

    function getState(uint256 index) external view returns (bytes32) {
        return stateRoots[index];
    }
}

// Are fallback data-availability mechanisms in place?
// Example: Check if alternative DA source (e.g., IPFS / Celestia) is configured
contract FallbackDataAvailability {
    mapping(bytes32 => string) public ipfsBackups;

    function registerBackup(bytes32 batchHash, string memory ipfsCid) public {
        ipfsBackups[batchHash] = ipfsCid;
    }

    function getBackup(bytes32 batchHash) public view returns (string memory) {
        return ipfsBackups[batchHash];
    }
}
```

## 3. Sequencer Failure

- [ ] Is there a fallback mechanism if the primary sequencer fails?
- [ ] Can users bypass the sequencer and submit directly to L1?
- [ ] Is auto-switching between sequencers clearly defined?

```solidity
// Is there a fallback mechanism if the primary sequencer fails?
// Example: Manual activation of backup sequencer
contract SequencerManager {
    address public primarySequencer;
    address public backupSequencer;
    bool public sequencerActive;

    function activateBackup() external {
        require(msg.sender == backupSequencer, "Not authorized");
        sequencerActive = false;
        primarySequencer = backupSequencer;
    }
}

// Can users bypass the sequencer and submit directly to L1?
// Example: Direct L1 submission in case of censorship
contract L1UserSubmission {
    event ForcedTx(bytes data);

    function forceSubmit(bytes calldata txData) external {
        emit ForcedTx(txData); // Can be picked up by L2 verifier directly from L1
    }
}

// Is auto-switching between sequencers clearly defined?
// Example: Time-based sequencer rotation logic
contract SequencerRotation {
    address[] public sequencerQueue;
    uint256 public rotationPeriod;
    uint256 public lastRotation;

    function rotateSequencer() public {
        require(block.timestamp >= lastRotation + rotationPeriod, "Too early");
        address next = sequencerQueue[0];
        // rotate logic...
        lastRotation = block.timestamp;
    }
}
```


## 4. Proposer Failure

- [ ] Does a backup proposer mechanism exist if the main proposer stops?
- [ ] Is there handling of prolonged proposer inactivity?
- [ ] Is the proposer role clearly separated from sequencer role?

```solidity
// Does a backup proposer mechanism exist if the main proposer stops?
// Example: Backup proposer can take over if main proposer is inactive
contract ProposerManager {
    address public currentProposer;
    address public backupProposer;

    function triggerFallback() external {
        require(msg.sender == backupProposer, "Not authorized");
        currentProposer = backupProposer;
    }
}

// Is there handling of prolonged proposer inactivity?
// Example: Timeout mechanism that deactivates inactive proposer
contract ProposerTimeout {
    mapping(address => uint256) public lastProposalTime;
    uint256 public timeoutThreshold = 1 days;

    function isProposerActive(address proposer) public view returns (bool) {
        return block.timestamp - lastProposalTime[proposer] <= timeoutThreshold;
    }

    function recordProposal(address proposer) external {
        lastProposalTime[proposer] = block.timestamp;
    }
}

// Is the proposer role clearly separated from sequencer role?
// Example: Role separation enforcement
contract RoleManager {
    address public sequencer;
    address public proposer;

    modifier onlySequencer() {
        require(msg.sender == sequencer, "Not sequencer");
        _;
    }

    modifier onlyProposer() {
        require(msg.sender == proposer, "Not proposer");
        _;
    }
}
```


## 5. Exit Window and Upgradeability

- [ ] Is there a sufficient delay (exit window) for users before upgrades are activated?
- [ ] Can users securely withdraw assets if disagreeing with an upgrade?
- [ ] Is governance process secure against malicious upgrades?
- [ ] Are upgrade procedures safe even in case of human error (e.g., misuse of `upgradeTo` without `call`)?
- [ ] Are input parameters strictly validated during deployments (e.g., consistency of game types)?
- [ ] Is the system upgrade-safe regarding withdrawal finality and anchor state migration?

```solidity
// Is there a sufficient delay (exit window) for users before upgrades are activated?
// Example: Upgrade scheduling with enforced delay
contract UpgradeController {
    uint256 public constant UPGRADE_DELAY = 3 days;
    uint256 public scheduledUpgradeTime;
    address public pendingImplementation;

    function scheduleUpgrade(address newImpl) external {
        scheduledUpgradeTime = block.timestamp + UPGRADE_DELAY;
        pendingImplementation = newImpl;
    }

    function activateUpgrade() external {
        require(block.timestamp >= scheduledUpgradeTime, "Upgrade delay not met");
        // upgrade logic here
    }
}

// Are upgrade procedures safe even in case of human error (e.g., misuse of `upgradeTo` without `call`)?
// Example: Enforcing safe upgrade call pattern
interface IUpgradeable {
    function upgradeTo(address newImpl) external;
}

contract SafeUpgradeProxy {
    function upgrade(address proxy, address newImpl) external {
        IUpgradeable(proxy).upgradeTo(newImpl);
        (bool success, ) = proxy.call(abi.encodeWithSignature("postUpgradeCheck()"));
        require(success, "Post-upgrade call failed");
    }
}

// Are input parameters strictly validated during deployments?
// Example: Validating game type consistency at initialization
contract DeploymentValidator {
    enum GameType { FraudProof, ValidityProof }

    function validateGameType(GameType provided, GameType expected) internal pure {
        require(provided == expected, "Mismatched game type");
    }
}
```


## 6. Transaction Finality & Chain Re-org

- [ ] Are transaction finality levels (pre-finality, soft, hard) clearly defined?
- [ ] Is rollup resistant to Ethereum re-org events?
- [ ] Are users clearly informed about transaction finality timing?

```solidity
// Is rollup resistant to Ethereum re-org events?
// Example: Delayed finalization of L1-published output to account for re-org risk
contract FinalityManager {
    struct Output {
        bytes32 root;
        uint256 timestamp;
        bool finalized;
    }

    Output[] public outputs;
    uint256 public constant FINALITY_DELAY = 12; // e.g., 12 L1 blocks

    function publishOutput(bytes32 root) external {
        outputs.push(Output({
            root: root,
            timestamp: block.number,
            finalized: false
        }));
    }

    function finalize(uint256 index) external {
        require(block.number >= outputs[index].timestamp + FINALITY_DELAY, "Not final yet");
        outputs[index].finalized = true;
    }
}

// Are users clearly informed about transaction finality timing?
// Example: Exposing finality status via view function
contract FinalityStatus {
    mapping(bytes32 => uint256) public txFinality; // timestamp or block number of finality

    function getFinalityStatus(bytes32 txHash) external view returns (string memory) {
        if (txFinality[txHash] == 0) return "pending";
        if (block.number - txFinality[txHash] < 6) return "soft finality";
        return "hard finality";
    }
}
```
## 7. L1 Withdrawal Latency

- [ ] How long does it take to withdraw assets to L1?
- [ ] Is withdrawal protected against griefing and censorship attacks?
- [ ] Is there an emergency withdrawal procedure?
- [ ] Are SuperRoot-based withdrawals correctly validated for timestamps and chainId ordering?
- [ ] Are donations or liquidity top-ups (e.g., via `donateETH`) immediately usable to unblock withdrawals?

```solidity
// Is there an emergency withdrawal procedure?
// Example: Emergency mode allowing immediate exits
contract EmergencyExit {
    address public admin;
    bool public emergencyMode;

    constructor() {
        admin = msg.sender;
    }

    function activateEmergency() external {
        require(msg.sender == admin, "Unauthorized");
        emergencyMode = true;
    }

    function emergencyWithdraw(address token, address to, uint256 amount) external {
        require(emergencyMode, "Not in emergency mode");
        IERC20(token).transfer(to, amount);
    }
}

// Are SuperRoot-based withdrawals correctly validated for timestamps and chainId ordering?
// Example: Validating withdrawal proofs against expected parameters
contract SuperRootValidator {
    struct WithdrawalProof {
        bytes32 root;
        uint256 timestamp;
        uint256 chainId;
    }

    function validateProof(WithdrawalProof memory proof) public view {
        require(proof.timestamp <= block.timestamp, "Invalid timestamp");
        require(proof.chainId == block.chainid, "Invalid chainId");
        // add more validation logic
    }
}

// Are donations or liquidity top-ups immediately usable to unblock withdrawals?
// Example: Allowing top-ups to cover pending user exits
contract LiquidityPool {
    mapping(address => uint256) public pendingWithdrawals;

    function donateETH() external payable {
        // ETH donated becomes immediately usable to fulfill pending exits
    }

    function processWithdrawal(address user, uint256 amount) external {
        require(address(this).balance >= amount, "Insufficient liquidity");
        payable(user).transfer(amount);
        pendingWithdrawals[user] = 0;
    }
}
```


## 8. RPC Latency and Reliability

- [ ] Have transaction propagation delays to RPC endpoints been minimized?
- [ ] Is there redundancy and fallback mechanisms for RPC nodes?
- [ ] Are RPC nodes geographically distributed?

```solidity
// Is there redundancy and fallback mechanisms for RPC nodes?
// Example: Client-side logic (off-chain) would handle this,
// but smart contracts may emit endpoints or versions for client discovery

contract RpcRegistry {
    string[] public endpoints;

    function registerRpc(string memory url) external {
        endpoints.push(url);
    }

    function getRpc(uint256 index) external view returns (string memory) {
        return endpoints[index];
    }
}
```

---

## 9. Decentralization

- [ ] Is there a clear roadmap to sequencer decentralization?
- [ ] Are there protections against censorship by sequencers?
- [ ] Have centralization risks been analyzed and mitigated?

```solidity
// Are there protections against censorship by sequencers?
// Example: Allow users to bypass sequencer and force tx inclusion
contract CensorshipBypass {
    event ForcedTransaction(bytes data);

    function submitForcedTx(bytes calldata txData) external {
        emit ForcedTransaction(txData);
    }
}
```


## 10. EVM Compatibility

- [ ] Can contracts be deployed unchanged from Ethereum?
- [ ] Are differences from Ethereum EVM clearly documented?
- [ ] Are there unsupported or modified EVM opcodes?

```solidity
// Are there unsupported or modified EVM opcodes?
// Example: L2 opcode shadowing detection via test contract
contract OpcodeTest {
    function testSelfBalance() public view returns (uint256) {
        return address(this).balance;
    }

    function testCreate2(bytes32 salt) public returns (address) {
        return address(new Contract{salt: salt}());
    }
}
```

---

## 11. Ethereum Feature Drift

- [ ] Does the rollup track latest Ethereum EIPs?
- [ ] Is there a clear policy for adopting future EIPs?
- [ ] Are compatibility breaks clearly communicated?
- [ ] Are assumptions about contract address behavior (e.g., `code.length`) updated to reflect changes like EIP-7702?

```solidity
// Are assumptions about contract address behavior updated (e.g. EIP-7702)?
// Example: Detecting code presence via `code.length`
contract AddressBehavior {
    function isContract(address addr) public view returns (bool) {
        return addr.code.length > 0;
    }
}
```

---

## 12. Operational Cost

- [ ] Have infrastructure operational costs been transparently communicated?
- [ ] Is the fee model clearly documented?
- [ ] Are measures in place to avoid fee manipulation?

```solidity
// Are measures in place to avoid fee manipulation?
// Example: Setting bounds on fee update rates
contract FeeController {
    uint256 public baseFee;
    uint256 public lastUpdate;

    function updateFee(uint256 newFee) external {
        require(block.timestamp > lastUpdate + 1 hours, "Update too frequent");
        require(newFee <= baseFee * 2, "Fee spike too high");
        baseFee = newFee;
        lastUpdate = block.timestamp;
    }
}
```


## 13. Transaction Costs

- [ ] Has typical transaction cost been analyzed and optimized?
- [ ] Are protections against economic attacks in place (spam, gas manipulation)?
- [ ] Is gas optimization actively maintained?

```solidity
// Are protections against economic attacks in place?
// Example: Anti-spam minimum fee threshold
contract AntiSpam {
    uint256 public minTxFee = 1e14; // 0.0001 ETH

    function submitTx() external payable {
        require(msg.value >= minTxFee, "Fee too low, possible spam");
        // process tx...
    }
}
```

---

## 14. Production Deployments & Real-world Experience

- [ ] Has the rollup been tested thoroughly in production environments?
- [ ] How many production-grade deployments are using this stack?
- [ ] Have insights from real deployments been integrated?

*(No contract code — assessed through project documentation and audits.)*

---

## 15. MEV and Sequencing Risks

- [ ] Are protections against sequencing manipulation (MEV) implemented?
- [ ] Is transparency for transaction ordering guaranteed?
- [ ] Are sequencers economically incentivized against manipulation?

```solidity
// Are protections against sequencing manipulation (MEV) implemented?
// Example: Commitment to ordering before execution (pre-image commitments)
contract OrderedTxPool {
    struct TxCommitment {
        bytes32 txHash;
        uint256 timestamp;
    }

    TxCommitment[] public committedTxs;

    function commitTx(bytes32 txHash) external {
        committedTxs.push(TxCommitment({
            txHash: txHash,
            timestamp: block.timestamp
        }));
    }

    // Execution happens after fixed delay to guarantee ordering
}
```


## 16. Emergency & Recovery

- [ ] Is there an emergency exit mechanism to L1?
- [ ] Is there an emergency operational mode?
- [ ] Are catastrophic failure scenarios documented with recovery steps?
- [ ] Can emergency mechanisms be applied selectively (e.g., per chainId)?
- [ ] Can access rights (e.g., to portals or lockboxes) be revoked dynamically?

```solidity
// Is there an emergency exit mechanism to L1?
// Example: Triggerable emergency exit path
contract EmergencyBridgeExit {
    bool public emergency;
    address public admin;

    constructor() {
        admin = msg.sender;
    }

    function activateEmergency() external {
        require(msg.sender == admin, "Unauthorized");
        emergency = true;
    }

    function emergencyWithdraw(address user, uint256 amount) external {
        require(emergency, "Not emergency mode");
        payable(user).transfer(amount);
    }
}

// Can access rights (e.g., to portals or lockboxes) be revoked dynamically?
contract DynamicAccess {
    mapping(address => bool) public isAuthorized;

    function revoke(address user) external {
        isAuthorized[user] = false;
    }
}
```

---

## 17. Bridges

- [ ] Are bridges trust-minimized?
- [ ] Is replay attack protection implemented?
- [ ] Have double-spending and atomicity concerns been addressed?
- [ ] Are bridge targets validated against full set of current and future contract variants?

```solidity
// Is replay attack protection implemented?
// Example: Using nonce per user per chain
contract ReplayGuardedBridge {
    mapping(address => mapping(uint256 => bool)) public usedNonces;

    function withdraw(uint256 nonce) external {
        require(!usedNonces[msg.sender][nonce], "Replay detected");
        usedNonces[msg.sender][nonce] = true;
        // proceed with withdrawal
    }
}
```

---

## 18. Oracle & External Dependencies

- [ ] Are oracle failures and manipulations protected against?
- [ ] Are backup oracle solutions implemented?
- [ ] Are oracle dependencies minimized?

```solidity
// Are backup oracle solutions implemented?
// Example: Fallback oracle logic
contract OracleFallback {
    address public primaryOracle;
    address public backupOracle;

    function getPrice() public view returns (uint256) {
        (bool success, bytes memory data) = primaryOracle.staticcall(
            abi.encodeWithSignature("latestPrice()")
        );
        if (success) return abi.decode(data, (uint256));

        (success, data) = backupOracle.staticcall(
            abi.encodeWithSignature("latestPrice()")
        );
        require(success, "Both oracles failed");
        return abi.decode(data, (uint256));
    }
}
```

---

## 19. Denial of Service & Resilience

- [ ] Is there protection from DoS attacks?
- [ ] Are spam and flooding transaction protections in place?
- [ ] Is there graceful degradation under overload scenarios?

```solidity
// Is there protection from DoS attacks?
// Example: Rate-limiting per user
contract DoSGuard {
    mapping(address => uint256) public lastCall;

    function protectedAction() external {
        require(block.timestamp - lastCall[msg.sender] > 10, "Rate limited");
        lastCall[msg.sender] = block.timestamp;
        // protected logic
    }
}
```

---

## 20. Miscellaneous

- [ ] Are there no hardcoded values or addresses sensitive to network changes?
- [ ] Is logging and monitoring effectively implemented?
- [ ] Is comprehensive documentation available for all roles (users/devs)?
- [ ] Are all protocol-level constants (e.g., gas limits, fee bounds, timestamps) validated on-chain where necessary?
- [ ] Are fallback mechanisms and partially migrated states handled predictably?

```solidity
// Are protocol-level constants validated on-chain?
// Example: Validating bounds for key parameters
contract ConstantBounds {
    uint256 public maxGasLimit;

    function setGasLimit(uint256 newLimit) external {
        require(newLimit <= 30_000_000, "Exceeds protocol max");
        maxGasLimit = newLimit;
    }
}
```
