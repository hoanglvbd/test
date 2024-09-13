# FuseBridge Review

FuseBridge contract provides a comprehensive solution for bridging tokens between networks, with features for swapping, permitting, and managing the bridging process. It incorporates important security measures, upgradeability, and flexibility in its design, making it a robust system for cross-chain token transfers. The contract verifies the caller's authorization, checking request statuses to prevent double-spending, and using a delegate call mechanism for swaps to maintain the contract's context. The fee calculation and event emissions provide transparency and allow for off-chain tracking of bridge operations.

The core functionality of the FuseBridge contract, enables:

1. Bridging of tokens from one network to another
2. Swapping tokens before bridging
3. Releasing bridged tokens on the destination network
4. Swapping tokens upon release on the destination network

### Main operations flowchart

```mermaid
flowchart TD
    A[Start] --> B{Choose Operation}
    B -->|Bridge| C[bridge]
    B -->|Swap and Bridge| D[swapAndBridge]
    B -->|Release| E[release]
    B -->|Swap and Release| F[swapAndRelease]

    subgraph Bridge Operation
    C --> C1{Is caller authorized?}
    C1 -->|No| C2[Revert: Unauthorized]
    C1 -->|Yes| C3[Burn FuseWrappedTon tokens]
    C3 --> C4[Calculate fee]
    C4 --> C5[Emit BridgeInitiated event]
    end

    subgraph Swap and Bridge Operation
    D --> D1{Is caller authorized?}
    D1 -->|No| D2[Revert: Unauthorized]
    D1 -->|Yes| D3[Delegate call to FuseSwap]
    D3 --> D4{Swap successful?}
    D4 -->|No| D5[Revert: Swap failed]
    D4 -->|Yes| D6[Verify output token is FuseWrappedTon]
    D6 --> D7[Burn FuseWrappedTon tokens]
    D7 --> D8[Calculate fee]
    D8 --> D9[Emit BridgeInitiated event]
    end

    subgraph Release Operation
    E --> E1{Is caller an executor?}
    E1 -->|No| E2[Revert: OnlyExecutor]
    E1 -->|Yes| E3{Check request status}
    E3 -->|Released| E4[Revert: AlreadyReleased]
    E3 -->|Prevented| E5[Revert: AlreadyPrevented]
    E3 -->|Not set| E6[Set status to RELEASED]
    E6 --> E7[Mint FuseWrappedTon tokens]
    E7 --> E8[Emit Release event]
    end

    subgraph Swap and Release Operation
    F --> F1{Is caller an executor?}
    F1 -->|No| F2[Revert: OnlyExecutor]
    F1 -->|Yes| F3{Check request status}
    F3 -->|Released| F4[Revert: AlreadyReleased]
    F3 -->|Prevented| F5[Revert: AlreadyPrevented]
    F3 -->|Not set| F6[Set status to RELEASED]
    F6 --> F7[Mint FuseWrappedTon tokens]
    F7 --> F8[Delegate call to FuseSwap]
    F8 --> F9{Swap successful?}
    F9 -->|No| F10[Revert: Swap failed]
    F9 -->|Yes| F11[Transfer swapped tokens]
    F11 --> F12[Emit Release event]
    end

    C5 --> G[End]
    D9 --> G
    E8 --> G
    F12 --> G
```

1. Bridge Operation:

The user calls the `bridge` function with the amount to bridge and the destination address.
The contract checks if the caller is authorized (this is implied in the contract through the use of tokens).
If authorized, the contract burns the specified amount of FuseWrappedTon tokens from the caller's account.
The contract calculates the fee based on the bridged amount.
Finally, it emits a BridgeInitiated event with the details of the transaction.

2. Swap and Bridge Operation:

The user calls the `swapAndBridge` function with swap data and the destination address.
The contract checks if the caller is authorized.
If authorized, it performs a delegate call to the FuseSwap contract to execute the swap.
If the swap is successful, it verifies that the output token is FuseWrappedTon.
The contract then burns the received FuseWrappedTon tokens.
It calculates the fee based on the bridged amount.
Finally, it emits a BridgeInitiated event.

