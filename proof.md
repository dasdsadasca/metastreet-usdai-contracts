```markdown
# Security Finding Report: CRI-01

## Summary
The `depositBaseYield` function in `BasePositionManager.sol` (inherited by `StakedUSDai.sol`) makes a call to `_usdai.deposit()` with a hardcoded `usdaiAmountMinimum` of `0`. If the `_wrappedMToken` (yield token) being deposited needs to be swapped to `USDai`'s underlying `_baseToken` via the `SwapAdapter`, this zero minimum output allows for sandwich attacks, leading to significant value loss for the `StakedUSDai` vault.

## Finding Description

The `depositBaseYield` function is designed to take yield generated in the form of `_wrappedMToken` (WMT) and convert it into USDai to bolster the assets of the `StakedUSDai` vault. The function calculates an amount of `wrappedMAmount` to process based on a target `usdaiAmount` input by the `STRATEGY_ADMIN_ROLE`.

The vulnerability lies in this specific call within `depositBaseYield`:
```solidity
// In BasePositionManager.sol (inherited by StakedUSDai.sol)
uint256 usdaiAmount_ = _usdai.deposit(address(_wrappedMToken), wrappedMAmount, 0, address(this));
```
The third parameter, `usdaiAmountMinimum`, is hardcoded to `0`.

This call propagates to `USDai.sol#deposit`, and if `_wrappedMToken` is not the same as `USDai`'s `_baseToken` (let's call this M), an internal swap is triggered:
```solidity
// In USDai.sol#_deposit
// ...
usdaiAmount = _scale(_swapAdapter.swapIn(depositToken, depositAmount, _unscaleUp(usdaiAmountMinimum), data));
// ...
```
Since `usdaiAmountMinimum` is `0`, `_unscaleUp(0)` results in `0`. This `0` is then passed as the `minBaseAmount` parameter to `_swapAdapter.swapIn()`.

The `UniswapV3SwapAdapter.sol#swapIn` function takes this `minBaseAmount` (which is `0`) and passes it as `amountOutMinimum` to the Uniswap V3 Router (either `exactInputSingle` or `exactInput`).

A `amountOutMinimum` of `0` in a Uniswap V3 swap instructs the router to accept any amount of output tokens greater than zero, regardless of how unfavorable the price has become due to slippage or market manipulation.

**How it breaks security guarantees:**
This breaks the guarantee that the `StakedUSDai` vault will receive a fair market value when converting its yield tokens (`_wrappedMToken`) into USDai. It allows an external actor (MEV bot) to extract value from this conversion process at the expense of sUSDai holders.

**Propagation of malicious input (MEV bot sandwich attack):**
1.  **Attacker monitors mempool:** Sees a `depositBaseYield` transaction initiated by `STRATEGY_ADMIN_ROLE`.
2.  **Attacker front-runs:**
    *   The attacker calculates the `wrappedMAmount` that will be swapped.
    *   They execute a large swap of WMT for M (or the relevant pair in the `SwapAdapter`) just before the victim's transaction, pushing down the price of WMT relative to M.
3.  **Victim's transaction executes:**
    *   `StakedUSDai.depositBaseYield()` calls `_usdai.deposit(...)` with `usdaiAmountMinimum = 0`.
    *   `_usdai.deposit()` calls `_swapAdapter.swapIn(...)` with `minBaseAmount = 0`.
    *   The `SwapAdapter` executes the WMT to M swap on Uniswap V3. Due to the manipulated pool price and `amountOutMinimum = 0`, the swap executes at a very poor rate, yielding significantly less M than expected.
    *   `USDai.sol` mints a smaller amount of USDai to `StakedUSDai` based on this reduced M amount.
4.  **Attacker back-runs:**
    *   The attacker reverses their initial swap (sells M back for WMT), profiting from the price difference they induced and exploited.

## Impact Explanation
The impact is **Critical**.
This vulnerability allows for direct and quantifiable value loss from the `StakedUSDai` vault each time `depositBaseYield` is called and an internal swap occurs. The amount lost depends on market conditions, liquidity of the involved pair, and the sophistication of the MEV attacker, but can be substantial. This directly reduces the assets backing sUSDai, harming all sUSDai holders by decreasing the value of their shares. Repeated exploitation would consistently drain yield from the protocol.

## Likelihood Explanation
The likelihood is **High**.
Sandwich attacks are a common and well-understood form of MEV in DeFi. Given that `depositBaseYield` is a permissioned function likely called by a known admin/keeper address, MEV bots can specifically target these transactions. The hardcoded `0` for minimum output makes these transactions trivial to exploit for profit whenever the underlying conditions (swap needed, sufficient liquidity for the attacker) are met. The `STRATEGY_ADMIN_ROLE` is expected to call this function periodically to realize yield, making it a recurring opportunity.

## Proof of Concept

A full on-chain PoC would involve setting up the described token (WMT, M, USDai, sUSDai) and pool infrastructure, then executing the sandwich attack. Below is a conceptual PoC outlining the steps an attacker would take, assuming the contracts are deployed and the WMT/M pool exists.

**Setup:**
*   `_wrappedMToken` (WMT) and `USDai._baseToken` (M) are distinct.
*   Uniswap V3 Pool: WMT/M with reasonable liquidity.
*   `StakedUSDai` holds `X` WMT from harvested yield.
*   `STRATEGY_ADMIN_ROLE` decides to call `depositBaseYield(targetUsdai)` which will process `X` WMT.

**Attacker's Steps (simplified):**

