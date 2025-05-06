# 🛠️ DApp Developer Security Checklist for Rollup Integration

## 1. Chain & Opcode Compatibility

- [ ] Are hardcoded chain IDs used consistently across contracts?
- [ ] Is `tx.origin` and `msg.sender` behavior consistent across all Rollups?
  - 🚨 *Issue example*: Unexpected contract permissions due to differences in `tx.origin`.
- [ ] Are all EVM opcodes used compatible across Optimism, Arbitrum, zkSync, and Ethereum mainnet?
  - 🚨 *Issue example*: Opcodes like `PUSH0` not supported universally (e.g., zkSync).

```solidity
// [✓] Are hardcoded chain IDs used consistently across contracts?
// ❗ Жестко заданный chainId приводит к сбоям при деплое в другой сети.
// Пример: контракты работают только на Ethereum Mainnet.
contract ChainRestricted {
    function restrictedAction() external view {
        require(block.chainid == 1, "Only mainnet allowed");
        // ⚠️ Контракт не будет работать на L2 (Arbitrum = 42161, Optimism = 10 и т.д.)
    }
}
```

```solidity
// [✓] Is tx.origin and msg.sender behavior consistent across all Rollups?
// ❗ tx.origin может изменяться в некоторых Rollups из-за aliasing или промежуточных прокси.
// Ошибки возникают, когда авторизация завязана на tx.origin.
contract OriginCheck {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function criticalAction() external {
        require(tx.origin == owner, "Not authorized");
        // ⚠️ Может быть эксплуатируемо через подделку tx.origin на L2
    }
}
```

```solidity
// [✓] Are all EVM opcodes used compatible across all L2s?
// ❗ Некоторые опкоды (например, PUSH0) могут не поддерживаться во всех сетях.
// zkSync Era (до обновлений) не поддерживал PUSH0, что ломает компиляцию или выполнение.
contract UnsafeOpcode {
    function usePush0() public pure returns (bytes1) {
        assembly {
            let zero
            mstore(0x0, zero) // ⚠️ PUSH0 (0x5f) может не существовать на некоторых Rollups
            return(0x0, 0x01)
        }
    }
}
```


## 2. Block.number & Block.timestamp Dependency

- [ ] Is the contract logic independent from hardcoded `block.number` or fixed intervals?
  - 🚨 *Issue example*: Optimism has ~2s blocks, Ethereum ~12s. Timings can diverge significantly.
- [ ] Are timestamp-dependent contracts tolerant of different average block times?
- [ ] Have Arbitrum-specific multi-tx block implications been considered?

```solidity
// [✓] Is the contract logic independent from hardcoded `block.number`?
// ❗ Жестко заданные интервалы по block.number могут сломаться в Rollups с другим временем блока.
contract BlockNumberLock {
    uint256 public unlockBlock;

    constructor() {
        unlockBlock = block.number + 1000;
        // ⚠️ На Optimism с блоками ~2s это ~30 мин,
        // на Ethereum с ~12s это уже 3+ часа.
    }

    function withdraw() external view returns (bool) {
        return block.number >= unlockBlock;
    }
}
```

```solidity
// [✓] Are timestamp-dependent contracts tolerant of different block production speeds?
// ❗ Использование `block.timestamp` без буферов может привести к эксплойтам или неработающему таймингу.
contract TimestampVoting {
    uint256 public votingEnd;

    function startVote(uint256 durationSeconds) external {
        votingEnd = block.timestamp + durationSeconds;
        // ⚠️ В сетях с атакуемыми sequencer'ами можно сместить тайминг.
    }

    function vote() external view returns (bool) {
        return block.timestamp <= votingEnd;
    }
}
```

```solidity
// [✓] Have Arbitrum-specific multi-tx block implications been considered?
// ❗ На Arbitrum возможно выполнение нескольких транзакций в одном L1-блоке,
// что может влиять на логику, завязанную на `block.timestamp` или `block.number`.
contract BlockSensitiveAuction {
    uint256 public lastBidBlock;

    function bid() external {
        require(block.number > lastBidBlock, "Only one bid per block");
        // ⚠️ Может быть нарушено при batch execution (multi-tx в одном блоке)
        lastBidBlock = block.number;
    }
}
```

## 3. Token Standards & ERC20 Decimals

- [ ] Is there consistency in ERC20 decimals across deployed Rollups?
  - 🚨 *Issue example*: USDT or USDC decimal discrepancies across chains.
- [ ] Does the contract logic adapt correctly to different token decimals?