3. Release Operation:

An executor calls the `release` function with the request ID, recipient address, and amount.
The contract checks if the caller is an authorized executor.
If authorized, it checks the status of the request.
If the request is not already released or prevented, it sets the status to RELEASED.
The contract mints the specified amount of FuseWrappedTon tokens to the recipient.
It emits a Release event with the transaction details.

4. Swap and Release Operation:

An executor calls the `swapAndRelease` function with the request ID, recipient address, amount, and swap data.
The contract checks if the caller is an authorized executor.
If authorized, it checks the status of the request.
If the request is not already released or prevented, it sets the status to RELEASED.
The contract mints the specified amount of FuseWrappedTon tokens to itself.
It performs a delegate call to the FuseSwap contract to execute the swap.
If the swap is successful, it transfers the swapped tokens to the recipient.
Finally, it emits a Release event with the transaction details.

### Contract Structure and Inheritance

This class diagram shows the inheritance structure and the main components of the FuseBridge contract.

```mermaid
classDiagram
    Initializable <|-- OwnableUpgradeable
    Initializable <|-- UUPSUpgradeable
    OwnableUpgradeable <|-- FuseBridge
    UUPSUpgradeable <|-- FuseBridge
    WithdrawFunds <|-- FuseBridge
    IFuseBridge <|.. FuseBridge

    class Initializable {
        <<abstract>>
        #_disableInitializers()
    }
    class OwnableUpgradeable {
        <<abstract>>
        #__Ownable_init(address)
        +transferOwnership(address)
        +renounceOwnership()
    }
    class UUPSUpgradeable {
        <<abstract>>
        #__UUPSUpgradeable_init()
        +upgradeToAndCall(address, bytes)
    }
    class WithdrawFunds {
        <<abstract>>
        #_withdrawEther(uint256, address)
        #_withdrawERC20(address, uint256, address)
    }
    class IFuseBridge {
        <<interface>>
        +bridge(uint256, string)
        +swapAndBridge(bytes, string)
        +swapAndBridgeWithPermit(SwapAndBridgeWithPermitParams)
        +release(bytes32, address, uint256)
        +swapAndRelease(bytes32, address, uint256, bytes)
        +prevent(bytes32)
        +setFuseSwap(address)
        +setExecutorStatus(address, bool)
        +setFeeBps(uint256)
    }
    class FuseBridge {
        +FuseWrappedTon fuseWrappedTon
        +address fuseSwap
        +uint256 feeBps
        +initialize(address, uint256, address)
        +bridge(uint256, string)
        +swapAndBridge(bytes, string)
        +swapAndBridgeWithPermit(SwapAndBridgeWithPermitParams)
        +release(bytes32, address, uint256)
        +swapAndRelease(bytes32, address, uint256, bytes)
        +prevent(bytes32)
        +setFuseSwap(address)
        +setExecutorStatus(address, bool)
        +setFeeBps(uint256)
        +withdrawEther(uint256, address)
        +withdrawERC20(address, uint256, address)
    }
```

### Key Components and State Variables

```mermaid
classDiagram
    class FuseBridge {
        <<contract>>
        +uint8 STATUS_RELEASED
        +uint8 STATUS_PREVENTED
        +uint256 MAX_FEE_BPS
        +FuseWrappedTon fuseWrappedTon
        +address fuseSwap
        +uint256 feeBps
        -mapping(address => bool) executors
        -mapping(bytes32 => uint8) requestStatus
        +initialize(address owner, uint256 feeBps_, address fuseWrappedTon_)
        +bridge(uint256 sourceAmount, string destinationAddress)
        +swapAndBridge(bytes swapData, string destinationAddress)
        +swapAndBridgeWithPermit(SwapAndBridgeWithPermitParams params)
        +release(bytes32 requestId, address to, uint256 amount)
        +swapAndRelease(bytes32 requestId, address to, uint256 amount, bytes swapData)
        +prevent(bytes32 requestId)
        +setFuseSwap(address newFuseSwap)
        +setExecutorStatus(address executor, bool status)
        +setFeeBps(uint256 newFeeBps)
        +withdrawEther(uint256 amount, address receiver)
        +withdrawERC20(address token, uint256 amount, address receiver)
        -_getFeeAmount(uint256 amount) internal
        -_authorizeUpgrade(address newImplementation) internal
    }
```