```javascript
// Attacker's perspective (conceptual)
// const WMT_ADDRESS = '0x...';
// const M_ADDRESS = '0x...';
// const UNISWAP_ROUTER = '0x...'; // Uniswap V3 Router
// const STAKED_USDAI_ADDRESS = '0x...';

// 1. Monitor mempool for StakedUSDai.depositBaseYield()
//    - Extract `usdaiAmount` parameter if visible, or estimate based on typical calls.
//    - Calculate `wrappedMAmount` that will be swapped (e.g., X WMT).

async function executeSandwich(wmtAmountToSwapByVictim) {
    // 2. Front-run Transaction
    //    - Calculate amount of WMT to dump to manipulate price significantly but still be profitable.
    const attackerWmtToDump = wmtAmountToSwapByVictim * 5; // Example large amount
    //    - Approve Uniswap Router to spend attacker's WMT
    //    - Attacker swaps `attackerWmtToDump` of WMT for M (driving down WMT price relative to M)
    //      const mReceivedFromFrontRun = await uniswapRouter.exactInputSingle({
    //          tokenIn: WMT_ADDRESS,
    //          tokenOut: M_ADDRESS,
    //          fee: poolFee,
    //          recipient: attacker.address,
    //          deadline: now + 300,
    //          amountIn: attackerWmtToDump,
    //          amountOutMinimum: 0, // Attacker doesn't care much about slippage here
    //          sqrtPriceLimitX96: 0
    //      });
    console.log(`Front-run: Swapped ${attackerWmtToDump} WMT for M`);

    // 3. Victim's transaction (StakedUSDai.depositBaseYield) executes now
    //    - It will swap `wmtAmountToSwapByVictim` of WMT for M at the manipulated, worse price.
    //    - It receives fewer M tokens (and thus fewer USDai) than it should have.
    console.log("Victim transaction (depositBaseYield) is expected to execute now...");
    //    (Wait for victim tx to be mined - in a real PoC, this is part of MEV bundle)

    // 4. Back-run Transaction
    //    - Attacker swaps the `mReceivedFromFrontRun` back to WMT.
    //    - Since WMT price is low (and victim's trade might have pushed it lower or kept it there),
    //      attacker gets more WMT than `attackerWmtToDump`.
    //      const wmtReceivedFromBackRun = await uniswapRouter.exactInputSingle({
    //          tokenIn: M_ADDRESS,
    *   // tokenOut: WMT_ADDRESS,
    *   // fee: poolFee,
    *   // recipient: attacker.address,
    *   // deadline: now + 300,
    *   // amountIn: mReceivedFromFrontRun,
    *   // amountOutMinimum: 0,
    *   // sqrtPriceLimitX96: 0
    *   // });
    //    const profit = wmtReceivedFromBackRun - attackerWmtToDump;
    //    console.log(`Back-run: Swapped M back for ${wmtReceivedFromBackRun} WMT. Profit: ${profit} WMT`);

    return profit;
}
```
**Expected Outcome of PoC:**
The `profit` for the attacker would be positive, representing value extracted from the `StakedUSDai` vault during the `depositBaseYield` process. The amount of USDai minted to `StakedUSDai` would be measurably less than if the swap had occurred at the pre-manipulation market price.

## Recommendation
The `depositBaseYield` function in `BasePositionManager.sol` must be modified to ensure that a non-zero, calculated `usdaiAmountMinimum` is passed to the `_usdai.deposit()` call.

**Option 1: Require Admin to Provide Minimum**
The `STRATEGY_ADMIN_ROLE` calling `depositBaseYield` should be responsible for calculating and providing this minimum expected USDai output as a parameter.

```diff
// In BasePositionManager.sol
interface IBasePositionManager {
    // ...
    function depositBaseYield(
        uint256 usdaiAmount,
+       uint256 minUsdaiAmountOut // New parameter
    ) external returns (uint256, uint256);
}

abstract contract BasePositionManager is ... IBasePositionManager ... {
    // ...
    function depositBaseYield(
        uint256 usdaiAmount, // This 'usdaiAmount' is now the 'max WMT to spend to get minUsdaiAmountOut' or similar
+       uint256 minUsdaiAmountOut // Strategy admin provides this
    ) external override onlyRole(STRATEGY_ADMIN_ROLE) nonReentrant returns (uint256, uint256) {
        /* Scale down the USDai amount (max WMT to spend) */
        uint256 wrappedMAmount = _unscale(usdaiAmount);

        if (wrappedMAmount > _wrappedMToken.balanceOf(address(this))) {
            revert InsufficientBalance();
        }

        _wrappedMToken.approve(address(_usdai), wrappedMAmount);

-       uint256 usdaiAmount_ = _usdai.deposit(address(_wrappedMToken), wrappedMAmount, 0, address(this));
+       uint256 usdaiAmount_ = _usdai.deposit(address(_wrappedMToken), wrappedMAmount, minUsdaiAmountOut, address(this));

        // ... rest of the function
    }
}
```
The off-chain caller (strategy admin) would then need to:
1.  Query the current expected output of `wrappedMAmount` of WMT for USDai (e.g., using a static call to `_usdai.deposit` or querying `SwapAdapter` via `eth_call`).
2.  Subtract a small, acceptable slippage percentage from this expected output to determine `minUsdaiAmountOut`.
3.  Call `depositBaseYield(usdaiAmount, minUsdaiAmountOut)`.

**Option 2: On-chain Calculation (More Complex, Gaslier)**
This is generally less preferred for admin functions where off-chain calculation is feasible and safer. An on-chain oracle could be used to get the current WMT/M price to estimate the minimum output, but this introduces oracle risk into this function as well.

**Preferred Solution:** Option 1 is generally safer and more standard for admin-controlled operations involving swaps. The entity calling the function (the `STRATEGY_ADMIN_ROLE`, likely a sophisticated keeper bot or multi-sig) should bear the responsibility of providing up-to-date and correct slippage parameters.
```
