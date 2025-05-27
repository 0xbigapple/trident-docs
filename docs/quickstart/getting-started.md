# 🚀 Getting Started with Trident

This guide will walk you through the basic operations using the Trident SDK.

## Initialize Client

The `ApiWrapper` in package client is the entrance of the wrapped APIs and smart contract functions. Before using functions in ApiWrapper, you should bind your private key to an ApiWrapper instance:

```java
import org.tron.trident.core.ApiWrapper;

public class QuickStart {
    public static void main(String[] args) {
        // Connect to Nile testnet with tronGrid endpoint
        ApiWrapper client = ApiWrapper.ofNile("your_private_key");
        
        // Or connect to Shasta testnet with tronGrid endpoint
        // ApiWrapper client = ApiWrapper.ofShasta("your_private_key");
        
        // Or connect to mainnet with tronGrid endpoint (requires TronGrid API key)
        // ApiWrapper client = ApiWrapper.ofMainnet("your_private_key", "your_api_key");

        // Initialize with custom RPC endpoints
        // ApiWrapper client = new ApiWrapper(
        //     "grpc.example.com:50051",     // Full node gRPC endpoint
        //     "grpc.example.com:50052",     // Solidity node gRPC endpoint
        //     "your_private_key"
        // );
    }
}
```

!!! note
    For testing purposes, we recommend using the Nile testnet. You can get test tokens from the [Nile Faucet](https://nileex.io/join/getJoinPage).


## Basic Operations

### Get Account Balance

```java

long balance = client.getAccountBalance("your_address"); // balance in SUN (1 TRX = 1,000,000 SUN)
System.out.println("Balance: " + balance / 1_000_000.0 + " TRX");
```

### Get Account with Solidity gRPC endpoint

```java
Account accountSolidity = client.getAccount("your_address", NodeType.SOLIDITY_NODE);
System.out.println(accountSolidity.getAssetCount());
```

### Send TRX

```java
import org.tron.trident.proto.Response.TransactionExtention;
import org.tron.trident.proto.Chain.Transaction;

// Transfer 100 TRX
long amount = 100_000_000L; // Amount in SUN
TransactionExtention txn = client.transfer("from_address", "to_address", amount);

// Sign and broadcast
Transaction signedTxn = client.signTransaction(txn);
String txid = client.broadcastTransaction(signedTxn);
System.out.println("Transaction sent: " + txid);
```

### Transfer TRC20 Token

```java
import org.tron.trident.abi.FunctionEncoder;
import org.tron.trident.abi.TypeReference;
import org.tron.trident.abi.datatypes.Function;
import org.tron.trident.abi.datatypes.generated.Uint256;
import java.math.BigInteger;

// Contract address of the TRC20 token
String contractAddress = "TXYZopYRdj2D9XRtbG411XZZ3kM5VkAeBf"; // USDT contract on Nile testnet

// Create transfer function
Function transfer = new Function(
    "transfer",
    Arrays.asList(
        new Address("recipient_address"),
        new Uint256(BigInteger.valueOf(100_000_000L)) // Amount with decimals (e.g., 100 USDT)
    ),
    Arrays.asList(new TypeReference<Bool>() {})
);

// Encode function call
String encodedHex = FunctionEncoder.encode(transfer);

// Trigger contract
TransactionExtention transactionExtention = client.triggerContract(
        fromAddr,        // Sender Address
        contractAddress, // Contract Address
        encodedHex,      // Encoded function call
        0,              // call value
        0,              // token value
        null,           // token id
        150_000_000L    // fee Limit
);

// Sign and broadcast
Transaction signedTxn = client.signTransaction(transactionExtention);
String txid = client.broadcastTransaction(signedTxn);
System.out.println("Token transfer sent: " + txid);
```

!!! important "Important Considerations"
    When working with TRC20 tokens:
    
    1. Token Decimals
        - Check the token's decimal places (e.g., USDT uses 6 decimals)
        - Adjust the amount accordingly (e.g., 1 USDT = 1_000_000)
    
    2. Transaction Settings
        - Set an appropriate fee limit for contract calls
        - Handle contract revert errors in production code

### Query Transaction

```java
import org.tron.trident.proto.Response.TransactionInfo;

// Get transaction info by ID
TransactionInfo txInfo = client.getTransactionInfoById(txid);
System.out.println("Transaction status: " + txInfo.getResult());
```

## Learn More

- Learn more about [Key Management](../guides/keypair/key-management.md)
- Explore [Transaction Guide](../guides/transactions/transaction-guide.md)
- Check out [Smart Contract Integration](../guides/smart-contracts/overview.md)
- View the complete [API Reference](../javadoc/org/tron/trident/core/ApiWrapper.html)