1. Constants:

- `STATUS_RELEASED` (uint8): Represents the status of a released transaction (value: 0x01).
- `STATUS_PREVENTED` (uint8): Represents the status of a prevented transaction (value: 0x02).
- `MAX_FEE_BPS` (uint256): The maximum fee in basis points (value: 500, equivalent to 5%).

2. State Variables:

- `fuseWrappedTon` (FuseWrappedTon): Address of the FuseWrappedTon token contract.
- `fuseSwap` (address): Address of the FuseSwap contract used for token swaps.
- `feeBps` (uint256): Current fee in basis points for bridge operations.
- `executors` (mapping): A mapping of addresses to boolean values, tracking authorized executors.
- `requestStatus` (mapping): A mapping of request IDs (bytes32) to their status (uint8).

3. Main Functions:

- `initialize`: Initializes the contract with owner, fee, and FuseWrappedTon address.
- `bridge`: Handles the basic bridge operation.
- `swapAndBridge`: Performs a swap before bridging.
- `swapAndBridgeWithPermit`: Similar to swapAndBridge but uses EIP-2612 permit for approval.
- `release`: Releases bridged tokens on the destination chain.
- `swapAndRelease`: Swaps tokens upon release on the destination chain.
- `prevent`: Prevents a specific bridge request from being executed.

4. Administrative Functions:

- `setFuseSwap`: Sets the address of the FuseSwap contract.
- `setExecutorStatus`: Adds or removes an address from the list of authorized executors.
- `setFeeBps`: Updates the fee in basis points.
- `withdrawEther`: Allows the owner to withdraw Ether from the contract.
- `withdrawERC20`: Allows the owner to withdraw ERC20 tokens from the contract.

5. Internal Functions:

- `_getFeeAmount`: Calculates the fee amount for a given source amount.
- `_authorizeUpgrade`: Authorizes an upgrade (part of the UUPS upgrade pattern).

### Sequence Diagram

This sequence diagram illustrates the detailed flow of a bridging operation, including the swap and prevent scenarios.

```mermaid
sequenceDiagram
    actor User
    participant EVMFuseBridge
    participant FuseWrappedTon
    participant DEX
    participant ValidatorNetwork
    participant Executor
    participant TONFuseBridge
    participant RequestStatus

    Note over User,RequestStatus: Scenario 1: Successful Swap and Bridge (EVM to TON)

    User->>EVMFuseBridge: swapAndBridge(swapData, destinationAddress)
    EVMFuseBridge->>DEX: Execute Swap
    DEX-->>EVMFuseBridge: Return Swapped Tokens
    EVMFuseBridge->>FuseWrappedTon: burn(swappedAmount)
    FuseWrappedTon-->>EVMFuseBridge: Tokens Burned
    EVMFuseBridge-->>ValidatorNetwork: Emit BridgeInitiated Event

    loop For each Validator
        ValidatorNetwork->>ValidatorNetwork: Validate Event
    end

    ValidatorNetwork->>Executor: Notify (Threshold Met)
    Executor->>TONFuseBridge: Release(amount, user, requestId)
    TONFuseBridge->>RequestStatus: Release(amount, user, requestId)
    RequestStatus->>User: Transfer Funds

    Note over User,RequestStatus: Scenario 2: Bridge Prevention (EVM to TON)

    User->>EVMFuseBridge: bridge(amount, destinationAddress)
    EVMFuseBridge->>FuseWrappedTon: burn(amount)
    FuseWrappedTon-->>EVMFuseBridge: Tokens Burned
    EVMFuseBridge-->>ValidatorNetwork: Emit BridgeInitiated Event

    loop For each Validator
        ValidatorNetwork->>ValidatorNetwork: Validate Event
        ValidatorNetwork->>ValidatorNetwork: Detect Suspicious Activity
    end

    ValidatorNetwork->>Executor: Notify (Suspicious Activity)
    Executor->>TONFuseBridge: Prevent(user, requestId)
    TONFuseBridge->>RequestStatus: Prevent(user, requestId)
    RequestStatus-->>Executor: Prevented Status Confirmed

    Note over User,RequestStatus: Scenario 3: Swap and Bridge (TON to EVM)

    User->>TONFuseBridge: Bridge(amount, destinationAddress)
    TONFuseBridge->>TONFuseBridge: Lock Funds
    TONFuseBridge-->>ValidatorNetwork: Emit BridgeInitiated Event

    loop For each Validator
        ValidatorNetwork->>ValidatorNetwork: Validate Event
    end

    ValidatorNetwork->>Executor: Notify (Threshold Met)
    Executor->>EVMFuseBridge: swapAndRelease(requestId, to, amount, swapData)
    EVMFuseBridge->>FuseWrappedTon: mint(address(this), amount)
    FuseWrappedTon-->>EVMFuseBridge: Tokens Minted
    EVMFuseBridge->>DEX: Execute Swap
    DEX-->>EVMFuseBridge: Return Swapped Tokens
    EVMFuseBridge->>User: Transfer Swapped Tokens
```

