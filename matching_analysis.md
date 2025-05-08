# Analysis of Matching as a MILP

Let's model an automated program (Bot) that matches limit orders using a Mixed-Integer Linear Program (MILP) model. Ultimately it was not applied due to the differences between uint128 and float64 arithmetics: using float64 to represent uint128 would suffer from many issues, such as the inability to represents sats of reasonably big values or, in the objective function, mixing reasonably big amounts with sats-alike amounts.

## Requirements

### Currency Balance Dynamics

Let Balance_A and Balance_B denote the available amounts of Currencies A and B respectively.

For each order, depending on its type, the flow is as follows:

- Buy Order (order creator wants to buy B using A):  
  - Bot is effectively selling B and receiving A.  
  - For a matched quantity yᵢ with limit price Lᵢ (units of A per unit of B):  
    - Balance_B decreases by yᵢ  
    - Balance_A increases by Lᵢ × yᵢ

- Sell Order (order creator wants to sell B for A):  
  - Bot is essentially buying B and spending A.  
  - For a matched quantity yᵢ with limit price Lᵢ:  
    - Balance_A decreases by Lᵢ × yᵢ  
    - Balance_B increases by yᵢ

### Profit Calculation

With a known reference market exchange ratio R (units of A per unit of B):

- For a Buy Order:  
  - Profit per unit = (Limit Price Lᵢ − R)  
  - Total profit = yᵢ × (Lᵢ − R)

- For a Sell Order:  
  - Profit per unit = (R − Limit Price Lᵢ)  
  - Total profit = yᵢ × (R − Lᵢ)

Note: Even if some individual order looks unprofitable on its own, matching it might unlock additional opportunities for follow-on arbitrage during the same snapshot. Thus, it’s acceptable to match orders resulting in negative profit if they enable overall positive arbitrage.

### Snapshot Matching

All matching decisions are made in a single snapshot, meaning that funds freed from one matched order (the increases in Balance_A or Balance_B) become immediately available for matching with other orders in the same optimization pass.

## Mathematical Formulation

### Indices and Sets

Let i ∈ I represent each limit order available in the snapshot.

### Parameters

- Qᵢ : Total order quantity (in units of Currency B)  
- mᵢ : Minimum partial match size for order i  
- Lᵢ : Limit price (in units of A per unit B) for order i  
- R : Reference market exchange ratio (units of A per unit B)  
- typeᵢ ∈ {buy, sell} indicating the order type:
  - For a buy order, the order creator wishes to buy B by providing A (Bot sells B).  
  - For a sell order, the order creator wishes to sell B for A (Bot buys B).  
- Balance_A⁰ and Balance_B⁰ : Initial available amounts in Currency A and B respectively.

### Decision Variables

- `xᵢ ∈ {0, 1}` indicator variable for whether order i is selected (1) or not (0)
- `yᵢ ≥ 0` continuous variable, matched quantity (in units of B) for order i

### Objective Function is Maximize Total Profit

 Profit depends on order type and amount matched. We define profit per order as follows:

- For a buy order (fulfilling an order where the order creator is buying B, and Bot sells B):  
  - Profit per unit = (Lᵢ – R)  
- For a sell order (fulfilling an order where the order creator is selling B, and Bot buys B):  
  - Profit per unit = (R – Lᵢ)

Thus, the objective is:

```logic
Maximize ∑ᵢ [ (if typeᵢ = buy then (Lᵢ – R) else (R – Lᵢ)) × yᵢ ]
```

This objective function may include orders with negative per-unit profit if they enable follow-on arbitrage.

### Constraints

#### 1. Order Matching Constraints (force minimum if matched)

```logic
∀ i ∈ I:
   yᵢ ≥ mᵢ · xᵢ       (1)
   yᵢ ≤ Qᵢ · xᵢ       (2)
```

#### 2. Currency Balance Constraints (Snapshot Constraint)

