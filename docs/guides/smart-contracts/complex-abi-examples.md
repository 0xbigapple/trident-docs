# Complex ABI Examples

Trident supports ABI v2 structs and nested arrays, which makes it possible to work with real-world DeFi protocols whose interfaces are built around `tuple` parameters and deeply nested calldata.

This page walks through two real TRON mainnet transactions:

- **JustLend V2 — the decode direction**: parse the calldata of a lending transaction into a typed struct, and verify it against the on-chain event.
- **SunSwap V4 — the encode direction**: construct the complete calldata of a swap from literal values, using nested structs, and reproduce a real transaction byte-for-byte.

Both examples are self-contained and runnable — they work on raw calldata bytes and do not require a node connection.

## JustLend V2: Decoding Calldata into Structs

JustLend V2 (Moolah) identifies each lending market by a `MarketParams` struct:

```solidity
struct MarketParams {
    address loanToken;
    address collateralToken;
    address oracle;
    address irm;
    uint256 lltv;
}

function supplyCollateral(MarketParams memory marketParams, uint256 assets,
    address onBehalf, bytes memory data) external;
```

The market id used in events and storage is `keccak256(abi.encode(marketParams))`. With struct support, Trident can decode the calldata into a typed `MarketParams`, compute the market id locally, and rebuild the original bytes — all without touching the chain.