### Key Functions Explained

1. `bridge(uint256 sourceAmount, string calldata destinationAddress)`

- Purpose: Initiates a bridge operation from EVM to TON.
- Functionality:
  - Burns the specified amount of FuseWrappedTon tokens from the sender.
  - Calculates and deducts the fee.
  - Emits a BridgeInitiated event with the remaining amount.

2. `swapAndBridge(bytes calldata swapData, string calldata destinationAddress)`

- Purpose: Allows users to swap tokens and then bridge in one transaction.
- Functionality:
  - Executes a swap operation using the provided swap data.
  - Burns the resulting FuseWrappedTon tokens.
  - Emits a BridgeInitiated event.

2. `swapAndBridgeWithPermit(SwapAndBridgeWithPermitParams calldata params)`

- Purpose: Similar to swapAndBridge, but uses EIP-2612 permit for approval.
- Functionality:
  - Attempts to execute the permit function for token approval.
  - Calls swapAndBridge with the approved tokens.

3. `release(bytes32 requestId, address to, uint256 amount)`

- Purpose: Releases tokens on the EVM chain for completed TON-to-EVM bridges.
- Access: Only callable by authorized executors.
- Functionality:
  - Checks if the request has already been processed.
  - Mints FuseWrappedTon tokens to the specified address.
  - Emits a Release event.

4. `swapAndRelease(bytes32 requestId, address to, uint256 amount, bytes calldata swapData)`

- Purpose: Releases tokens and immediately swaps them for the user.
- Access: Only callable by authorized executors.
- Functionality:
  - Mints FuseWrappedTon tokens to the contract.
  - Executes a swap operation.
  - Transfers the resulting tokens to the user.
  - Emits a Release event.

5. `prevent(bytes32 requestId)`

- Purpose: Prevents a specific bridge request from being processed.
- Access: Only callable by authorized executors.
- Functionality:
  - Marks a request as prevented, blocking its future processing.
  - Emits a Prevent event.

6. `setFuseSwap(address newFuseSwap)`

- Purpose: Sets the address of the FuseSwap contract used for token swaps.
- Access: Only callable by the contract owner.
- Functionality: Updates the FuseSwap contract address.

7. `setExecutorStatus(address executor, bool status)`

- Purpose: Manages the list of authorized executors.
- Access: Only callable by the contract owner.
- Functionality: Adds or removes executors who can call sensitive functions.

8. `setFeeBps(uint256 newFeeBps)`

- Purpose: Updates the fee percentage for bridge operations.
- Access: Only callable by the contract owner.
- Functionality: Sets a new fee in basis points (BPS).

### Security and Access Control

