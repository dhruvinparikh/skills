# Solidity & EVM Development + Security Instructions for GitHub Copilot

> Comprehensive AI-assisted development guidelines combining Solidity/EVM development patterns with Trail of Bits security analysis techniques. Based on [awesome-solidity-skills](https://github.com/gonzaloetjo/awesome-solidity-skills) and [Trail of Bits Skills](https://github.com/trailofbits/skills).

---

## 📋 Table of Contents

- [Security & Auditing](#security--auditing)
- [Trail of Bits Security Skills](#trail-of-bits-security-skills)
- [Development & Architecture](#development--architecture)
- [Testing Strategies](#testing-strategies)
- [DeFi & Token Standards](#defi--token-standards)
- [Deployment & Verification](#deployment--verification)
- [Code Review & Analysis](#code-review--analysis)
- [Code Style & Best Practices](#code-style--best-practices)

---

## 🔒 Security & Auditing

### Core Security Principles

When writing or auditing Solidity contracts, always apply these checks:

#### 1. **CEI Pattern (Checks-Effects-Interactions)**
The most critical pattern for preventing reentrancy:

```solidity
function withdraw(uint256 amount) external nonReentrant {
    // 1. CHECKS
    if (balances[msg.sender] < amount) revert InsufficientBalance();
    
    // 2. EFFECTS - Update state BEFORE external calls
    balances[msg.sender] -= amount;
    
    // 3. INTERACTIONS - External calls last
    (bool success,) = msg.sender.call{value: amount}("");
    if (!success) revert TransferFailed();
}
```

#### 2. **Common Vulnerabilities to Check**

| Vulnerability | Description | Prevention |
|--------------|-------------|------------|
| **Reentrancy** | Attacker calls back during external call | Use CEI pattern, ReentrancyGuard |
| **Integer Overflow** | Pre-0.8.0 arithmetic issues | Use Solidity 0.8+ or SafeMath |
| **Access Control** | Unauthorized function access | Implement onlyOwner, role-based access |
| **Front-Running** | Transaction ordering manipulation | Use commit-reveal, private mempools |
| **Oracle Manipulation** | Price feed attacks | Multi-oracle setups, TWAP, staleness checks |
| **Unchecked Call Returns** | Ignoring call failures | Always check return values |
| **Delegatecall Issues** | Storage collision | Careful with proxy patterns |
| **Timestamp Dependence** | Miner manipulation | Don't rely on block.timestamp for critical logic |
| **Insecure Defaults** | Fail-open configurations | Fail-safe, explicit initialization |
| **Hardcoded Credentials** | Secrets in code | Use env vars, key management |

#### 3. **Security Audit Checklist**

Before deployment, verify:

- [ ] All state-changing functions follow CEI pattern
- [ ] Access control on admin/privileged functions
- [ ] Input validation (zero address, bounds checking, overflow)
- [ ] Reentrancy guards on external calls
- [ ] Events emitted for all state changes
- [ ] Custom errors used (gas efficient)
- [ ] No use of `tx.origin` for authentication
- [ ] Safe external call patterns (`call` over `transfer`/`send`)
- [ ] Proper use of `delegatecall` with storage awareness
- [ ] Oracle price staleness and manipulation checks
- [ ] Flash loan attack vectors considered
- [ ] Upgrade mechanism security (if using proxies)
- [ ] No insecure default configurations
- [ ] No hardcoded credentials or secrets
- [ ] State-changing entry points identified and documented

#### 4. **Static Analysis Integration**

Integrate tools for automated vulnerability detection:

```bash
# Slither - Static analysis
slither . --detect all

# Mythril - Symbolic execution
myth analyze contracts/MyContract.sol

# Echidna - Fuzz testing
echidna-test . --contract MyContract

# Semgrep - Custom pattern detection
semgrep --config=p/solidity contracts/
```

#### 5. **Audit Documentation Pattern**

When documenting security considerations:

```solidity
/// @notice Withdraws tokens from the vault
/// @dev SECURITY: Follows CEI pattern to prevent reentrancy
/// @dev Access: Public, but restricted to user's own balance
/// @dev External Calls: Uses low-level call with success check
/// @param amount The amount to withdraw in wei
/// @custom:security-consideration Flash loan attacks prevented by balance check
/// @custom:invariant User balance never negative
/// @custom:invariant Total balances = contract balance
function withdraw(uint256 amount) external nonReentrant {
    // Implementation
}
```

---

## 🛡️ Trail of Bits Security Skills

### Smart Contract Security Analysis

#### Building Secure Contracts Framework

When analyzing smart contracts for security:

**Entry Point Analysis:**
- Identify all state-changing entry points
- Map external/public functions that modify storage
- Track cross-contract calls and callbacks
- Document access control for each entry point

**State Invariants:**
```solidity
// Document invariants that should ALWAYS hold
/// @custom:invariant totalSupply == sum of all balances
/// @custom:invariant owner cannot be zero address
/// @custom:invariant pause can only decrease privileges, never increase
```

#### Audit Context Building

**Phase 1: Initial Orientation**
```
1. Map all contracts in the system
2. Identify entry points (external/public functions)
3. List all actors (users, admins, operators)
4. Document storage layout
5. Identify external dependencies
```

**Phase 2: Ultra-Granular Function Analysis**
For each function, analyze line-by-line:

```solidity
function complexLogic(uint256 input) external {
    // Line 1: What does this check?
    require(input > 0, "Invalid input");  
    // WHY: Prevents zero input, but allows MAX_UINT256
    // ASSUMPTION: Caller validates upper bounds
    // RISK: No upper bound check could cause overflow in calculations
    
    // Line 2: State change
    counter += input;
    // EFFECT: Increases counter
    // INVARIANT: counter should never exceed totalSupply
    // RISK: Not checked! Could break invariant
    
    // Line 3: External call
    token.transfer(msg.sender, input);
    // INTERACTION: External call AFTER state change (good - CEI)
    // RISK: What if transfer fails? No check of return value
}
```

**Phase 3: Global System Understanding**
- Reconstruct state machine
- Map all workflows end-to-end
- Identify trust boundaries
- Document upgrade mechanisms
- Verify invariants across system

**Anti-Hallucination Rules:**
- Never assume; always verify in code
- Update mental model when contradicted
- Use "Unclear; need to inspect X" instead of guessing
- Cross-reference constantly

#### Differential Review (Code Changes)

When reviewing code changes:

```bash
# 1. Identify what changed
git diff main..feature-branch

# 2. Analyze blast radius
- Which functions are modified?
- Which contracts depend on changed code?
- Are there storage layout changes?
- Do upgrades maintain compatibility?

# 3. Security impact analysis
- Does change introduce new entry points?
- Are access controls still valid?
- Could change break existing invariants?
- Are there new external calls?

# 4. Test coverage
- Are changed lines covered by tests?
- Do tests verify security properties?
- Are edge cases tested?
```

#### Variant Analysis (Finding Similar Bugs)

If you find a vulnerability, search for similar patterns:

```solidity
// Found: Unchecked return value
token.transfer(recipient, amount);

// Search for variants:
// 1. Other unchecked external calls
// 2. Similar patterns with different tokens
// 3. Same pattern in different contracts
// 4. Analogous issues with other external interfaces

// Pattern to search:
// - Any call to external contracts
// - Any use of low-level call/delegatecall
// - Any token transfer operations
```

#### Fix Review

When reviewing security fixes:

```
1. Does the fix address the root cause?
2. Does the fix introduce new issues?
3. Are there edge cases not covered?
4. Is the fix tested with PoC from original issue?
5. Does fix maintain backward compatibility?
6. Are there similar bugs not fixed?
```

#### Specification Compliance

Compare code behavior to documentation:

```
For each specification requirement:
1. Locate implementing code
2. Verify logic matches spec
3. Check edge cases are handled
4. Identify deviations
5. Assess security impact of deviations

Document:
- Requirement: "Users can only withdraw their balance"
- Implementation: ✅ Matches (line 42)
- Edge cases: ✅ Zero withdrawal blocked (line 40)
- Deviations: ⚠️ Spec silent on paused state, code allows admin bypass
```

#### Static Analysis Tooling

**CodeQL Patterns:**
```ql
// Find all external calls not followed by checks
from FunctionCall call
where call.getTarget().isExternalCall()
  and not exists(IfStmt check | 
    check.getCondition().getAChild*() = call
  )
select call, "Unchecked external call"
```

**Semgrep Rules:**
```yaml
rules:
  - id: unsafe-transfer
    pattern: |
      token.transfer($RECIPIENT, $AMOUNT);
    message: "Use safeTransfer instead of transfer"
    severity: WARNING
    languages: [solidity]
```

#### Insecure Defaults Detection

Patterns to check:

```solidity
// ❌ Bad: Fail-open authentication
function onlyAdmin() {
    if (msg.sender == admin) {
        // execute
    }
    // Missing else - anyone can execute!
}

// ✅ Good: Fail-closed authentication
function onlyAdmin() {
    require(msg.sender == admin, "Not admin");
    // execute
}

// ❌ Bad: Uninitialized state
bool public paused;  // Defaults to false, system starts unpaused

// ✅ Good: Explicit initialization
bool public paused = true;  // Explicit, starts paused (safe default)

// ❌ Bad: Hardcoded credentials
address private constant ADMIN = 0x1234...;

// ✅ Good: Constructor parameter
address private immutable i_admin;
constructor(address admin) {
    require(admin != address(0), "Invalid admin");
    i_admin = admin;
}
```

#### Sharp Edges (Dangerous APIs)

Common footguns to avoid:

```solidity
// ❌ DANGEROUS: transfer (2300 gas limit)
payable(recipient).transfer(amount);

// ✅ SAFE: call with success check
(bool success,) = recipient.call{value: amount}("");
require(success, "Transfer failed");

// ❌ DANGEROUS: tx.origin for auth
require(tx.origin == owner);

// ✅ SAFE: msg.sender
require(msg.sender == owner);

// ❌ DANGEROUS: delegatecall without understanding
target.delegatecall(data);

// ✅ SAFE: Understand storage layout, use libraries
library SafeMath { ... }

// ❌ DANGEROUS: Block timestamp for randomness
uint256 random = uint256(keccak256(abi.encode(block.timestamp)));

// ✅ SAFE: Chainlink VRF or commit-reveal
```

### Property-Based Testing (Trail of Bits)

Define properties that should ALWAYS hold:

```solidity
// PROPERTY: Sum of balances equals totalSupply
function invariant_sumOfBalances() public {
    uint256 sum = 0;
    for (uint256 i = 0; i < users.length; i++) {
        sum += balances[users[i]];
    }
    assertEq(sum, totalSupply);
}

// PROPERTY: User balance never negative (implicit in uint256)
// PROPERTY: Only authorized addresses can mint
function test_UnauthorizedCannotMint() public {
    vm.prank(attacker);
    vm.expectRevert("Unauthorized");
    token.mint(attacker, 1000);
}

// PROPERTY: Transfer decreases sender, increases recipient
function testProperty_TransferPreservesSum(
    address from,
    address to,
    uint256 amount
) public {
    vm.assume(from != to);
    vm.assume(balances[from] >= amount);
    
    uint256 sumBefore = balances[from] + balances[to];
    
    vm.prank(from);
    token.transfer(to, amount);
    
    uint256 sumAfter = balances[from] + balances[to];
    assertEq(sumBefore, sumAfter);
}
```

### Constant-Time Analysis

For cryptographic code, verify constant-time execution:

```rust
// Check for timing side-channels
// Patterns to avoid:
if secret_key[0] == guess {  // ❌ Early return on mismatch
    if secret_key[1] == guess { 
        // ...
    }
}

// Use constant-time comparison:
fn constant_time_eq(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false;
    }
    let mut result = 0u8;
    for i in 0..a.len() {
        result |= a[i] ^ b[i];
    }
    result == 0
}
```

---

## 🏗️ Development & Architecture

### Language Features (Solidity 0.8+)

#### Modern Solidity Best Practices

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

// Use custom errors (gas efficient)
error Unauthorized();
error InsufficientBalance(uint256 available, uint256 required);

// Use events for state changes
event Deposit(address indexed user, uint256 amount);
event Withdrawal(address indexed user, uint256 amount);

contract ModernContract {
    // State variable naming conventions
    address private s_owner;           // s_ for storage
    uint256 private immutable i_fee;   // i_ for immutable
    
    constructor(uint256 fee) {
        i_fee = fee;
        s_owner = msg.sender;
    }
    
    // Visibility: external > public for functions not called internally
    function deposit() external payable {
        emit Deposit(msg.sender, msg.value);
    }
    
    // Use custom errors over require strings
    modifier onlyOwner() {
        if (msg.sender != s_owner) revert Unauthorized();
        _;
    }
}
```

### Design Patterns

#### 1. **Factory Pattern** (Contract Deployment)

```solidity
contract TokenFactory {
    event TokenCreated(address indexed token, address indexed owner, string name);
    
    function createToken(
        string memory name,
        string memory symbol,
        uint256 totalSupply
    ) external returns (address) {
        Token newToken = new Token(name, symbol, totalSupply, msg.sender);
        emit TokenCreated(address(newToken), msg.sender, name);
        return address(newToken);
    }
    
    // With salt for deterministic addresses (CREATE2)
    function createToken2(
        string memory name,
        bytes32 salt
    ) external returns (address) {
        Token token = new Token{salt: salt}(name, msg.sender);
        return address(token);
    }
}
```

#### 2. **Clone Pattern (EIP-1167)** - Gas-Efficient Copies

```solidity
import "@openzeppelin/contracts/proxy/Clones.sol";

contract MinimalProxyFactory {
    address public immutable implementation;
    
    constructor(address _implementation) {
        implementation = _implementation;
    }
    
    function clone() external returns (address) {
        return Clones.clone(implementation);
    }
}
```

#### 3. **Upgradeable Proxy Patterns**

**UUPS (Universal Upgradeable Proxy Standard):**
```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MyContract is UUPSUpgradeable, OwnableUpgradeable {
    function initialize() public initializer {
        __Ownable_init(msg.sender);
        __UUPSUpgradeable_init();
    }
    
    function _authorizeUpgrade(address) internal override onlyOwner {}
}
```

**Security Note for Upgrades:**
- Always use storage gaps for future variables
- Never change storage layout in upgrades
- Use initializers instead of constructors
- Test upgrade paths thoroughly

#### 4. **Access Control Patterns**

```solidity
import "@openzeppelin/contracts/access/AccessControl.sol";

contract RoleBasedContract is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
    
    function mint(address to, uint256 amount) 
        external 
        onlyRole(MINTER_ROLE) 
    {
        // Mint logic
    }
}
```

### Development Workflows

#### Foundry Workflow

```bash
# Project initialization
forge init my-project
cd my-project

# Install dependencies
forge install openzeppelin/openzeppelin-contracts
forge install foundry-rs/forge-std

# Compile
forge build

# Test with verbosity
forge test -vvvv

# Gas optimization
forge test --gas-report

# Coverage
forge coverage

# Format code
forge fmt

# Create gas snapshots
forge snapshot

# Deploy
forge create src/MyContract.sol:MyContract \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --constructor-args arg1 arg2
```

#### Hardhat Workflow

```bash
# Initialize
npx hardhat init

# Compile
npx hardhat compile

# Test
npx hardhat test
npx hardhat test --grep "specific test"

# Coverage
npx hardhat coverage

# Deploy
npx hardhat run scripts/deploy.js --network sepolia

# Verify on Etherscan
npx hardhat verify --network sepolia CONTRACT_ADDRESS "Constructor Arg 1"
```

---

## 🧪 Testing Strategies

### Foundry Test Patterns

#### 1. **Unit Tests**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/Vault.sol";

contract VaultTest is Test {
    Vault public vault;
    address public alice = makeAddr("alice");
    address public bob = makeAddr("bob");
    
    function setUp() public {
        vault = new Vault();
        vm.deal(alice, 100 ether);
        vm.deal(bob, 100 ether);
    }
    
    function test_Deposit() public {
        vm.prank(alice);
        vault.deposit{value: 1 ether}();
        
        assertEq(vault.balances(alice), 1 ether);
        assertEq(address(vault).balance, 1 ether);
    }
    
    function test_RevertWhen_InsufficientBalance() public {
        vm.prank(alice);
        vm.expectRevert(Vault.InsufficientBalance.selector);
        vault.withdraw(1 ether);
    }
    
    function test_EmitsDepositEvent() public {
        vm.prank(alice);
        vm.expectEmit(true, true, false, true);
        emit Vault.Deposit(alice, 1 ether);
        vault.deposit{value: 1 ether}();
    }
}
```

#### 2. **Fuzz Testing**

```solidity
function testFuzz_Withdraw(uint256 amount) public {
    // Bound inputs to valid range
    amount = bound(amount, 0.01 ether, 100 ether);
    
    vm.startPrank(alice);
    vault.deposit{value: amount}();
    
    uint256 balanceBefore = alice.balance;
    vault.withdraw(amount);
    uint256 balanceAfter = alice.balance;
    
    assertEq(balanceAfter - balanceBefore, amount);
    assertEq(vault.balances(alice), 0);
    vm.stopPrank();
}

function testFuzz_CannotWithdrawMoreThanBalance(
    uint256 depositAmount,
    uint256 withdrawAmount
) public {
    depositAmount = bound(depositAmount, 1, 100 ether);
    withdrawAmount = bound(withdrawAmount, depositAmount + 1, 200 ether);
    
    vm.startPrank(alice);
    vault.deposit{value: depositAmount}();
    vm.expectRevert();
    vault.withdraw(withdrawAmount);
    vm.stopPrank();
}
```

#### 3. **Invariant Testing**

```solidity
contract InvariantVault is Test {
    Vault public vault;
    Handler public handler;
    
    function setUp() public {
        vault = new Vault();
        handler = new Handler(vault);
        
        targetContract(address(handler));
    }
    
    // Invariant: Total deposits = contract balance
    function invariant_balanceEqualsSum() public {
        assertEq(
            address(vault).balance,
            handler.ghost_depositSum() - handler.ghost_withdrawSum()
        );
    }
    
    // Invariant: Sum of user balances = contract balance
    function invariant_userBalancesSumMatchesTotal() public view {
        // Check all users
    }
}
```

#### 4. **Fork Testing**

```solidity
contract ForkTest is Test {
    uint256 mainnetFork;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    
    function setUp() public {
        mainnetFork = vm.createFork(vm.envString("MAINNET_RPC_URL"));
        vm.selectFork(mainnetFork);
    }
    
    function test_InteractWithMainnetUSDC() public {
        IERC20 usdc = IERC20(USDC);
        uint256 balance = usdc.balanceOf(address(this));
        // Test against real mainnet state
    }
}
```

### Property-Based Testing (Advanced)

Define properties that should always hold:

```solidity
// Property: Withdrawing should never increase vault balance
function testProperty_WithdrawDecreasesBalance() public {
    uint256 balanceBefore = address(vault).balance;
    
    if (vault.balances(alice) > 0) {
        vm.prank(alice);
        vault.withdraw(vault.balances(alice));
        
        assert(address(vault).balance <= balanceBefore);
    }
}
```

---

## 💰 DeFi & Token Standards

### AMM (Automated Market Maker) Development

#### Constant Product Formula (Uniswap V2 Style)

```solidity
library SwapMath {
    uint256 constant FEE_NUMERATOR = 997;    // 0.3% fee
    uint256 constant FEE_DENOMINATOR = 1000;
    
    /// @notice Calculate output amount for constant product AMM (x * y = k)
    /// @dev Includes 0.3% fee deduction
    function getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) internal pure returns (uint256 amountOut) {
        require(amountIn > 0, "INSUFFICIENT_INPUT");
        require(reserveIn > 0 && reserveOut > 0, "INSUFFICIENT_LIQUIDITY");
        
        uint256 amountInWithFee = amountIn * FEE_NUMERATOR;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = (reserveIn * FEE_DENOMINATOR) + amountInWithFee;
        
        amountOut = numerator / denominator;
    }
    
    /// @notice Calculate price impact in basis points (10000 = 100%)
    function getPriceImpact(
        uint256 amountIn,
        uint256 reserveIn
    ) internal pure returns (uint256) {
        return (amountIn * 10000) / (reserveIn + amountIn);
    }
}
```

#### Liquidity Pool Pattern

```solidity
contract LiquidityPool {
    uint256 public reserveA;
    uint256 public reserveB;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    
    function addLiquidity(uint256 amountA, uint256 amountB) 
        external 
        returns (uint256 liquidity) 
    {
        if (totalSupply == 0) {
            liquidity = Math.sqrt(amountA * amountB);
        } else {
            liquidity = Math.min(
                (amountA * totalSupply) / reserveA,
                (amountB * totalSupply) / reserveB
            );
        }
        
        // Transfer tokens, mint LP tokens
        // ... implementation
        
        reserveA += amountA;
        reserveB += amountB;
        totalSupply += liquidity;
        balanceOf[msg.sender] += liquidity;
    }
}
```

### Lending Protocol Patterns

#### Interest Rate Model

```solidity
contract InterestRateModel {
    uint256 public constant BASE_RATE = 2e16;        // 2%
    uint256 public constant MULTIPLIER = 3e17;       // 30%
    uint256 public constant JUMP_MULTIPLIER = 5e18;  // 500%
    uint256 public constant KINK = 8e17;             // 80%
    
    /// @notice Calculate borrow rate based on utilization
    function getBorrowRate(
        uint256 cash,
        uint256 borrows,
        uint256 reserves
    ) public pure returns (uint256) {
        uint256 utilization = getUtilization(cash, borrows, reserves);
        
        if (utilization <= KINK) {
            return BASE_RATE + (utilization * MULTIPLIER) / 1e18;
        } else {
            uint256 normalRate = BASE_RATE + (KINK * MULTIPLIER) / 1e18;
            uint256 excessUtil = utilization - KINK;
            return normalRate + (excessUtil * JUMP_MULTIPLIER) / 1e18;
        }
    }
    
    function getUtilization(
        uint256 cash,
        uint256 borrows,
        uint256 reserves
    ) public pure returns (uint256) {
        if (borrows == 0) return 0;
        return (borrows * 1e18) / (cash + borrows - reserves);
    }
}
```

### Oracle Integration

#### Chainlink Price Feed

```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceConsumer {
    AggregatorV3Interface internal priceFeed;
    uint256 public constant STALENESS_THRESHOLD = 3600; // 1 hour
    
    constructor(address _priceFeed) {
        priceFeed = AggregatorV3Interface(_priceFeed);
    }
    
    /// @notice Get latest price with staleness check
    function getLatestPrice() public view returns (uint256) {
        (
            uint80 roundId,
            int256 price,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();
        
        require(price > 0, "Invalid price");
        require(answeredInRound >= roundId, "Stale price");
        require(block.timestamp - updatedAt < STALENESS_THRESHOLD, "Price too old");
        
        return uint256(price);
    }
}
```

#### TWAP (Time-Weighted Average Price)

```solidity
contract TWAP {
    struct Observation {
        uint32 timestamp;
        uint224 priceCumulative;
    }
    
    Observation[2] public observations;
    
    function update(uint256 price) external {
        uint32 timeElapsed = uint32(block.timestamp) - observations[0].timestamp;
        
        observations[1] = observations[0];
        observations[0] = Observation({
            timestamp: uint32(block.timestamp),
            priceCumulative: observations[0].priceCumulative + uint224(price * timeElapsed)
        });
    }
    
    function consult(uint32 period) external view returns (uint256) {
        Observation memory oldest = observations[1];
        uint32 timeElapsed = uint32(block.timestamp) - oldest.timestamp;
        require(timeElapsed >= period, "Period not elapsed");
        
        uint224 priceCumulativeDelta = observations[0].priceCumulative - oldest.priceCumulative;
        return uint256(priceCumulativeDelta) / timeElapsed;
    }
}
```

### Token Standards

#### ERC-20 Implementation

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, Ownable {
    uint256 public constant MAX_SUPPLY = 1_000_000 * 10**18;
    
    constructor() ERC20("MyToken", "MTK") Ownable(msg.sender) {
        _mint(msg.sender, 100_000 * 10**18);
    }
    
    function mint(address to, uint256 amount) external onlyOwner {
        require(totalSupply() + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
    }
}
```

#### ERC-721 (NFT) Implementation

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyNFT is ERC721, Ownable {
    uint256 private _tokenIdCounter;
    string private _baseTokenURI;
    
    constructor(string memory baseURI) 
        ERC721("MyNFT", "MNFT") 
        Ownable(msg.sender) 
    {
        _baseTokenURI = baseURI;
    }
    
    function mint(address to) external onlyOwner returns (uint256) {
        uint256 tokenId = _tokenIdCounter++;
        _safeMint(to, tokenId);
        return tokenId;
    }
    
    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }
}
```

#### ERC-1155 (Multi-Token) Implementation

```solidity
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";

contract GameItems is ERC1155 {
    uint256 public constant SWORD = 0;
    uint256 public constant SHIELD = 1;
    uint256 public constant POTION = 2;
    
    constructor() ERC1155("https://game.example/api/item/{id}.json") {
        _mint(msg.sender, SWORD, 100, "");
        _mint(msg.sender, SHIELD, 200, "");
        _mint(msg.sender, POTION, 500, "");
    }
}
```

---

## 🚀 Deployment & Verification

### Deployment Scripts

#### Foundry Deploy Script

```solidity
// script/Deploy.s.sol
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/MyContract.sol";

contract DeployScript is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        vm.startBroadcast(deployerPrivateKey);
        
        MyContract myContract = new MyContract(
            vm.envAddress("CONSTRUCTOR_ARG1"),
            vm.envUint("CONSTRUCTOR_ARG2")
        );
        
        console.log("Deployed to:", address(myContract));
        
        vm.stopBroadcast();
    }
}
```

```bash
# Deploy
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url $SEPOLIA_RPC_URL \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

#### Hardhat Deploy Script

```javascript
// scripts/deploy.js
const { ethers } = require("hardhat");

async function main() {
  const [deployer] = await ethers.getSigners();
  
  console.log("Deploying with account:", deployer.address);
  
  const MyContract = await ethers.getContractFactory("MyContract");
  const myContract = await MyContract.deploy(arg1, arg2);
  
  await myContract.waitForDeployment();
  
  console.log("Contract deployed to:", await myContract.getAddress());
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### Contract Verification

#### Foundry Verification

```bash
# Verify after deployment
forge verify-contract \
  --chain-id 11155111 \
  --num-of-optimizations 200 \
  --watch \
  --constructor-args $(cast abi-encode "constructor(address,uint256)" $ARG1 $ARG2) \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --compiler-version v0.8.24+commit.e11b9ed9 \
  $CONTRACT_ADDRESS \
  src/MyContract.sol:MyContract
```

#### Hardhat Verification

```bash
npx hardhat verify \
  --network sepolia \
  --constructor-args arguments.js \
  CONTRACT_ADDRESS
```

```javascript
// arguments.js
module.exports = [
  "0x...", // arg1
  "1000"   // arg2
];
```

---

## 🔍 Code Review & Analysis

### Ask Questions If Underspecified

Before implementing, clarify:

**When to Ask:**
- Request has multiple plausible interpretations
- Success criteria unclear
- Constraints not specified
- Environment details missing

**Format:**
```
I need clarification on a few points before proceeding:

1. **Access Control**: Should only the owner be able to call this function?
   - [ ] Yes, only owner
   - [ ] No, any user
   - [x] Role-based (MINTER_ROLE)

2. **Fee Handling**: How should protocol fees be calculated?
   - [ ] Flat fee (specify amount)
   - [x] Percentage (0.3% of transaction)
   - [ ] Tiered based on volume

3. **Upgrade Mechanism**: Should this contract be upgradeable?
   - [x] Yes, using UUPS
   - [ ] Yes, using Transparent Proxy
   - [ ] No, immutable

Proceeding with checked assumptions unless you specify otherwise.
```

### Semgrep Rule Creation

For custom vulnerability detection:

```yaml
rules:
  - id: missing-zero-address-check
    pattern: |
      constructor($PARAM: address) {
        ...
        owner = $PARAM;
      }
    pattern-not: |
      require($PARAM != address(0), ...);
    message: "Constructor missing zero address check for owner"
    languages: [solidity]
    severity: WARNING

  - id: unchecked-call-return
    pattern: |
      $ADDR.call(...)
    pattern-not: |
      (bool $SUCCESS, ...) = $ADDR.call(...);
      require($SUCCESS, ...);
    message: "Low-level call return value not checked"
    languages: [solidity]
    severity: ERROR
```

### Modern Python Development

For Python scripts in your Solidity project:

```bash
# Use uv for dependency management
uv init
uv add web3 eth-abi

# Use ruff for linting and formatting
ruff check .
ruff format .

# Use pytest for testing
pytest tests/ -v
```

---

## 📝 Code Style & Best Practices

### Naming Conventions

```solidity
// State variables
uint256 private s_totalSupply;        // s_ for storage
uint256 private immutable i_owner;    // i_ for immutable
uint256 private constant MAX_SUPPLY = 1000; // UPPER_CASE for constants

// Functions
function _internalHelper() internal {} // _ for internal/private
function publicFunction() external {}   // camelCase for public/external

// Events
event Transfer(address indexed from, address indexed to, uint256 amount);

// Errors
error InsufficientBalance();
error Unauthorized(address caller);
```

### NatSpec Documentation

```solidity
/// @title A Secure Vault Contract
/// @author Your Name
/// @notice This contract allows users to deposit and withdraw ETH securely
/// @dev Implements CEI pattern and reentrancy guards
/// @custom:security-note Uses OpenZeppelin ReentrancyGuard
contract Vault {
    /// @notice Deposits ETH into the vault
    /// @dev Emits a Deposit event upon success
    /// @return success Whether the deposit was successful
    function deposit() external payable returns (bool success) {
        // Implementation
    }
    
    /// @notice Withdraws ETH from the vault
    /// @dev Follows CEI pattern to prevent reentrancy
    /// @param amount The amount to withdraw in wei
    /// @custom:security Uses nonReentrant modifier
    /// @custom:invariant balances[msg.sender] >= 0
    function withdraw(uint256 amount) external {
        // Implementation
    }
}
```

### Gas Optimization Tips

```solidity
// ❌ Bad: Using storage in loops
function sumBad() external view returns (uint256) {
    uint256 sum = 0;
    for (uint256 i = 0; i < s_array.length; i++) {
        sum += s_array[i];
    }
    return sum;
}

// ✅ Good: Cache array length
function sumGood() external view returns (uint256) {
    uint256 sum = 0;
    uint256 length = s_array.length;
    for (uint256 i = 0; i < length; i++) {
        sum += s_array[i];
    }
    return sum;
}

// ✅ Good: Use unchecked for safe increments
function sumUnchecked() external view returns (uint256) {
    uint256 sum = 0;
    uint256 length = s_array.length;
    for (uint256 i = 0; i < length;) {
        sum += s_array[i];
        unchecked { ++i; }
    }
    return sum;
}

// ✅ Good: Use custom errors over require strings
error InvalidAmount();
require(amount > 0); // 23k gas
if (amount == 0) revert InvalidAmount(); // 20k gas

// ✅ Good: Use calldata for read-only arrays
function process(uint256[] calldata data) external {
    // calldata is cheaper than memory for read-only
}

// ✅ Good: Pack structs efficiently
struct Packed {
    uint128 a;  // 16 bytes
    uint128 b;  // 16 bytes  -> Same slot (32 bytes)
}

struct Unpacked {
    uint256 a;  // 32 bytes -> Slot 1
    uint128 b;  // 16 bytes -> Slot 2 (wastes 16 bytes)
}
```

### Common Pitfalls & Solutions

#### Stack Too Deep Error

```solidity
// ❌ Problem: Too many local variables
function complexFunction(uint256 a, uint256 b, uint256 c, ...) external {
    uint256 var1 = a + b;
    uint256 var2 = c * 2;
    // ... 10 more variables -> Stack too deep!
}

// ✅ Solution 1: Use structs
struct Params {
    uint256 var1;
    uint256 var2;
    uint256 var3;
}

function complexFunction(Params memory params) external {
    // Work with params.var1, params.var2, etc.
}

// ✅ Solution 2: Block scoping
function complexFunction() external {
    {
        uint256 temp1 = calculation1();
        // Use temp1
    }
    {
        uint256 temp2 = calculation2();
        // Use temp2
    }
}

// ✅ Solution 3: Split into helper functions
function complexFunction() external {
    uint256 result1 = _helperFunction1();
    uint256 result2 = _helperFunction2();
}
```

#### Contract Size Limit

```bash
# Check sizes
forge build --sizes

# Solutions:
# 1. Split logic into libraries
# 2. Use Diamond pattern (EIP-2535)
# 3. Remove unused code
# 4. Optimize string storage
```

### Multi-Chain Considerations

```solidity
// Handle different chains
if (block.chainid == 1) {
    // Ethereum mainnet
} else if (block.chainid == 137) {
    // Polygon
} else if (block.chainid == 42161) {
    // Arbitrum
}

// Chain-specific addresses
mapping(uint256 => address) public chainToUSDC;

constructor() {
    chainToUSDC[1] = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;    // ETH
    chainToUSDC[137] = 0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174;  // Polygon
    chainToUSDC[42161] = 0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8; // Arbitrum
}
```

---

## 🎯 Quick Reference Commands

### Foundry

```bash
forge init                          # Initialize project
forge install <repo>                # Install dependency
forge build                         # Compile
forge test                          # Run tests
forge test -vvvv                    # Verbose tests
forge test --match-test testName    # Run specific test
forge test --match-contract Contract # Test specific contract
forge coverage                      # Coverage report
forge snapshot                      # Gas snapshot
forge fmt                           # Format code
forge doc                           # Generate docs
cast call <addr> "func()" --rpc-url # Call view function
cast send <addr> "func()" --rpc-url # Send transaction
```

### Hardhat

```bash
npx hardhat compile                 # Compile
npx hardhat test                    # Run tests
npx hardhat coverage                # Coverage
npx hardhat node                    # Start local node
npx hardhat run scripts/deploy.js   # Run script
npx hardhat verify CONTRACT_ADDRESS # Verify on Etherscan
npx hardhat clean                   # Clean artifacts
```

### Analysis Tools

```bash
slither .                           # Static analysis
slither . --detect reentrancy       # Specific detector
mythril analyze contract.sol        # Symbolic execution
echidna contract.sol                # Fuzz testing
semgrep --config=p/solidity .       # Pattern detection
surya graph contracts/**/*.sol | dot -Tpng > graph.png # Visualize
solhint 'contracts/**/*.sol'        # Linter
```

---

## ⚠️ Security Warnings

**NEVER use these dangerous patterns:**

1. ❌ Generating/storing private keys in contracts
2. ❌ Exposing private keys to AI context
3. ❌ Using `tx.origin` for authentication
4. ❌ Defaulting to mainnet in scripts
5. ❌ Ignoring return values from `call`
6. ❌ Using `transfer`/`send` (gas limit issues)
7. ❌ State changes after external calls (reentrancy)
8. ❌ Hardcoded credentials or API keys
9. ❌ Fail-open security patterns
10. ❌ Uninitialized proxy implementations

**Always:**

✅ Test on testnet first  
✅ Use hardware wallets for mainnet  
✅ Implement timelock for upgrades  
✅ Multi-sig for admin functions  
✅ Professional audits before mainnet  
✅ Bug bounty program  
✅ Document all assumptions and invariants  
✅ Verify storage layouts in upgrades  
✅ Check for variant vulnerabilities  
✅ Build context before security review  

---

## 📚 Resources

### Documentation
- [Solidity Documentation](https://docs.soliditylang.org/)
- [Foundry Book](https://book.getfoundry.sh/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [Ethereum Improvement Proposals](https://eips.ethereum.org/)

### Security
- [Smart Contract Security Best Practices](https://consensys.github.io/smart-contract-best-practices/)
- [Trail of Bits Building Secure Contracts](https://github.com/crytic/building-secure-contracts)
- [SWC Registry](https://swcregistry.io/)
- [Solidity by Example](https://solidity-by-example.org/)

### Tools
- [Slither](https://github.com/crytic/slither)
- [Echidna](https://github.com/crytic/echidna)
- [Semgrep](https://semgrep.dev/)
- [Foundry](https://github.com/foundry-rs/foundry)

---

*This guide combines [awesome-solidity-skills](https://github.com/gonzaloetjo/awesome-solidity-skills) (40+ development skills) with [Trail of Bits Skills](https://github.com/trailofbits/skills) (25+ security analysis techniques) for comprehensive AI-assisted Solidity development.*

**Skills Included:**
- Security & Auditing: 14 skills
- Trail of Bits Security Analysis: 25 plugins
- Development & Architecture: 15 skills
- Testing: 3 skills + property-based testing
- DeFi & Tokens: 5 skills
- Deployment & Tooling: 2 skills
- Code Review: 10 Trail of Bits techniques