The calldata below is taken from mainnet transaction [`6f3d2dbf...207fa0d5`](https://tronscan.org/#/transaction/6f3d2dbfea4cfb30a52660f6d8903c2282fcd371bfd67e5e94198282207fa0d5), which supplies 4,375 USDT as collateral to the WTRX-borrow/USDT-collateral market:

```java
import java.math.BigInteger;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import org.tron.trident.abi.FunctionEncoder;
import org.tron.trident.abi.FunctionReturnDecoder;
import org.tron.trident.abi.TypeEncoder;
import org.tron.trident.abi.TypeReference;
import org.tron.trident.abi.Utils;
import org.tron.trident.abi.datatypes.Address;
import org.tron.trident.abi.datatypes.DynamicBytes;
import org.tron.trident.abi.datatypes.Function;
import org.tron.trident.abi.datatypes.StaticStruct;
import org.tron.trident.abi.datatypes.Type;
import org.tron.trident.abi.datatypes.generated.Uint256;
import org.tron.trident.crypto.Hash;

public class JustLendDecodeDemo {

  /** All five fields are static types, so the struct extends StaticStruct. */
  public static class MarketParams extends StaticStruct {
    public MarketParams(Address loanToken, Address collateralToken,
        Address oracle, Address irm, Uint256 lltv) {
      super(loanToken, collateralToken, oracle, irm, lltv);
    }
  }

  // Raw calldata of the mainnet transaction: supplyCollateral(MarketParams,uint256,address,bytes)
  static final String CALLDATA =
      "238d6579"                                                             // selector
          + "000000000000000000000000891cdb91d149f23b1a45d9c5ca78a88d0cb44c18"   // loanToken (WTRX)
          + "000000000000000000000000a614f803b6fd780986a42c78ec9c7f77e6ded13c"   // collateralToken (USDT)
          + "000000000000000000000000c8274e6459a06655c363e7ff4314985522472881"   // oracle
          + "000000000000000000000000b979d1612757f74cad2cc7133f1cd5201354f270"   // irm
          + "0000000000000000000000000000000000000000000000000b1a2bc2ec500000"   // lltv (80%, 1e18)
          + "0000000000000000000000000000000000000000000000000000000104c533c0"   // assets
          + "000000000000000000000000eac97cb8af68589f7429c925c6c8eb834597ed03"   // onBehalf
          + "0000000000000000000000000000000000000000000000000000000000000100"   // offset of `data`
          + "0000000000000000000000000000000000000000000000000000000000000000";  // `data` length: 0

  public static void main(String[] args) {
    // 1. Decode the calldata (after the 4-byte selector) into typed values.
    //    The struct is referenced like any other type, and the decoder
    //    instantiates it through the constructor defined above.
    List<Type> decoded = FunctionReturnDecoder.decode(
        CALLDATA.substring(8),
        Utils.convert(Arrays.asList(
            new TypeReference<MarketParams>() {},
            new TypeReference<Uint256>() {},
            new TypeReference<Address>() {},
            new TypeReference<DynamicBytes>() {})));

    MarketParams market = (MarketParams) decoded.get(0);
    BigInteger assets = (BigInteger) decoded.get(1).getValue();
    Address onBehalf = (Address) decoded.get(2);
    System.out.println("supply " + assets + " collateral on behalf of " + onBehalf);

    // 2. Compute the market id locally: keccak256(abi.encode(marketParams)).
    //    It equals topic1 of the SupplyCollateral event emitted by this transaction.
    String marketId = Hash.sha3("0x" + TypeEncoder.encode(market));
    System.out.println("market id: " + marketId);
    // 0x7ad8b9c73e6b4eafd2aebdd9e41da067d8fb2b569f63e60064bff32c0b19d853

    // 3. Rebuild the call from the decoded values — byte-for-byte identical
    //    to the original calldata, including the empty `bytes` tail layout.
    String rebuilt = FunctionEncoder.encode(new Function(
        "supplyCollateral",
        Arrays.asList(market, new Uint256(assets), onBehalf, new DynamicBytes(new byte[0])),
        Collections.<TypeReference<?>>emptyList()));
    System.out.println(rebuilt.equals(CALLDATA));   // true
  }
}
```

The same pattern applies to every JustLend V2 entry point (`supply`, `borrow`, `repay`, `withdrawCollateral`, `liquidate`, ...) — they all take the `MarketParams` struct as their first parameter.

## SunSwap V4: Building Nested Calldata from Literal Values

SunSwap V4 uses a singleton architecture in which a swap request is layered: the outer `UniversalRouter.execute(bytes commands, bytes[] inputs, uint256 deadline)` call carries a `V4_SWAP` command whose input is `abi.encode(bytes actions, bytes[] params)`, and the swap action's parameter is a *dynamic struct nesting a static struct*:

```solidity
struct PoolKey {                      // static struct
    address currency0;
    address currency1;
    address hooks;
    uint24 fee;
    bytes32 parameters;               // tickSpacing packed in bits [16, 40)
}

struct CLSwapExactInputSingleParams { // dynamic struct (contains `bytes`)
    PoolKey poolKey;
    bool zeroForOne;
    uint128 amountIn;
    uint128 amountOutMinimum;
    bytes hookData;
}
```

The example below constructs the complete calldata of a real swap — 7,200 USDT traded for 21,879.86117 TRX through the TRX/USDT 0.05% pool — purely from literal values. The result is byte-for-byte identical to the calldata of mainnet transaction [`e0673ddd...a45ea693`](https://tronscan.org/#/transaction/e0673ddd36ff5cdf6d481dccb466f2dd2077d1eb8db9983526ab3c00a45ea693); to send such a swap yourself, build the calldata like this, then sign and broadcast it with `ApiWrapper.triggerContract`.

```java
import java.math.BigInteger;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import org.tron.trident.abi.FunctionEncoder;
import org.tron.trident.abi.TypeReference;
import org.tron.trident.abi.datatypes.Address;
import org.tron.trident.abi.datatypes.Bool;
import org.tron.trident.abi.datatypes.DynamicArray;
import org.tron.trident.abi.datatypes.DynamicBytes;
import org.tron.trident.abi.datatypes.DynamicStruct;
import org.tron.trident.abi.datatypes.Function;
import org.tron.trident.abi.datatypes.StaticStruct;
import org.tron.trident.abi.datatypes.Type;
import org.tron.trident.abi.datatypes.generated.Bytes32;
import org.tron.trident.abi.datatypes.generated.Uint128;
import org.tron.trident.abi.datatypes.generated.Uint24;
import org.tron.trident.abi.datatypes.generated.Uint256;
import org.tron.trident.utils.Numeric;

public class SunSwapV4EncodeDemo {

  public static class PoolKey extends StaticStruct {
    public PoolKey(Address currency0, Address currency1, Address hooks,
        Uint24 fee, Bytes32 parameters) {
      super(currency0, currency1, hooks, fee, parameters);
    }
  }

  /** A dynamic struct (it contains `bytes hookData`) nesting the static PoolKey. */
  public static class ExactInSingleParams extends DynamicStruct {
    public ExactInSingleParams(PoolKey poolKey, Bool zeroForOne, Uint128 amountIn,
        Uint128 amountOutMinimum, DynamicBytes hookData) {
      super(poolKey, zeroForOne, amountIn, amountOutMinimum, hookData);
    }
  }

  // V4 action codes (periphery Actions.sol) and the UniversalRouter V4_SWAP command
  static final byte SETTLE = 0x0b;
  static final byte CL_SWAP_EXACT_IN_SINGLE = 0x06;
  static final byte TAKE = 0x0e;
  static final byte CMD_V4_SWAP = 0x12;

  static final String USDT = "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t";

  public static void main(String[] args) {
    // The pool: TRX/USDT, fee 0.05%. Native TRX is address(0);
    // tickSpacing=10 is packed into bits [16, 40) of `parameters`.
    PoolKey key = new PoolKey(
        Address.DEFAULT, new Address(USDT), Address.DEFAULT,
        new Uint24(500),
        new Bytes32(Numeric.hexStringToByteArray(
            "00000000000000000000000000000000000000000000000000000000000a0000")));

    // Action sequence: SETTLE (pay USDT in) → swap → TAKE (send TRX to the sender)
    List<DynamicBytes> params = Arrays.asList(
        // SETTLE(currency, amount, payerIsUser)
        encoded(new Address(USDT), new Uint256(7_200_000_000L), new Bool(true)),
        // CL_SWAP_EXACT_IN_SINGLE(ExactInSingleParams)
        encoded(new ExactInSingleParams(key,
            new Bool(false),                  // false: swap currency1 → currency0 (USDT → TRX)
            new Uint128(7_200_000_000L),      // amountIn: 7,200 USDT
            new Uint128(21_771_006_139L),     // amountOutMinimum: 21,771.006139 TRX
            new DynamicBytes(new byte[0]))),
        // TAKE(currency, recipient, amount): address(1) = MSG_SENDER, 0 = OPEN_DELTA
        encoded(Address.DEFAULT, new Address(BigInteger.ONE), new Uint256(0)));

    // The V4_SWAP input is abi.encode(bytes actions, bytes[] params),
    // where each byte of `actions` pairs with one element of `params`
    DynamicBytes input = encoded(
        new DynamicBytes(new byte[] {SETTLE, CL_SWAP_EXACT_IN_SINGLE, TAKE}),
        new DynamicArray<>(DynamicBytes.class, params));

    // Outermost layer: UniversalRouter.execute(bytes commands, bytes[] inputs, uint256 deadline)
    String calldata = FunctionEncoder.encode(new Function("execute",
        Arrays.asList(
            new DynamicBytes(new byte[] {CMD_V4_SWAP}),
            new DynamicArray<>(DynamicBytes.class, Collections.singletonList(input)),
            new Uint256(1784773862L)),
        Collections.<TypeReference<?>>emptyList()));

    // Byte-for-byte identical to the calldata of the mainnet transaction
    System.out.println(calldata);
  }

  /** abi.encode the given values and wrap the result as a `bytes` parameter. */
  static DynamicBytes encoded(Type... values) {
    return new DynamicBytes(Numeric.hexStringToByteArray(
        FunctionEncoder.encodeConstructor(Arrays.asList(values))));
  }
}
```

Decoding works the same way in reverse: reference `ExactInSingleParams` with `new TypeReference<ExactInSingleParams>() {}` and `FunctionReturnDecoder.decode` reconstructs the whole nested structure — including the inner `PoolKey` — in one call.

!!! note "Verifying against the chain"
    Both transactions above emit events keyed by `keccak256(abi.encode(struct))` identifiers (JustLend market ids, SunSwap V4 pool ids). Recomputing these hashes locally from the decoded structs, as shown in the JustLend example, is a handy way to confirm that your Java struct definition matches the on-chain layout exactly.