```mermaid
graph TD
    subgraph "Roles and Permissions"
        Owner[Owner]
        Executor[Executor]
        User[User]
    end

    subgraph "FuseBridge Contract"
        setExecutorStatus[setExecutorStatus]
        setFeeBps[setFeeBps]
        setFuseSwap[setFuseSwap]
        bridge[bridge]
        swapAndBridge[swapAndBridge]
        release[release]
        prevent[prevent]
        swapAndRelease[swapAndRelease]
    end

    subgraph "FuseWrappedTon Contract"
        mint[mint]
        burn[burn]
        setFuseBridge[setFuseBridge]
    end

    subgraph "Security Checks"
        executorCheck{Executor Check}
        ownerCheck{Owner Check}
        releaseCheck{Already Released?}
        preventCheck{Already Prevented?}
        feeCheck{Valid Fee?}
    end

    Owner -->|can call| setExecutorStatus
    Owner -->|can call| setFeeBps
    Owner -->|can call| setFuseSwap
    Owner -->|can call| setFuseBridge

    Executor -->|can call| release
    Executor -->|can call| prevent
    Executor -->|can call| swapAndRelease

    User -->|can call| bridge
    User -->|can call| swapAndBridge

    setExecutorStatus --> ownerCheck
    setFeeBps --> ownerCheck
    setFuseSwap --> ownerCheck

    release --> executorCheck
    prevent --> executorCheck
    swapAndRelease --> executorCheck

    release --> releaseCheck
    release --> preventCheck

    prevent --> releaseCheck
    prevent --> preventCheck

    setFeeBps --> feeCheck

    FuseBridge[FuseBridge Contract] -->|can call| mint
    FuseBridge -->|can call| burn

    classDef securityCheck fill:#f9f,stroke:#333,stroke-width:2px;
    class executorCheck,ownerCheck,releaseCheck,preventCheck,feeCheck securityCheck;
```

### Key components

1. Roles and Permissions:

- Owner: Has the highest level of control, can manage executors and set critical parameters.
- Executor: Can perform sensitive operations like releasing funds or preventing transactions.
- User: Can initiate bridge and swap operations.

2. FuseBridge Contract:

- Contains the main functions for bridging operations and system management.
- Functions are color-coded based on who can access them.

3. FuseWrappedTon Contract:

- Contains functions for minting and burning wrapped tokens.
- Can only be called by the FuseBridge contract for security.

4. Security Checks:

- Represented by diamond shapes, these are critical checkpoints in the system.
- Include checks for executor status, owner status, transaction status (released/prevented), and fee validity.

### Key Security Features Illustrated:

1. Role-Based Access Control:

- Clear separation of functions accessible by Owner, Executor, and User.
- Owner has exclusive access to system configuration functions.
- Executors have exclusive access to sensitive operational functions.

2. Multi-Step Verification:

- Functions like `release` and `prevent` go through multiple checks (executor status, transaction status).

3. Parameter Validation:

- Fee changes are validated to ensure they're within acceptable limits.

4. Restricted Token Operations:

- Only the FuseBridge contract can mint or burn FuseWrappedTon tokens.

5. State Checks:

- Prevents double-processing of transactions (`release` or `prevent`) through status checks.

### Upgradeability

The contract uses the UUPS (Universal Upgradeable Proxy Standard) pattern:

- The `_authorizeUpgrade` function is overridden to restrict upgrades to the contract owner.
- The `upgradeToAndCall` function (inherited from UUPSUpgradeable) allows for upgrading the contract implementation.

### Fund Management

The contract includes functions for withdrawing both Ether and ERC20 tokens:

- `withdrawEther(uint256 amount, address receiver)`
- `withdrawERC20(address token, uint256 amount, address receiver)`

These functions are restricted to the contract owner and use the `WithdrawFunds` abstract contract for implementation.

### Events and Error Handling

The contract emits several events for important actions:

`BridgeInitiated`: When a bridge operation is started.
`Release`: When tokens are released on the destination chain.
`Prevent`: When a bridge request is prevented.
`SetFuseSwap`, `SetExecutorStatus`, `SetFeeBps`: For administrative actions.

Custom errors are used for better gas efficiency and more informative error messages:

`OnlyFuseBridge`, `OnlyExecutor`, `InvalidFeeBps`, `AlreadyReleased`, `AlreadyPrevented`, `InvalidAddress`
