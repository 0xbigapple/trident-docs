# Smart Contracts Overview

## Introduction

TRON smart contracts are programs that run on the TRON Virtual Machine (TVM). The TVM is compatible with the Ethereum Virtual Machine (EVM), allowing developers to deploy and execute Solidity smart contracts.

With Trident you can [deploy contracts](deployment.md) and [interact with them](interaction.md), encode and decode calls through the [Application Binary Interface](abi.md), and follow [worked examples](complex-abi-examples.md) that rebuild real JustLend V2 and SunSwap V4 transactions with ABI v2 structs.

## Best Practices

### Resource Management
- Calculate required energy before deployment
- Set appropriate fee limits for transactions
- Monitor resource consumption

### Security Considerations
- Test contracts thoroughly before deployment
- Use verified contract patterns
- Consider having contracts audited

### Performance Tips
- Optimize contract code for lower energy costs
- Cache frequently accessed data
- Use batch operations when possible

!!! note
    Smart contracts are immutable once deployed. Make sure to thoroughly test your contract before deployment.
