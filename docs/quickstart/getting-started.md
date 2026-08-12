# 🚀 Getting Started with Trident

This guide will walk you through the basic operations using the Trident SDK.

## Initialize Client

The `ApiWrapper` in package client is the entry point of the wrapped APIs and smart contract functions. Before using functions in ApiWrapper, you should bind your private key to an ApiWrapper instance:

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
    }
}
```

`ApiWrapperBuilder` is the recommended way to create a client. Configuration methods are chainable — later calls to the same setting override earlier ones — and the Solidity node endpoint is optional:

```java
import org.tron.trident.core.ApiWrapper;
import org.tron.trident.core.ApiWrapperBuilder;

ApiWrapper client = new ApiWrapperBuilder("grpc.example.com:50051") // Full node gRPC endpoint
    .withGrpcEndpointSolidity("grpc.example.com:50052")             // Optional: Solidity node gRPC endpoint
    .withPrivateKey("your_private_key")
    .withApiKey("your_api_key")     // Optional: TronGrid API key
    .withTimeout(10_000)            // Optional: timeout in milliseconds for all requests
    .withTLS()                      // Optional: enable TLS with system trust certificates
    .build();
```

To connect to a node with a self-signed certificate, pass the trust certificate file: `.withTLS(new File("path/to/cert.pem"))`. Custom gRPC interceptors can be added with `.addInterceptors(...)`.

!!! note "Deprecated constructors"
    The `ApiWrapper` constructors (e.g. `new ApiWrapper(grpcEndpoint, grpcEndpointSolidity, privateKey)`) are deprecated since 1.0.0. They are kept for compatibility, but new code should use `ApiWrapperBuilder`.

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

## Scenario Examples

### Build a Multi-sign Transaction

Developers can send multi-sign transactions easily using Trident. Here is as an example of how to create a transfer transaction using account active permissions.

The steps below illustrate a complete multi-sign transaction process:

1. Modify account permission (make a multi-sign account, requires 100 TRX)

2. Select permission and create transfer transaction

3. Sign transaction with permission accounts

4. Broadcast transaction

```

package org.tron.trident.core;

import static java.lang.Thread.sleep;

import com.google.protobuf.ByteString;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import org.tron.trident.core.account.AccountPermissions;
import org.tron.trident.core.account.ActivePermissionOperationsUtils;
import org.tron.trident.core.exceptions.IllegalException;
import org.tron.trident.core.key.KeyPair;
import org.tron.trident.core.transaction.TransactionBuilder;
import org.tron.trident.proto.Chain.Transaction;
import org.tron.trident.proto.Chain.Transaction.Contract.ContractType;
import org.tron.trident.proto.Common.Permission;
import org.tron.trident.proto.Response.TransactionExtention;

/**
 * A demo for TRON multi-sign transactions.
 *
 * This demo illustrates the following steps:
 * 1. Modify an account's permissions to require multiple signatures for certain operations.
 * 2. Create a transaction that requires multi-signature.
 * 3. Sign the transaction with the required keys.
 * 4. Broadcast the multi-signed transaction.
 */
public class MultiSignDemo {

  // NOTE: Replace with your private key. In a real application,
  // use a secure way to manage private keys, such as environment variables or a secret manager.
  private static final String OWNER_PRIVATE_KEY = "...";
  // The account that will have its permissions updated for multi-sign.
  private static final String OWNER_ADDRESS = new KeyPair(OWNER_PRIVATE_KEY).toBase58CheckAddress();

  // The recipient address for the transfer.
  private static final String TO_ADDRESS = "T...";

