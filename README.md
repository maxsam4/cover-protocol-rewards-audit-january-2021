# Cover Protocol rewards audit by Mudit Gupta

## Objective of the audit

The audit's focus was to verify that the smart contract system is secure, resilient, and working according to its specifications. The audit activities can be grouped in the following three categories:

**Security**: Identifying security related issues within each contract and the system of contracts.

**Sound Architecture**: Evaluation of this system's architecture through the lens of established smart contract best practices and general software best practices.

**Code Correctness and Quality**: A full audit of the contract source code.

This audit was based on commit hash `a878059a4e63cd1e96b72d8b577184b54f5fc480` and the fixes were reviewed on commit hash `6ee437a5a0c159303a671ee504391907650ef273` of <https://github.com/CoverProtocol/cover-rewards/>

## Findings

Overall, the code follows high-quality software development standards and best practices.

During the audit, 0 Critical, 3 Major, 3 Minor, and 2 Informational issues were found.

- A critical issue represents something that can be relatively easily exploited and will likely lead to loss of funds.
- A major issue represents something that can result in an unintended behavior of the smart contracts. These issues can also lead to loss of funds but are typically harder to execute than critical issues.
- A minor issue represents an oddity discovered during the audit. These issues are typically situational or hard to exploit.
- An informational issue represents a potential improvement. These issues do not pose any practical security risks.

### Major

#### 1. The `updatePool` function can be used to prevent adding new pools.

##### Description

The `updatePool` function does not check if the pool exists. It sets the `lastUpdatedAt` time to current time regardless. This means, a DOS attack can be mounted by front running `addPoolsAndAllowBonus` transactions and calling `updatePool` for the pools being added. Since the `updatePool` function will set the `lastUpdatedAt` to a non zero value, `addPoolsAndAllowBonus` will revert, thinking the pool already exists.

It is suggested to change L80 `if (block.timestamp <= pool.lastUpdatedAt) return;` to
```
uint256 poolLastUpdated = pool.lastUpdatedAt;
if (poolLastUpdated == 0 || block.timestamp <= pool.lastUpdatedAt) return;
```

##### Status update

This issue has been fixed.

#### 2. An edge case can make it impossible to claim rewards.

##### Description

On L313, `uint256 rewardsWriteoff = rewardsWriteoffsLen == i ? 0 : _user.rewardsWriteoffs[i];` , the check should be <= instead of ==. Since `_updateUserWriteoffs` is called after `_claimRewards`, it's possible that `rewardsWriteoffs` has not been populated yet and `i` becomes greater than `rewardsWriteoffsLen`. The else clause of the ternary operator will be executed in that edge case and `_user.rewardsWriteoffs[i]` will return an index out of bounds error.

##### Status update

This issue has been fixed.

#### 3. Stuck ETH can not be recovered using `collectDust`.

##### Description

The `collectDust` function is meant to allow for withdrawal of stuck ETH by passing 0x00 as the token address. However, due to L246 - `uint256 balance = IERC20(_token).balanceOf(address(this));`, withdrawing eth is not possible. Calling `balanceOf` function on 0x00 will result in an error.

##### Status update

This issue has been fixed.

### Minor

#### 4. Reward tracking breaks down if an LP token is used as a bonus token as well.

##### Description

If an LP token is added as a Bonus token, it will mess up the rewards tracking. This is a low severity issue since this is not the intended use cases and can only be triggered by permissioned users. Nevertheless, It's suggested to add a check in the `addPoolsAndAllowBonus` function to make sure the that the LP token is not an existing bonus token. Also, when adding bonus tokens, it should be checked that they aren't an existing pool token.

##### Status update

This issue has been fixed.

#### 5. A new pool may overwrite `_authorizers` of an old pool.

##### Description

If two pools share a common bonus tokens, they also share a common `_authorizers` list in the current implementation. The contract does not check this and allows the pool that is added last to overwrite the `_authorizers` list of the previous pool. It's recommended to either make `_authorizers` be different for different pools in the contract or add a check to ensure previous `_authorizers` aren't being overwritten.


##### Status update

This issue has been fixed.

#### 6. Considerable amount of GAS can be saved by packing variables.

##### Description

The `startTime` and `endTime` variables can be changed from `uint256` to a smaller type to save storage slots.

##### Status update

This issue has been fixed.

### Informational

#### 7. Typo in README.

##### Description

`npm hardhat compile` should be `npx hardhat compile`.

##### Status update

This issue has been fixed.

#### 8. Minor optimization.

##### Description

`allowedTokenAuthorizers` and `responders` should be mappings instead of arrays to allow O(1) verification instead of O(n).

##### Status update

Acknowledged.

## Disclaimer

This report is not an endorsement or indictment of any particular project or team, and the report do not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. This report does not provide any warranty or representation to any Third-Party in any respect, including regarding the bugfree nature of code, the business model or proprietors of any such business model, and the legal compliance of any such business. No third party should rely on this report in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project. I owe no duty to any Third-Party by virtue of publishing these Reports.

The scope of my audit is limited to a audit of Solidity code and only the Solidity code noted as being within the scope of the audit within this report. The Solidity language itself remains under development and is subject to unknown risks and flaws. The audit does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty.

This audit does not give any warranties on finding all possible security issues of the given smart contracts, i.e., the evaluation result does not guarantee the nonexistence of any further findings of security issues. As one audit cannot be considered comprehensive, I always recommend proceeding with several independent audits and a public bug bounty program to ensure the security of smart contracts.