Because this is a single snapshot where the orders are processed simultaneously—and funds released immediately after matching—one convenient approach is to write “net flow” constraints ensuring that Bot final balances (after executing all matches) remain nonnegative. Here’s how:

Define the net flow (delta) for each currency per order:

For each order i:

- If order i is a buy order, so Bot gains A by selling B:

```logic
Delta_A(i) = + (Lᵢ × yᵢ)
Delta_B(i) = – yᵢ  
```

- If order i is a sell order, so Bot sells A to gain B:

```logic
Delta_A(i) = – (Lᵢ × yᵢ)
Delta_B(i) = + yᵢ  
```

Now, because funds freed are immediately available, we must ensure that the net effect does not require “negative” funds beyond Bot initial balances. A commonly used way in such snapshot models is to require that the sum of upfront resource usages (for A and B) does not exceed what Bot has. In our case, because the orders are executed simultaneously, we must ensure that the aggregate flows leave us with at least 0 balance.

Thus, the balance constraints are:

```logic
Balance_A⁰ + [∑ (Flow in A from orders)] ≥ 0  
Balance_B⁰ + [∑ (Flow in B from orders)] ≥ 0
```

This gives:

```logic
Balance_A⁰ + ∑_{i∈I} {  
   if order i is buy:  + Lᵢ · yᵢ  
   if order i is sell:  – Lᵢ · yᵢ  
} ≥ 0               (3)

Balance_B⁰ + ∑_{i∈I} {  
   if order i is buy:  – yᵢ  
   if order i is sell:  + yᵢ  
} ≥ 0               (4)
```

Note: In a snapshot model with simultaneous execution, we may also consider modeling these constraints within the optimization engine's “flow” logic. Alternatively, if the matching order can be sequenced, then a sequential model would update the available balances after each match (but that leads into multi-period modeling).

#### 3. Additional constraints

Additional constraints may include bounds on yᵢ (non-negativity is already declared) and xᵢ binary.

## MILP Model

```logic
Maximize: Z = ∑ᵢ Pᵢ yᵢ  
  where
   Pᵢ = { (Lᵢ – R) if buy order  
         (R – Lᵢ) if sell order }

Subject to:
 ∀ i ∈ I:
  yᵢ ≥ mᵢ · xᵢ  
  yᵢ ≤ Qᵢ · xᵢ  
  xᵢ ∈ {0, 1},  yᵢ ≥ 0

 Balance Constraints:
  Balance_A⁰ + ∑_{i: buy orders} (Lᵢ · yᵢ) – ∑_{i: sell orders} (Lᵢ · yᵢ) ≥ 0  
  Balance_B⁰ – ∑_{i: buy orders} yᵢ + ∑_{i: sell orders} yᵢ ≥ 0
```

## Fractional Variant

- Fractional Representation:  
  Instead of using the continuous variable yᵢ (matched quantity in original units), we reparameterized it using fᵢ ∈ [0, 1] so that yᵢ = Qᵢ · fᵢ. This represents the fraction of each order that is matched.

- Precomputed Constants:  
  We introduced constants to simplify the formulation:  
  - Gᵢ = Pᵢ · Qᵢ (maximum gain per order)  
  - μᵢ = mᵢ / Qᵢ (minimum fractional match)  
  - LQᵢ = Lᵢ · Qᵢ (maximum currency flow per order)
  
- Updated Constraints with Binary Coupling:  
  The activation of an order is now controlled by a binary variable xᵢ that is coupled with fᵢ through:  
  - fᵢ ≥ μᵢ · xᵢ  
  - fᵢ ≤ xᵢ  
  ensuring that if xᵢ = 0 then fᵢ = 0, and if xᵢ = 1 then fᵢ is at least μᵢ.

- Normalization Considerations for float64 Solvers:  
  With the need to convert bigint representations to float64, the revised model uses normalized fractions (e.g., 1.0 for a full match) to maintain consistency and ease conversion between bigints and float64.

These changes aim to simplify the model, improve clarity, and address precision/scale issues when working with bigint data converted into float64.