  public static void main(String[] args) throws IllegalException, InterruptedException {
    // Initialize the API wrapper to connect to the Nile testnet.
    ApiWrapper client = ApiWrapper.ofNile(OWNER_PRIVATE_KEY);

    // Generate 3 new key pairs to be used as active permissions.
    List<KeyPair> activeAddressPairs = new ArrayList<>();
    for (int i = 0; i < 3; i++) {
      activeAddressPairs.add(KeyPair.generate());
    }
    System.out.println("Generated active addresses:");
    System.out.println("1. " + activeAddressPairs.get(0).toBase58CheckAddress());
    System.out.println("2. " + activeAddressPairs.get(1).toBase58CheckAddress());
    System.out.println("3. " + activeAddressPairs.get(2).toBase58CheckAddress());


    //======= STEP 1: Modify Account Permission to enable Multi-signature =======

    System.out.println("\n======= STEP 1: Modifying account permissions... =======");

    // Get the existing permissions of the owner account.
    AccountPermissions accountPermissions = client.getAccountPermissions(OWNER_ADDRESS);

    // Create a map of the new active keys and their weights. For this demo, all keys have a weight of 1.
    Map<String, Long> activeKeyMap = new HashMap<>();
    for (KeyPair keyPair : activeAddressPairs) {
      activeKeyMap.put(keyPair.toBase58CheckAddress(), 1L);
    }

    // Define the operations that this active permission will control.
    // Here, we restrict it to only TransferContract (TRX transfers).
    ByteString trxTransferOperations = ActivePermissionOperationsUtils.buildOperations(
        ByteString.EMPTY, true, ContractType.TransferContract);

    // Create a new active permission.
    // - "active": A custom name for the permission.
    // - permissionId=2: The ID for this new permission. ID 0 is for owner, 1 is for witness, 2 is for active.
    // - threshold=2: The sum of weights of signatures required to approve a transaction (2 out of 3 in this case).
    Permission activePermission = accountPermissions.createActivePermission("active", 2,
        2, trxTransferOperations, activeKeyMap);

    List<Permission> activePermissions = new ArrayList<>();
    activePermissions.add(activePermission);

    // Set the new active permissions for the account. The owner permission remains unchanged.
    accountPermissions.setActivePermission(activePermissions);

    // Create the transaction to update the account permissions on the blockchain.
    TransactionExtention txnExt = client.accountPermissionUpdate(
        OWNER_ADDRESS,
        accountPermissions);

    // The permission update transaction must be signed by the owner key.
    Transaction signedTxn = client.signTransaction(txnExt);
    String updateTxId = client.broadcastTransaction(signedTxn);

    System.out.println("Account permission update transaction sent. TXID: " + updateTxId);
    // Wait for the transaction to be confirmed on the blockchain.
    System.out.println("Waiting for confirmation...");
    sleep(10_000L);


    // ======== STEP 2: Create a transaction using the new permission ========

    System.out.println("\n======= STEP 2: Creating a multi-signature transfer... =======");

    // Create a standard transfer transaction of 1 TRX (1,000,000 SUN).
    TransactionExtention transferTxnExt = client.transfer(OWNER_ADDRESS, TO_ADDRESS, 1_000_000);

    // Set the permission ID on the transaction to '2'.
    // This tells the blockchain that this transaction must be signed by keys
    // associated with the active permission we created in STEP 1.
    TransactionBuilder transactionBuilder = new TransactionBuilder(transferTxnExt.getTransaction());
    Transaction transferTransaction = transactionBuilder.setContractPermissionId(2).build();


    // ======== STEP 3: Sign the transaction with multiple keys ========

    System.out.println("\n======= STEP 3: Signing the transaction with active keys... =======");

    // The transaction is signed sequentially by the active keys.
    // Since the threshold is 2, we need signatures from any 2 of the 3 active keys.

    // First active account signs.
    System.out.println("Signing with key 1...");
    Transaction signedTxn1 = client.signTransaction(transferTransaction, activeAddressPairs.get(0));

    /******************************************************************************************
     * NOTE: Handling multi-signature signing across different parties.
     *
     * In a real-world scenario, signatures come from different users at different locations.
     * The partially signed `Transaction` object (`signedTxn1`) could be serialized and
     * sent to the next signer.
     *
     * 1. First signer serializes the transaction:
     *    byte[] transactionBytes = signedTxn1.toByteArray();
     *    // To transport these bytes via a text-based medium (like a JSON API or email),
     *    // you can encode them. Base64 is a common choice for this.
     *    String base64EncodedTx = java.util.Base64.getEncoder().encodeToString(transactionBytes);
     *
     * 2. The encoded string is sent to the next signer.
     *
     * 3. Next signer decodes and signs:
     *    byte[] decodedBytes = java.util.Base64.getDecoder().decode(base64EncodedTx);
     *    Transaction partiallySignedTx = Transaction.parseFrom(decodedBytes);
     *    Transaction signedTxn2 = client.signTransaction(partiallySignedTx, nextKeyPair);
     *
     * For this demo, we simulate this by directly passing the `signedTxn1` object.
     ******************************************************************************************/

    // Second active account signs the already partially signed transaction.
    System.out.println("Signing with key 2...");
    Transaction signedTxn2 = client.signTransaction(signedTxn1, activeAddressPairs.get(1));


    // ======== STEP 4: Broadcast the multi-signed transaction ========

    System.out.println("\n======= STEP 4: Broadcasting the transaction... =======");
    String transferTxId = client.broadcastTransaction(signedTxn2);

    System.out.println("Multi-signed transfer transaction sent. TXID: " + transferTxId);
    // Wait for the transaction to be confirmed.
    System.out.println("Waiting for confirmation...");
    sleep(10_000L);

    System.out.println("\nDemo finished successfully!");
  }
}


```




## Learn More

- Learn more about [Key Management](../guides/keypair/key-management.md)
- Explore [Transaction Guide](../guides/transactions/transaction-guide.md)
- Check out [Smart Contract Integration](../guides/smart-contracts/overview.md)
- View the complete [API Reference](../javadoc/org/tron/trident/core/ApiWrapper.html)