```solidity
// [✓] Is there consistency in ERC20 decimals across deployed Rollups?
// ❗ Некоторые токены (например, USDT, USDC) имеют разные decimals в зависимости от сети.
// Контракты, которые ожидают фиксированные 18 decimal, могут работать некорректно.
interface IERC20Decimals {
    function decimals() external view returns (uint8);
}

contract FixedMathAssumption {
    address public token = 0x...;

    function calculateAmount(uint256 rawAmount) external view returns (uint256) {
        // ⚠️ Ошибка: предполагается, что токен всегда имеет 18 decimal
        return rawAmount * 1e18;
    }
}
```

```solidity
// [✓] Does the contract logic adapt correctly to different token decimals?
// ❗ Расчёты, привязанные к "жестко забитым" decimal, могут давать некорректные результаты на Rollups.
contract MisalignedDecimals {
    mapping(address => uint256) public userDeposits;

    function deposit(address token, uint256 amount) external {
        require(amount >= 1e18, "Minimum deposit is 1 token");
        // ⚠️ Проблема: если токен имеет 6 decimal (например, USDC), 1e18 — это 1,000,000 токенов.
        userDeposits[msg.sender] += amount;
    }
}
```

## 4. Cross-chain Messaging & Bridges

- [ ] Have messaging implementations (bridges) been reviewed thoroughly?
- [ ] Are replay protection mechanisms (e.g., chain-specific nonce) in place?
  - 🚨 *Issue example*: Replay attacks using signed messages across Rollups.
- [ ] Is there a whitelist of verified and compatible chains?

```solidity
// [✓] Have messaging implementations been reviewed thoroughly?
// ❗ Часто используется кастомная логика мостов без аудита — особенно в малых Rollups.
// Внедрение неподтверждённых бриджей может привести к фальшивым вызовам или двойному исполнению.
contract UnsafeBridgeReceiver {
    event MessageReceived(address sender, bytes data);

    function onMessage(address from, bytes calldata data) external {
        // ⚠️ Нет проверки источника, авторизации, идентификатора сети — может вызвать внешнее исполнение
        emit MessageReceived(from, data);
    }
}
```

```solidity
// [✓] Are replay protection mechanisms in place?
// ❗ Повторное воспроизведение одного и того же сообщения на другой сети без nonce защиты.
contract ReplayVulnerableBridge {
    mapping(bytes32 => bool) public processed;

    function handleMessage(bytes calldata payload, uint256 chainId) external {
        // ⚠️ Нет уникального nonce или context per chain — возможно воспроизведение с другой сети
        bytes32 hash = keccak256(abi.encode(payload));
        require(!processed[hash], "Already processed");
        processed[hash] = true;
        // process...
    }
}
```

```solidity
// [✓] Is there a whitelist of verified and compatible chains?
// ❗ Без whitelisting можно получить сообщения с неподдерживаемых или поддельных цепочек.
contract OpenBridgeHandler {
    function receiveMessage(uint256 sourceChainId, address sender, bytes calldata data) external {
        // ⚠️ Нет проверки, что sourceChainId — известная сеть
        // Атакующий может подделать значение и вызвать непредусмотренное поведение
    }
}
```

## 5. Upgradeability Across Chains

- [ ] Is contract upgradeability consistent across deployed chains?
- [ ] Have upgradeability governance processes been checked across all Rollups?
  - 🚨 *Issue example*: Different upgrade permissions causing divergence in state or behavior.

```solidity
// [✓] Is contract upgradeability consistent across deployed chains?
// ❗ Часто Rollup-инстансы одного и того же контракта используют разные proxy-логики или upgraders.
// Это ведёт к рассинхронизации состояния и нарушению инвариантов.
contract Proxy {
    address public implementation;
    address public admin;

    function upgrade(address newImpl) external {
        require(msg.sender == admin, "Not authorized");
        implementation = newImpl;
    }

    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

// ⚠️ Проблема: на одном Rollup может быть `admin = Multisig`, на другом — `EOA` или `Governance`.
// В результате — расхождение логики, несанкционированные апгрейды или недоступность фич.
```

```solidity
// [✓] Have upgradeability governance processes been checked across all Rollups?
// ❗ Отсутствие синхронизированного governance flow может привести к split-state атакам.
contract UpgradeControl {
    address public governor;

    function upgradeTo(address newImpl) external {
        require(msg.sender == governor, "Not governor");
        // ⚠️ Если governor разный на каждой сети — можно внедрить несовместимые версии
    }
}
```

