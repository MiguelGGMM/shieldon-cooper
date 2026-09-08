# LP-Fee-Funded Buyback — Architecture Notes

The canonical implementation is in `src/`; the old buy-input skim design has been retired.

- `BuybackHook` does not modify trader swap deltas. Its only swap permission is `afterSwap`.
- `RepurchaseTreasury` owns a direct Uniswap v4 PoolManager liquidity position.
- After each swap, the hook asks the treasury to collect fees by poking that position with `liquidityDelta = 0`.
- Both fee currencies are retained by the treasury. Accumulated quote fees fund curve-based project-token buybacks; purchased tokens are burned.
- Principal liquidity is not removed during fee collection.

The position is identified by the configured tick range and salt. It must be created through `RepurchaseTreasury.addLiquidity`; an NFT or position owned by another address cannot be collected by the treasury.

See the root `README.md` for the overview, `DEPLOYMENT.md` for real deployment, `TESTING.md` for tests and local demos, and `DIAGRAM.md` for current sequence diagrams. The Solidity files in this directory import the canonical contracts to avoid maintaining stale duplicate implementations.
