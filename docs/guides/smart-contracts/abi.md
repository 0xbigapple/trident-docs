# Application Binary Interface

The Application Binary Interface (ABI) is a data encoding scheme for working with smart contracts. The types defined in the ABI are the same as those you encounter when writing smart contracts with Solidity.

## Using ABI in Trident

The `abi` package in Trident includes libraries to encode/decode ABIs. Take calling `function totalSupply() public view returns (uint256)` function as an example to illustrate how to interact with a contract using ABI

```java
// 1. Define function parameters (empty for totalSupply)
List<Type> inputParameters = Collections.emptyList();

// 2. Create function and encode
Function function = new Function(
    "totalSupply",     // Function name
    inputParameters,   // Function input parameters
    Arrays.asList(new TypeReference<Uint256>() {})  // Function output parameters
);
String encodedHex = FunctionEncoder.encode(function);

// 3. Call the contract
TransactionExtention txnExt = client.triggerConstantContract(
    ownerAddress,      // Caller address
    contractAddress,   // Contract address
    encodedHex        // Encoded function call
);

// 4. Decode the result
String result = Numeric.toHexString(txnExt.getConstantResult(0).toByteArray());
BigInteger totalSupply = (BigInteger) FunctionReturnDecoder.decode(
    result, 
    function.getOutputParameters()
).get(0).getValue();

// For tokens with 18 decimals (like JST), the result might be:
// 9900000000000000000000000000
```

## Type Matching

The parameter types in your code must exactly match the function definition in the smart contract. For example:

```solidity
// In smart contract
function transfer(address _to, uint _value) public returns (bool)  // uint is alias for uint256
```

```java
// In Java code - Correct ✓
List<Type> params = Arrays.asList(
    new Address("TRxxxxxxxxxxxxxxxxxxxxxxxxxxx"),  // address type
    new Uint256(1000000)                         // uint256 type
);

// In Java code - Wrong ✗
List<Type> params = Arrays.asList(
    new Utf8String("TRxxxxxxxxxxxxxxxxxxxxxxxxxxx"),  // wrong type for address
    new Int256(1000000)                             // wrong type for uint
);
```

For TRC10 token, use `Uint256` even if the contract parameter is defined as `trcToken`:

```solidity
// In smart contract
function transferToken(address _to, trcToken _id, uint256 _amount) external;
```

```java
// In Java code
List<Type> params = Arrays.asList(
    new Address("TRxxxxxxxxxxxxxxxxxxxxxxxxxxx"),  // address _to
    new Uint256(1000016),                        // trcToken _id
    new Uint256(1000000)                         // uint256 _amount
);
```

Using incorrect types will result in transaction failure or unexpected behavior.