## 6. Signature Replay Vulnerabilities

- [ ] Are signatures strictly validated with chain-specific data included?
  - 🚨 *Issue example*: An attacker can reuse signatures intended for another chain.
- [ ] Is `chain_id` always explicitly verified in signed payloads?

```solidity
// [✓] Are signatures strictly validated with chain-specific data included?
// ❗ Подписи без chain-specific данных можно переиграть на других Rollups, если адреса совпадают.
contract SignatureReplay {
    mapping(bytes32 => bool) public used;

    function verifyAndExecute(
        bytes32 message,
        bytes calldata signature
    ) external {
        require(!used[message], "Already used");
        address signer = recoverSigner(message, signature);
        // ⚠️ message может быть одинаковым на других сетях — replay-вектор
        used[message] = true;
        // proceed...
    }

    function recoverSigner(bytes32 msgHash, bytes memory sig) public pure returns (address) {
        // ecrecover logic...
    }
}
```

```solidity
// [✓] Is chain_id always explicitly verified in signed payloads?
// ❗ Подписанные данные должны включать chainId, иначе подпись может быть действительна на другой сети.
contract UnsafePermit {
    function getMessageHash(
        address user,
        uint256 amount,
        uint256 nonce
    ) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(user, amount, nonce));
        // ⚠️ Нет chainId — одинаковые подписи валидны на всех совместимых Rollups
    }
}
```


## 7. Low Gas Fee Exploits & DoS

- [ ] Is there protection against mass transactions exploiting low fees on certain Rollups?
  - 🚨 *Issue example*: Spam attacks economically feasible due to cheaper fees on zkSync or Optimism.
- [ ] Are contracts resilient to transaction spam causing DoS?

```solidity
// [✓] Is there protection against mass transactions exploiting low fees?
// ❗ При низких gas-стоимостях атака "1000 tx за 1$" становится реальной. Без защиты контракт можно засорить.
contract NoAntiSpam {
    mapping(address => uint256) public actions;

    function trigger() external {
        actions[msg.sender]++;
        // ⚠️ Отсутствие rate-limit или минимального fee позволяет бесконечно вызывать trigger на L2
    }
}
```

```solidity
// [✓] Are contracts resilient to transaction spam causing DoS?
// ❗ Контракты с дорогостоящими записями (storage writes) могут быть заблокированы DoS-спамом на дешёвых Rollups.
contract ExpensiveWrite {
    mapping(uint256 => string) public records;

    function register(uint256 id, string calldata data) external {
        records[id] = data;
        // ⚠️ Можно создать сотни записей за копейки → переполнить storage / задушить другие функции
    }
}
```

## 8. Hardcoded Addresses & Parameters

- [ ] Have all hardcoded addresses (e.g., routers like WETH) been verified per chain?
  - 🚨 *Issue example*: WETH router address varies significantly by Rollup.
- [ ] Are critical parameters dynamic and adjustable per deployment?

```solidity
// [✓] Have all hardcoded addresses been verified per chain?
// ❗ WETH адреса, DEX-роутеры, системные контракты — все отличаются между Rollups.
// Жестко заданные адреса приводят к фатальным ошибкам при деплое на другую сеть.
contract HardcodedRouter {
    address public constant UNISWAP_ROUTER = 0xE592...; // Mainnet router

    function swapExactETHForTokens() external payable {
        // ⚠️ На Arbitrum/Optimism этот адрес не существует или ведёт к другой логике
        (bool success, ) = UNISWAP_ROUTER.call{value: msg.value}("");
        require(success, "Router call failed");
    }
}
```

```solidity
// [✓] Are critical parameters dynamic and adjustable per deployment?
// ❗ Если комиссии, лимиты, мультисиги и др. заданы в коде — их невозможно адаптировать под каждую сеть.
contract StaticConfig {
    uint256 public constant MAX_DEPOSIT = 1000 ether;

    function deposit(uint256 amount) external {
        require(amount <= MAX_DEPOSIT, "Too much");
        // ⚠️ Может быть неприменимо в Rollup с другими лимитами/механизмами
    }
}
```

## 9. AMM Pools token0/token1 Order

- [ ] Is token ordering logic (Uniswap-style `token0/token1`) stable across chains?
  - 🚨 *Issue example*: Different address ordering causing AMM pairs mismatch between chains.

