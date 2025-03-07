# Deploying Smart Contracts

Trident allows you to deploy smart contracts using ABI and bytecode. The deployment process consists of two steps:

## 1. Compile Smart Contract

First, compile your Solidity contract to get the bytecode and ABI. You can use:

### Solidity Compiler
```shell
$ solc <contract>.sol --bin --abi --optimize -o <output-dir>/
```

This generates two files:
- `<contract>.bin`: Contains the bytecode
- `<contract>.abi`: Contains the ABI definition

### Online Tools
You can also use online tools on [TRON IDE](https://www.tronide.io/)

## 2. Deploy Contract

Once you have the bytecode and ABI, deploy the contract using Trident:

```java
// Deploy contract
TransactionExtention txn = client.deployContract(
        "MyContract",                // Contract name
        abiStr,                      // Contract ABI string
        bytecode,                    // Contract bytecode
        null,                        // Constructor parameters (if any)
        100_000_000L,               // Fee limit (100 TRX)
        100L,                        // Consume user resource percent (0-100)
        10_000_000L,                // Origin energy limit
        0L,                         // Call value (amount of TRX to send)
        "",                         // Token ID (for TRC10 token, empty if not used)
        0L                          // Token value (for TRC10 token)
);

// Sign and broadcast
Transaction signedTxn = client.signTransaction(txn);
String txid = client.broadcastTransaction(signedTxn);
```

## 3. Deploy with Constructor Parameters

If your contract has a constructor with parameters:

```solidity
// Sample contract
contract Test {
    uint256 public param;
    
    constructor(uint256 p) {
        param = p;
    }
}
```

You need to provide the constructor parameters:

```java
// Prepare constructor parameters
List<Type<?>> params = Arrays.asList(new Uint256(15));  // Initial value for param

// Deploy with parameters
TransactionExtention txn = client.deployContract(
    "MyContract",                // Contract name
    abiStr,                     // Contract ABI string
    bytecode,                   // Contract bytecode
    params,                     // Constructor parameters
    100_000_000L,              // Fee limit (100 TRX)
    100L,                       // Consume user resource percent (0-100)
    10_000_000L,               // Origin energy limit
    0L,                        // Call value (amount of TRX to send)
    "",                        // Token ID (for TRC10 token)
    0L                         // Token value (for TRC10 token)
);
```

!!! note
    - Make sure not to send TRX (call value) to a non-payable constructor
    - For TRC10 token operations, provide the token ID and value
