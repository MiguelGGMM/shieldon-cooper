# System Diagrams

These diagrams describe the implemented flow. The hook never returns a swap delta and never takes part of a trader's input or output.

## 1. Architecture and fund ownership

```mermaid
flowchart LR
    Trader[Trader / Router]
    Pool[Uniswap v4 PoolManager]
    Hook[BuybackHook<br/>AFTER_SWAP only]
    Treasury[RepurchaseTreasury<br/>direct LP position owner]
    Keeper[Permissionless keeper]
    Dead[Burn address<br/>0x...dEaD]

    Trader -->|normal swaps<br/>no hook tax| Pool
    Pool -->|afterSwap callback| Hook
    Hook -->|collectFees + price/sell reports| Treasury
    Treasury -->|liquidityDelta = 0<br/>position ticks + salt| Pool
    Pool -->|accrued currency0/currency1 LP fees| Treasury
    Keeper -->|executeBuyback| Treasury
    Treasury -->|swap accumulated quote fees| Pool
    Pool -->|purchased project tokens| Treasury
    Treasury -->|project tokens| Dead
```

### Ownership boundaries

```mermaid
flowchart TB
    Position[Direct v4 position]
    Position --> Owner[Treasury is position owner]
    Position --> Identity[Identity = poolId + treasury + tickLower + tickUpper + salt]
    Position --> Principal[Principal liquidity remains in pool]
    Position --> Fees[Only accrued LP fees are collected]

    Other[PositionManager NFT or another account's position]
    Other --> NoAccess[Treasury cannot collect its fees]
```

## 2. Deployment and initialization

```mermaid
sequenceDiagram
    actor Deployer
    participant Treasury as RepurchaseTreasury
    participant Miner as HookMiner
    participant Hook as BuybackHook
    participant Pool as Uniswap v4 PoolManager

    Deployer->>Treasury: deploy with temporary hooks = address(0)
    Deployer->>Miner: find CREATE2 salt for AFTER_SWAP flag
    Miner-->>Deployer: predicted hook address + salt
    Deployer->>Hook: deploy CREATE2(manager, treasury, token ordering)
    Deployer->>Treasury: setHook(hook)
    Deployer->>Treasury: setPoolHook(hook)
    Note over Treasury,Hook: Treasury PoolKey now contains exact deployed hook
    Deployer->>Pool: initialize(PoolKey, initialRawSqrtPriceX96)
    Note right of Pool: This must be a new pool because hook is part of PoolKey
```

## 3. Treasury-owned liquidity setup

```mermaid
sequenceDiagram
    actor Owner
    participant Token0 as currency0 token
    participant Token1 as currency1 token
    participant Treasury as RepurchaseTreasury
    participant Pool as Uniswap v4 PoolManager

    opt ERC-20 currency0
        Owner->>Token0: approve(Treasury, amount0Max)
    end
    opt ERC-20 currency1
        Owner->>Token1: approve(Treasury, amount1Max)
    end
    opt Native currency
        Owner->>Treasury: prefund native currency
    end

    Owner->>Treasury: addLiquidity(liquidity, amount0Max, amount1Max)
    Treasury->>Pool: unlock(action = ADD_LIQUIDITY)
    Pool->>Treasury: unlockCallback(data)
    Treasury->>Pool: modifyLiquidity(+liquidity, configured ticks + salt)
    Pool-->>Treasury: negative deltas = amounts owed
    Treasury->>Treasury: require owed0 <= max0 and owed1 <= max1
    Treasury->>Pool: settle currency0 and currency1 from Owner / native balance
    Treasury-->>Owner: emit LiquidityAdded
    Note over Treasury,Pool: Position owner recorded by PoolManager is Treasury
```

## 4. Trader swap and automatic fee collection

```mermaid
sequenceDiagram
    actor Trader
    participant Router
    participant Pool as Uniswap v4 PoolManager
    participant Hook as BuybackHook
    participant Treasury as RepurchaseTreasury

    Trader->>Router: exact-input swap of 100 quote
    Router->>Pool: swap(amountSpecified = -100)
    Note right of Pool: Full 100 is processed and the standard pool LP fee applies
    Pool->>Pool: update price, fee growth, and swap balances
    Pool->>Hook: afterSwap(key, params, delta)
    Note right of Hook: No beforeSwap permission<br/>No before/after return-delta permission

    Hook->>Treasury: collectFees(key)
    Treasury->>Treasury: require caller == Hook and PoolId matches
    Treasury->>Pool: modifyLiquidity(liquidityDelta = 0)
    Pool-->>Treasury: positive deltas = accrued LP fees
    Treasury->>Pool: take(currency0 fee, Treasury)
    Treasury->>Pool: take(currency1 fee, Treasury)
    Treasury->>Treasury: tvlAth = max(tvlAth, quote balance)
    Treasury-->>Hook: emit FeesCollected

    Hook->>Pool: getSlot0(poolId)
    Pool-->>Hook: raw sqrtPriceX96
    Hook->>Hook: normalize project price for currency ordering
    Hook->>Treasury: updatePriceAth(normalizedProjectPriceX96)

    alt swap input is project token (sell)
        Hook->>Treasury: notifySell()
        Treasury->>Treasury: lastSellAt = block.timestamp
    else swap input is quote token (buy)
        Note over Hook,Treasury: No sell notification
    end

    Hook-->>Pool: AFTER_SWAP selector, delta = 0
    Pool-->>Router: original swap delta
    Router-->>Trader: normal swap output
    Note over Trader,Treasury: Trader is not taxed by the hook<br/>Treasury revenue is the position's standard LP fee share
```