```solidity
// [✓] Is token ordering logic stable across chains?
// ❗ Многие AMM (Uniswap, Sushi) используют сортировку token0 < token1 по address.
// Однако на разных сетях контракты могут быть развёрнуты в другом порядке → нарушение предсказуемости пар.
contract PairCreator {
    function getPairKey(address tokenA, address tokenB) public pure returns (bytes32) {
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        return keccak256(abi.encodePacked(token0, token1));
        // ⚠️ Если токены деплоятся в другом порядке на другой сети — другая пара, другой адрес, другой пул
    }
}
```

## 10. RPC Latency & State Consistency

- [ ] Does frontend/RPC integration correctly handle varied RPC response latencies?
- [ ] Are state updates verified consistently across RPC endpoints?

```solidity
// [✓] Are state updates verified consistently across RPC endpoints?
// ❗ Rollups могут иметь out-of-sync RPC-ноды. Без верификации данных с нескольких источников — риск ошибок.
contract RpcStateReader {
    function readBalance(address user) public view returns (uint256) {
        // ⚠️ Если фронтенд запрашивает balance через RPC, который отстаёт от L1 sync,
// пользователь может видеть устаревшее состояние и принимать неверные действия.
        return user.balance;
    }
}
```

## 11. L2-specific MEV Considerations

- [ ] Has the DApp accounted for MEV risk unique to Optimistic and ZK-rollups?
  - 🚨 *Issue example*: Sequencer front-running transactions on Optimism due to centralized sequencing.
- [ ] Are protections (such as private mempool integrations or encrypted mempool) considered?

```solidity
// [✓] Has the DApp accounted for MEV risk unique to Rollups?
// ❗ Централизованный Sequencer на Optimism или Base может переставить транзакции в выгодном порядке.
contract Auction {
    uint256 public highestBid;

    function placeBid(uint256 bid) external {
        require(bid > highestBid, "Bid too low");
        highestBid = bid;
        // ⚠️ MEV-атака: Sequencer видит вашу транзакцию и вставляет свою перед вашей с чуть большей ставкой.
    }
}
```


## 12. L2 Sequencer Uptime & Reliability (e.g., Chainlink feeds)

- [ ] Do contracts verify sequencer uptime/availability?
  - 🚨 *Issue example*: Funds frozen if sequencer goes down without fallback mechanisms.
- [ ] Is a fallback mechanism in place if Chainlink sequencer uptime feed is unavailable?

```solidity
// [✓] Do contracts verify sequencer uptime?
// ❗ При отсутствии доступа к Chainlink Sequencer Feed — нельзя оценить доступность L2, рискуя заморозить средства.
interface AggregatorV3Interface {
    function latestRoundData() external view returns (
        uint80, int256, uint256, uint256 updatedAt, uint80
    );
}

contract TimeSensitiveAction {
    AggregatorV3Interface public sequencerUptimeFeed;

    function performAction() external {
        (, int256 status,, uint256 updatedAt,) = sequencerUptimeFeed.latestRoundData();
        require(status == 0, "Sequencer down");
        // ⚠️ Если фид недоступен или не обновляется — действие становится навсегда недоступным
        // ⚠️ Нет fallback-логики
    }
}
```

## 13. Ethereum Feature Drift & Solidity Versions

- [ ] Is Solidity compiler version consistent and supported across all Rollups?
  - 🚨 *Issue example*: zkSync may lag behind mainline Solidity version support.

```solidity
// [✓] Is Solidity version supported across Rollups?
// ❗ Некоторые Rollups отстают от последних версий Solidity.
// Например, zkSync Era не сразу поддерживает ^0.8.20+ (PUSH0, custom errors, и т.д.)

// ⚠️ Контракт может не компилироваться на других Rollups без модификаций
pragma solidity ^0.8.21;

contract FeatureDrift {
    error Unauthorized();

    function fail() external pure {
        revert Unauthorized(); // ❗ Может не поддерживаться на всех L2 без соответствующей версии компилятора
    }
}
```
## 14. Production Deployments & Testing

- [ ] Has the contract been tested comprehensively in actual rollup environments?
- [ ] Are real-world edge cases from existing multichain deployments integrated into testing?

## 15. Documentation & User Guidance

- [ ] Are rollup-specific behaviors and risks clearly documented for end-users?
- [ ] Have deployment instructions and best practices been specified per rollup?

## 16. Miscellaneous & Additional Checks

- [ ] Is the contract resilient against sudden changes or hard forks in underlying L2 chains?
- [ ] Has contract logic been audited specifically with a focus on rollup differences?
