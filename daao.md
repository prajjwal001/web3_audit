# Report

- Audit Platform: **Cantina**
- Protocol: **[Daao](https://cantina.xyz/competitions/bd43bdd1-bc7f-473b-96c0-d35d37f3db33)**

----

| severity  | issue     |
|-----------|-----------|
| H-01    | Lack of slippage protection for swaps in CLPoolRouter::getSwapResult    |
| H-02    | Critical token ratio hardcoded in Daao::finalizeFundraising    |
| H-03    | Lack of access control in CLPoolRouter::uniswapV3SwapCallback allows arbitrary token High Risk transfers   |
| L-01    | Lack of slippage protection in pool creation Daao::finalizeFundraising    |
| L-02    | Lack of Deadline Checks in CLPoolRouter::getSwapResult  |
| NC-01   | Swap Failure Risk- Incorrect Token Approval Logic in CLPoolRouter::_handleApproval   |
| NC-02   | Protocol Admin can forcibly rugpull user funds with Daao::emergencyEscape()   |

## [H-01] Lack of slippage protection for swaps in CLPoolRouter::getSwapResult

### Summary
The `CLPoolRouter.sol` contract executes swaps without allowing users to specify a minimum output amount (`minAmountOut`). This omission can expose users to unfavorable price slippage, potentially resulting in significant losses during volatile market conditions.

### Description
The contract retrieves the swap output amount from the liquidity pool but does not include a mechanism to enforce a minimum received amount. Instead, it only verifies that the delta values (`amount0Delta` or `amount1Delta`) are negative before assigning `outputAmount`.

```solidity
93: uint256 outputAmount;
       if (zeroForOne) {
        require(amount1Delta < 0, "Invalid amount1Delta");
        outputAmount = uint256(-amount1Delta);
    } else {
        require(amount0Delta < 0, "Invalid amount0Delta");
        outputAmount = uint256(-amount0Delta);
    }

    if (outputAmount > 0) {
        uint256 poolBalance = IERC20Minimal(tokenOut).balanceOf(
            address(this)
        );
        require(
            poolBalance >= outputAmount,
            "Insufficient pool balance for output"
        );
        IERC20Minimal(tokenOut).transfer(msg.sender, outputAmount);
    }
```
This implementation lacks a **slippage protection mechanism**, meaning that if the price moves against the trader, they will receive a lower output than expected without any way to prevent the transaction from executing.

### Impact
- **Financial Loss for Traders:** Users can receive significantly fewer tokens than expected due to sudden price movements or malicious front-running attacks.
- **Potential Exploitation:** Malicious actors can manipulate the pool’s pricing using flash loans or sandwich attacks to extract value from unsuspecting traders.
- **User Experience Risk:** Without an explicit `minAmountOut` check, users must rely on `sqrtPriceLimitX96` for protection, which is less precise and does not guarantee minimum output.

### Likelihood
- In decentralized finance (DeFi), price volatility is common, making slippage a frequent issue.
- Many automated trading bots and MEV (Miner Extractable Value) strategies actively look for such vulnerabilities.
- The absence of slippage protection could result in significant trader losses, making this an attractive attack vector.

### Recommendation
Introduce a `minAmountOut` parameter in the swap function to enforce a minimum output requirement and revert the transaction if slippage exceeds the user's tolerance.

```solidity
function getSwapResult(
        address pool,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        uint256 minAmountOut // New parameter to enforce slippage protection
    )
        external
        returns (
            int256 amount0Delta,
            int256 amount1Delta,
            uint160 nextSqrtRatio
        )
    {
        address tokenIn = zeroForOne
            ? ICLPool(pool).token0()
            : ICLPool(pool).token1();
        address tokenOut = zeroForOne
            ? ICLPool(pool).token1()
            : ICLPool(pool).token0();
        uint256 amount = uint256(
            amountSpecified > 0 ? amountSpecified : -amountSpecified
        );

        _handleApproval(tokenIn, amount);

        IERC20Minimal(tokenIn).transferFrom(msg.sender, pool, amount);

        (amount0Delta, amount1Delta) = ICLPool(pool).swap(
            address(this),
            zeroForOne,
            amountSpecified,
            sqrtPriceLimitX96,
            abi.encode(msg.sender)
        );

        (nextSqrtRatio, , , , , ) = ICLPool(pool).slot0();
        uint256 outputAmount;
        if (zeroForOne) {
            require(amount1Delta < 0, "Invalid amount1Delta");
            outputAmount = uint256(-amount1Delta);
        } else {
            require(amount0Delta < 0, "Invalid amount0Delta");
            outputAmount = uint256(-amount0Delta);
        }

        // Enforce slippage protection
        require(outputAmount >= minAmountOut, "Slippage tolerance exceeded");

        if (outputAmount > 0) {
            uint256 poolBalance = IERC20Minimal(tokenOut).balanceOf(
                address(this)
            );
            require(
                poolBalance >= outputAmount,
                "Insufficient pool balance for output"
            );
            IERC20Minimal(tokenOut).transfer(msg.sender, outputAmount);
        }

        emit SwapExecuted(
            pool,
            msg.sender,
            zeroForOne,
            amountSpecified,
            sqrtPriceLimitX96,
            amount0Delta,
            amount1Delta,
            outputAmount
        );
    }
```
----

## [H-02] Critical token ratio hardcoded in Daao::finalizeFundraising

### Summary  
The `finalizeFundraising()` function in the **Daao** contract hardcodes a **4:1 token ratio** for liquidity provision, which may lead to **price misalignment, arbitrage opportunities, and economic inefficiencies**. The function does not account for **market conditions, token decimal differences, or dynamic pricing mechanisms**, leading to potential losses or mispricing of the project's liquidity pool.

### Description  
The contract includes the following hardcoded token ratio assignments:  

```solidity
319: amountToken0ForLP = 4 * amountForLP;
320: amountToken1ForLP = amountForLP;
```
And

```solidity
325: amountToken0ForLP = amountForLP;
326: amountToken1ForLP = 4 * amountForLP;
```
This forces a fixed 4:1 liquidity ratio when adding liquidity, without considering **real-time market prices, fundraising results, or the project's intended economic model**.

### Impact
1. **Market Price Mismatch**  
   - The contract assumes a fixed price for the token, which might not align with real-world supply and demand.  
   - If the market price differs, arbitrage traders can exploit the mispricing, draining liquidity or destabilizing the price.  

2. **Impermanent Loss & Liquidity Inefficiency**  
   - If the project’s actual value deviates from the hardcoded ratio, liquidity providers (LPs) face higher **impermanent loss**.  
   - This can **discourage liquidity providers** from participating, reducing overall pool efficiency.  

3. **Token Decimal Mismatch**  
   - Different tokens have varying decimal places (**WETH = 18 decimals, some tokens = 6 or 8 decimals**).  
   - The current implementation does not normalize these values, potentially **causing liquidity misallocation** or incorrect price calculations.  

4. **Inflexibility in Tokenomics**  
   - The project cannot **adjust the liquidity ratio dynamically** based on fundraising outcomes, market conditions, or governance decisions.  
   - If a more optimal ratio is determined later, **contract upgrades would be required**, increasing operational complexity.  

### Likelihood  
- **Medium Likelihood**: Since liquidity provision occurs **at a critical phase (post-fundraising)**, any mispricing will immediately be **exploited by arbitrageurs**.

### Recommendation  
#### **1. Implement a Dynamic Pricing Mechanism**  
Use an **oracle** (e.g., Chainlink) to fetch the real-time price of the token and adjust liquidity accordingly:  

```solidity
uint256 tokenPrice = priceOracle.getPrice();
amountToken0ForLP = amountForLP * tokenPrice;
amountToken1ForLP = amountForLP;
```

#### **2. Normalize Token Decimals**  
Before assigning values, **ensure decimals are accounted for** to avoid miscalculations:  

```solidity
uint8 token0Decimals = ERC20(token0).decimals();
uint8 token1Decimals = ERC20(token1).decimals();

uint256 adjustedAmountToken0 = amountForLP * (10 ** uint256(18 - token0Decimals));
uint256 adjustedAmountToken1 = amountForLP * (10 ** uint256(18 - token1Decimals));

amountToken0ForLP = adjustedAmountToken0;
amountToken1ForLP = adjustedAmountToken1;
```
----
## [H-03] Lack of access control in CLPoolRouter::uniswapV3SwapCallback allows arbitrary token High Risk transfers

### Summary  
The `uniswapV3SwapCallback` function in `CLPoolRouter.sol` lacks proper validation of `msg.sender`, allowing unauthorized external contracts or EOAs to invoke the function. This vulnerability can be exploited to transfer tokens from an arbitrary sender without their consent, potentially leading to significant financial loss.

### Finding Description  
The function `uniswapV3SwapCallback` is implemented as follows:

```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata data
) external override {
    address sender = abi.decode(data, (address));

    if (amount0Delta > 0) {
        IERC20Minimal(ICLPool(msg.sender).token0()).transferFrom(
            sender,
            msg.sender,
            uint256(amount0Delta)
        );
    } else if (amount1Delta > 0) {
        IERC20Minimal(ICLPool(msg.sender).token1()).transferFrom(
            sender,
            msg.sender,
            uint256(amount1Delta)
        );
    }
}
```

1. **Lack of `msg.sender` verification**:  
   - The function assumes `msg.sender` is a legitimate Uniswap V3 pool contract but does not verify this.  
   - Any contract or externally owned account (EOA) can invoke this function.

2. **Arbitrary token transfer vulnerability**:  
   - If an attacker calls `uniswapV3SwapCallback` with:
     - `msg.sender` as their malicious contract.
     - `data` encoding an arbitrary victim’s address as `sender`.
     - `amount0Delta` or `amount1Delta` as a positive value.
   - This would **force a token transfer from the victim to `msg.sender`**, effectively stealing funds.

### Impact 
- Attackers can **force token transfers from arbitrary users**, leading to **direct financial loss**.  
- Since the callback function **blindly trusts `msg.sender`**, users interacting with this contract could be at severe risk.  
- The impact is critical as funds can be **drained from users who interact with the contract**.

### Likelihood  
- **Public Function Exposure**: `uniswapV3SwapCallback` is `external`, allowing **anyone** to call it.
- **No `msg.sender` Validation**: The lack of validation makes it **trivially exploitable**.
- **Common User Behavior**: Many users **blindly approve tokens for routers**, making them susceptible.

### Recommendation  
To prevent unauthorized access and token theft, **verify that `msg.sender` is a legitimate Uniswap V3 pool contract**. Use the **Uniswap V3 Factory** to enforce this validation.

```solidity
import "@uniswap/v3-core/contracts/interfaces/IUniswapV3Factory.sol";

address public immutable uniswapV3Factory;

constructor(address _factory) {
    uniswapV3Factory = _factory;
}

function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata data
) external override {
    address sender = abi.decode(data, (address));

    // Ensure msg.sender is a valid Uniswap V3 pool
    require(
        IUniswapV3Factory(uniswapV3Factory).getPool(
            ICLPool(msg.sender).token0(),
            ICLPool(msg.sender).token1(),
            3000
        ) == msg.sender,
        "Invalid pool caller"
    );

    if (amount0Delta > 0) {
        IERC20Minimal(ICLPool(msg.sender).token0()).transferFrom(
            sender,
            msg.sender,
            uint256(amount0Delta)
        );
    } else if (amount1Delta > 0) {
        IERC20Minimal(ICLPool(msg.sender).token1()).transferFrom(
            sender,
            msg.sender,
            uint256(amount1Delta)
        );
    }
}
```
----
## [L-01] Lack of slippage protection in pool creation Daao::finalizeFundraising
### Summary
The 'Daao.sol' contains a vulnerability in the 'finalizeFundraising' function, where liquidity is added to Uniswap V3 without slippage protection. This allows potential frontrunning attacks that can result in financial losses for the protocol.

### Description  
The contract constructs `INonfungiblePositionManager.MintParams` with `amountMin0 = 0` and `amountMin1 = 0`, meaning that liquidity will be added regardless of market conditions at execution time. This lack of slippage protection makes it possible for an attacker to manipulate the pool price before minting the position, causing the protocol to suffer from unfavorable liquidity placement.

```solidity
INonfungiblePositionManager.MintParams(
    token0,
    token1,
    TICKING_SPACE,
    initialTick,
    upperTick,
    amountToken0ForLP,
    amountToken1ForLP,
@> 0, // amount0Min (should not be 0)
@> 0, // amount1Min (should not be 0)
    address(this),
    block.timestamp,
    sqrtPriceX96
);

```

### Impact  
- **Frontrunning vulnerability**: Attackers can manipulate the price right before liquidity is added, causing the contract to deposit liquidity at an artificially high or low price, leading to **significant value loss**.  
- **Unfavorable liquidity provision**: The lack of slippage protection means liquidity may be added at an inefficient price range, resulting in **higher impermanent loss** and **ineffective liquidity deployment**.  
- **Potential loss of funds**: If an attacker forces the contract to mint liquidity at an artificially set price, the protocol could end up with mispriced assets, causing **direct financial damage**.

### Likelihood 
- **High likelihood**: Uniswap V3 pools are frequently targeted by frontrunning bots that take advantage of poorly protected liquidity additions.  
- **Zero slippage protection makes this trivial to exploit**: Since `amountMin0` and `amountMin1` are set to zero, a bot can easily push the price out of range, wait for liquidity to be minted, and then revert the price back, profiting at the protocol’s expense.  
- **Automated attack feasibility**: MEV bots on Ethereum and other EVM chains actively scan and exploit such vulnerabilities in real-time, making this a **highly exploitable issue**.

### Recommendation  
#### **Implement Slippage Protection**  
Modify the `amountMin0` and `amountMin1` values to include a **slippage buffer**. This ensures that the transaction will revert if the price moves beyond an acceptable range:
```solidity
uint256 amountMin0 = amountToken0ForLP * 95 / 100; // 5% slippage tolerance
uint256 amountMin1 = amountToken1ForLP * 95 / 100;

INonfungiblePositionManager.MintParams(
    token0,
    token1,
    TICKING_SPACE,
    initialTick,
    upperTick,
    amountToken0ForLP,
    amountToken1ForLP,
    amountMin0,  // Slippage protection applied
    amountMin1,  // Slippage protection applied
    address(this),
    block.timestamp,
    sqrtPriceX96
);
```
This will prevent liquidity from being added if the price moves unfavorably beyond 5%, mitigating frontrunning risks and ensuring safer liquidity provision.

## [L-02] Lack of Deadline Checks in CLPoolRouter::getSwapResult

### Summary  
The `getSwapResult` function in the `CLPoolRouter` does not implement a deadline parameter, Without a deadline, transactions may be executed at significantly different prices than expected, exposing users to price slippage and potential manipulation. This report details the issue, its impact, likelihood, and recommendations for mitigation.

### Finding Description  
The `getSwapResult` function is responsible for executing swaps on a liquidity pool. However, it lacks a deadline parameter, allowing transactions to remain in the mempool indefinitely. This can lead to execution at unfavorable prices due to market volatility.

```solidity
57: function getSwapResult(
    address pool,
    bool zeroForOne,
    int256 amountSpecified,
    uint160 sqrtPriceLimitX96
    // missing deadline parameter
    ) 
    external
    returns (
        int256 amount0Delta,
        int256 amount1Delta,
        uint160 nextSqrtRatio
    ) 
    {
        .......
    }

```
- There is no deadline verification, meaning the swap could execute long after the user submits it.

### Impact
1. **Execution at Unfavorable Prices:**  
   - If the transaction is delayed due to network congestion or low gas fees, it may execute at an **unintended price** when the market conditions have changed.  
   
2. **Front-running and Sandwich Attacks:**  
   - Malicious actors can **monitor pending transactions** and exploit price slippage by manipulating the liquidity pool before execution.  

3. **Mempool Manipulation Risks:**  
   - Transactions without a deadline can remain pending indefinitely, **potentially leading to losses** when finally executed at a different price.  

### Likelihood 

- DEX transactions rely on precise price execution delays can be frequent due to network congestion.  
- MEV (Maximal Extractable Value) attacks are common, and malicious actors frequently exploit mempool transactions.  

### Recommendation  
To mitigate this risk, the function should include a `deadline` parameter and ensure that the transaction is executed before it expires.

```solidity
function getSwapResult(
    address pool,
    bool zeroForOne,
    int256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    uint256 deadline // Added deadline parameter
) 
    external
    returns (
        int256 amount0Delta,
        int256 amount1Delta,
        uint160 nextSqrtRatio
    ) 
{
    require(block.timestamp <= deadline, "Transaction expired"); // Added deadline check
        address tokenIn = zeroForOne
            ? ICLPool(pool).token0()
            : ICLPool(pool).token1();
        address tokenOut = zeroForOne
            ? ICLPool(pool).token1()
            : ICLPool(pool).token0();
        uint256 amount = uint256(
            amountSpecified > 0 ? amountSpecified : -amountSpecified
        );

        _handleApproval(tokenIn, amount);

        IERC20Minimal(tokenIn).transferFrom(msg.sender, pool, amount);

        (amount0Delta, amount1Delta) = ICLPool(pool).swap(
            address(this),
            zeroForOne,
            amountSpecified,
            sqrtPriceLimitX96,
            abi.encode(msg.sender)
        );

        (nextSqrtRatio, , , , , ) = ICLPool(pool).slot0();
        uint256 outputAmount;
        if (zeroForOne) {
            require(amount1Delta < 0, "Invalid amount1Delta");
            outputAmount = uint256(-amount1Delta);
        } else {
            require(amount0Delta < 0, "Invalid amount0Delta");
            outputAmount = uint256(-amount0Delta);
        }

        if (outputAmount > 0) {
            uint256 poolBalance = IERC20Minimal(tokenOut).balanceOf(
                address(this)
            );
            require(
                poolBalance >= outputAmount,
                "Insufficient pool balance for output"
            );
            IERC20Minimal(tokenOut).transfer(msg.sender, outputAmount);
        }

        emit SwapExecuted(
            pool,
            msg.sender,
            zeroForOne,
            amountSpecified,
            sqrtPriceLimitX96,
            amount0Delta,
            amount1Delta,
            outputAmount
        );
}
```
- Added a `deadline` parameter** to the function.  
- Implemented `require(block.timestamp <= deadline, "Transaction expired")` to ensure transactions expire after a set time.

## [NC-01] Swap Failure Risk- Incorrect Token Approval Logic in CLPoolRouter::_handleApproval

### **Summary**  
The `_handleApproval` function in `CLPoolRouter.sol` contains a critical flaw where the router contract attempts to approve its own allowance instead of setting the user's allowance for token transfers. This incorrect implementation prevents successful swaps and leads to unnecessary transaction failures.

### **Description**  
The function `_handleApproval` incorrectly tries to modify the ERC20 token allowance for the router contract (`address(this)`) instead of ensuring that the user (`msg.sender`) has approved the router to spend their tokens. In ERC20, `approve(...)` must be called by the actual token owner, but in this implementation, the contract itself is invoking the function, making the approval ineffective.

```solidity
36: function _handleApproval(address token, uint256 amount) internal {
    IERC20Minimal tokenContract = IERC20Minimal(token);
    uint256 currentAllowance = tokenContract.allowance(msg.sender, address(this));
    
    if (currentAllowance < amount) {
        if (currentAllowance > 0) {
@>          tokenContract.approve(address(this), 0);
        }
        require(
@>          tokenContract.approve(address(this), amount),
            "Approval failed"
        );
        emit ApprovalHandled(token, msg.sender, amount);
    }
}

```
- The contract (`address(this)`) is calling `approve(...)`, which only modifies its own allowance, not the user's allowance for the router.
- This does not allow the router to transfer tokens on behalf of the user, causing transaction failures.

### **Impact**  
Due to this faulty implementation:
1. **Failed Token Transfers:** The router will not be able to execute swaps or transfers because the user’s approval to the router is never actually set.
2. **Unnecessary Transaction Reverts:** If the required approval is not set, any function depending on token transfers will fail, leading to wasted gas costs for the user.
3. **Broken Swap Functionality:** Any swap or liquidity operation relying on this approval mechanism will not execute as expected, affecting the core functionality of the contract.

### **Likelihood**  
- **High Likelihood:** This issue is highly likely to occur since ERC20 tokens strictly require that `approve(...)` be called by the token owner.  
- **No Workarounds:** Users cannot manually fix this unless the contract is updated to handle approvals correctly.
- **Direct Impact on Core Functionality:** Since this contract is likely part of a decentralized exchange or liquidity system, failing approvals means users cannot perform swaps or other token operations.

### **Recommendation**  
To fix this issue, ensure that the user (`msg.sender`) explicitly approves the router contract **before** executing any token transfers. The router should only verify the allowance, rather than attempting to set it.

#### **Fixed Code:**
```solidity
function _handleApproval(address token, uint256 amount) internal {
    IERC20Minimal tokenContract = IERC20Minimal(token);
    uint256 currentAllowance = tokenContract.allowance(msg.sender, address(this));

    // Require that the user has already approved enough tokens to the router
    require(currentAllowance >= amount, "Insufficient allowance from user");

    emit ApprovalHandled(token, msg.sender, amount);
}
```

## [NC-02] Protocol Admin can forcibly rugpull user funds with Daao::emergencyEscape()

### Summary  
The `emergencyEscape()` function in the `Daao.sol` contract allows the `protocolAdmin` to withdraw all Ether held by the contract before the fundraising is finalized. This creates a **severe centralization risk**, as the admin can unilaterally drain contributed funds without community approval.  

### Description  
```solidity  
function emergencyEscape() external {  
    require(msg.sender == protocolAdmin, "must be protocol admin");  
    require(!fundraisingFinalized, "fundraising already finalized");  
    (bool success, ) = protocolAdmin.call{value: address(this).balance}("");  
    require(success, "Transfer failed");  
}  
```  
- The function **allows** `protocolAdmin` to withdraw **all** funds from the contract.  
- No **multi-sig authorization**, **community voting**, or **emergency justification** is required.  
- This enables the protocolAdmin to execute a **forced rug pull**, where user funds are stolen before the fundraising process is finalized.  

### Impact
- **Loss of Funds**: Contributors who have deposited Ether can have their funds drained without recourse.  
- **Severe Centralization Risk**: The contract contradicts decentralization principles, relying solely on the admin’s honesty.  
- **Potential Legal & Reputation Risks**: The ability to withdraw all funds without restrictions could classify this as a high-risk or fraudulent contract.  

### Likelihood 
- **High Likelihood**: Since there are no checks on `protocolAdmin`'s actions, an intentional or unintentional misuse of `emergencyEscape()` can easily occur.  
- **No Safeguards in Place**: The contract lacks time-locked withdrawals, DAO approval, or any security conditions that could prevent abuse.  

### Recommendation  
To mitigate the **centralization risk** and **rug pull potential**, the `emergencyEscape()` function should be **replaced with an emergency refund mechanism** that ensures contributors can retrieve their funds instead of allowing the `protocolAdmin` to drain the contract balance. Also create a onlyDAO modifier so that this function can be triggered when DAO wants to.   
```solidity  
function emergencyRefund() external onlyDAO {  
    require(!fundraisingFinalized, "fundraising already finalized");  

    uint256 balance = address(this).balance;  
    require(balance > 0, "No funds available");  

    for (uint256 i = 0; i < contributors.length; i++) {  
        address payable user = contributors[i];  
        uint256 refundAmount = contributions[user];  

        if (refundAmount > 0) {  
            contributions[user] = 0; // Prevent re-entrancy issues  
            (bool success, ) = user.call{value: refundAmount}("");  
            require(success, "Refund failed");  
        }  
    }  
}