## 5. Currency ordering and normalized project price

Raw Uniswap price is always currency1/currency0. The strategy normalizes it so a larger value consistently means a higher project-token price.

```mermaid
flowchart TD
    Raw[Pool raw sqrtPriceX96] --> Order{Project token is currency0?}
    Order -->|yes| Direct[normalizedProjectPriceX96 = rawSqrtPriceX96]
    Order -->|no| Invert[normalizedProjectPriceX96 = 2^192 / rawSqrtPriceX96]
    Direct --> ATH[update priceAth if normalized price is higher]
    Invert --> ATH
```

Swap direction classification:

| Project-token position | Buy: quote → project | Sell: project → quote |
|---|---|---|
| Project is currency0 | `zeroForOne = false` | `zeroForOne = true` |
| Project is currency1 | `zeroForOne = true` | `zeroForOne = false` |

## 6. Buyback decision tree

```mermaid
flowchart TB
    A[executeBuyback normalizedCurrentPrice] --> A1{current price > 0?}
    A1 -->|no| Z0[revert InvalidPrice]
    A1 -->|yes| B{cooldown elapsed?}
    B -->|no| Z1[revert CooldownActive]
    B -->|yes| C{project sell since last buyback?}
    C -->|no| Z2[revert NoRecentSell]
    C -->|yes| D{current price < priceAth?}
    D -->|no| Z3[revert NoDrawdown]
    D -->|yes| E[drawdownBps = ATH-current / ATH]
    E --> F[fractionBps = cubic curve]
    F --> G[target = tvlAth × fractionBps / 10,000]
    G --> H{target >= 0.25% of tvlAth?}
    H -->|no| Z4[revert MinRepurchaseNotMet]
    H -->|yes| I{quote balance >= minimum?}
    I -->|no| Z5[revert NoFunds]
    I -->|yes| J[spend = min target, quote balance]
    J --> K[derive or accept raw sqrtPriceLimitX96]
    K --> L[swap quote → project]
    L --> M[send received project tokens to burn address]
    M --> N[emit BuybackExecuted]
```

## 7. Buyback transaction

```mermaid
sequenceDiagram
    actor Keeper
    participant Treasury as RepurchaseTreasury
    participant Pool as Uniswap v4 PoolManager
    participant Hook as BuybackHook
    participant Dead as 0x...dEaD

    Keeper->>Treasury: executeBuyback(normalizedCurrentPrice[, rawLimit])
    Treasury->>Treasury: validate price, sell gate, cooldown, curve, balance
    Treasury->>Treasury: lastBuybackAt = block.timestamp
    Treasury->>Pool: unlock(action = BUYBACK)
    Pool->>Treasury: unlockCallback(data)
    Treasury->>Pool: swap exact quote input for project
    Pool->>Hook: afterSwap()
    Hook->>Treasury: collectFees(key)
    Hook->>Treasury: updatePriceAth(normalized price)
    Note over Hook,Treasury: Treasury buyback is a project buy,<br/>so it does not call notifySell
    Treasury->>Pool: settle quote owed
    Treasury->>Pool: take project received, recipient = Dead
    Pool-->>Treasury: unlock result
    Treasury-->>Keeper: emit BuybackExecuted
```

## 8. Treasury balances after activity

```mermaid
flowchart LR
    QuoteFees[Quote-token LP fees] --> QuoteBalance[Treasury quote balance]
    QuoteBalance --> Curve[Available for curve buybacks]
    Curve --> Swap[Quote → project swap]
    Swap --> Burn[Purchased project tokens burned]

    ProjectFees[Project-token LP fees] --> ProjectBalance[Treasury project balance]
    ProjectBalance --> Held[Remain held by treasury]

    Note[Principal LP liquidity] --> Pool[Remains in the pool position]
```

## 9. Current production warning

```mermaid
flowchart LR
    Spot[Raw pool spot price] --> Normalize[Currency-order normalization]
    Normalize --> ATH[ATH tracking]
    Caller[Caller-supplied current price] --> Buyback[Buyback sizing]
    ATH --> Buyback
    Attack[Flash manipulation / stale input] -.risk.-> Spot
    Attack -.risk.-> Caller
    Fix[Required before production:<br/>TWAP or robust oracle] --> ATH
    Fix --> Buyback
```
