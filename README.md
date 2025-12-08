# 🧩 TakeProfitsHook — Automated Take-Profit Orders for Uniswap V4

**TakeProfitsHook** is a Uniswap V4 Hook that allows users to place **automated take-profit orders** directly inside a Uniswap V4 pool.
Instead of monitoring prices manually, users can:

* Place limit-style take-profit orders at specific ticks
* Receive ERC1155 claim tokens representing their order position
* Automatically get filled once price crosses their target tick
* Redeem output tokens after execution

The hook batches multiple orders at the same tick into a single swap, saving gas and simplifying execution.

---

# ✨ Features

### ✅ **Automated Take-Profits**

Users specify:

* the pool,
* the tick to execute at,
* the direction (sell token0 or sell token1),
* the input amount.

The hook executes the swap *only* when the pool’s tick crosses the selected target.

### ✅ **ERC1155 Claim Tokens**

Each order mints an ERC1155 token:

* token ID = hash(poolId, tick, direction)
* supply = total order size at that tick
* holders redeem proportional output after execution

### ✅ **Batch Execution**

If multiple wallets set orders at the same tick:

* only one swap is executed
* all holders share the output proportionally

### ✅ **Full On-Chain Settlement**

No off-chain automation.
Everything is executed natively inside `_afterSwap`.

### ✅ **Supports both directions**

* **zeroForOne = true** → Sell token0 for token1
* **zeroForOne = false** → Sell token1 for token0

---

# 📦 Installation & Setup

### **1. Clone Repository**

```bash
git clone https://github.com/TheOnma/take-profits-hook.git
cd take-profits-hook
```

### **2. Install Dependencies**

#### Install Foundry (if not installed)

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

#### Initialize Uniswap V4 Core & Periphery (submodules)

```bash
git submodule update --init --recursive
```

If submodules haven’t been set yet, add them manually:

```bash
git submodule add https://github.com/Uniswap/v4-core.git lib/v4-core
git submodule add https://github.com/Uniswap/v4-periphery.git lib/v4-periphery
git submodule update --init --recursive
```

#### Install OpenZeppelin + Solmate

```bash
forge install OpenZeppelin/openzeppelin-contracts --no-commit
forge install transmissions11/solmate --no-commit
```

### **3. Build**

```bash
forge build
```

### **4. Test**

```bash
forge test -vvv
```

---

# 🏗 How It Works (Architecture)

## 1. **Placing an Order**

When a user calls:

```solidity
placeOrder(key, tickToSellAt, zeroForOne, amount);
```

The hook:

1. Rounds the tick to the nearest usable tick
2. Adds `inputAmount` to:

   ```
   pendingOrders[poolId][tick][direction]
   ```
3. Mints ERC1155 claim tokens
4. Takes user’s input tokens into the Hook contract

---

## 2. **Execution Trigger**

Execution happens inside **_afterSwap**.

Whenever the pool tick moves:

* If price **increases**, check orders selling **token0**
* If price **decreases**, check orders selling **token1**

For every tick crossed:

* If an order exists → execute a swap with exact input
* Add output to `claimableOutputTokens[orderId]`
* Reduce pending order amounts

Tick is updated after every execution.

---

## 3. **Claiming Output Tokens**

After an order executes:

```solidity
redeem(key, tick, zeroForOne, amountToClaimFor)
```

The user receives:

* proportional output tokens
* their ERC1155 claim tokens are burned

---

## 4. **Cancelling an Order**

User can fully or partially cancel an unfilled order:

```solidity
cancelOrder(key, tick, zeroForOne, amount)
```

They get back the *unexecuted* input amount.

---

# 🧠 Order ID Model

Orders at the same tick + direction in a pool are merged into a single ERC1155 position:

```
orderId = keccak256(poolId, tick, zeroForOne)
```

This makes order handling:

* gas-efficient
* composable
* batch-friendly

---

# 🔁 Swap Settlement Logic

`swapAndSettleBalances()` handles settlement logic around PoolManager:

* For **zeroForOne**:

  * token0 is debited from hook
  * token1 is received
* For **oneForZero**:

  * token1 is debited
  * token0 is received

Using the built-in V4 functions:

* `_settle`
* `_take`

---

# 🧪 Testing Strategy (Recommended)

Test:

* order placement & ERC1155 minting
* cancel edge cases
* tick rounding logic
* batch execution
* multi-order distribution math
* reverting conditions

If you want, I can generate a **complete Foundry test suite** for this hook.

---

# 🗺 Example Usage

### **Place a take-profit order:**

```solidity
hook.placeOrder(key, 200_000, true, 5e18);
```

Sell token0 when tick reaches 200k.

### **Cancel part of the order:**

```solidity
hook.cancelOrder(key, 200_000, true, 2e18);
```

### **Redeem after execution:**

```solidity
hook.redeem(key, 200_000, true, 5e18);
```

---

# 🛠 Deploying the Hook

When creating the pool using the PoolManager:

1. Deploy the hook contract:

```solidity
TakeProfitsHook hook = new TakeProfitsHook(poolManager, "uri");
```

2. Enable the hook in the pool's `Hooks` field.

3. Initialize the pool.

---

# 📄 License

MIT

---

# 🙌 Contributions

PRs are welcome — feel free to improve batching logic, optimize settlement, or integrate MEV-aware execution.

