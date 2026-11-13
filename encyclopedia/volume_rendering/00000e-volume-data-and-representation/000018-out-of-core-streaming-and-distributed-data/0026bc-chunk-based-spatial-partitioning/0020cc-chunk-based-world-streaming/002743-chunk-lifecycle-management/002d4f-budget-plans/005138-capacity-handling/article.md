# Capacity Handling

Also called: Block Quota Rounding, Budget Assignment, Integer Budget Allocation

## Overview
Capacity Handling converts continuous block quotas derived from logarithmic smoothing into integer budgets that strictly respect block capacity limits and sum to the global target count $K$. It serves as the final step in [Budget Plans](5497-budget-plans.md), ensuring that the number of retained Gaussians per chunk is a valid integer before [Quadratic Selection](52-quadratic-selection.md) determines which specific Gaussians to keep. It differs from simple rounding by handling capacity clipping and deficit redistribution through iterative passes.

## How it works
**Inputs:**
- $K$: Target global retained count.
- $\tilde{K}_b$: Continuous block quota for block $b$, computed via logarithmic smoothing (Equation 5 in [S1, unofficial]).
- $n_b$: Capacity (maximum number of Gaussians) of block $b$.
- $B$: Total number of blocks.

**Outputs:**
- $K_b$: Integer budget for block $b$, such that $0 \le K_b \le n_b$ and $\sum_{b=1}^B K_b = K$.

**Data Structures:**
- Block list: Array of blocks indexed $1 \dots B$.
- Fields per block: $n_b$ (int), $\tilde{K}_b$ (float), $K_b$ (int), $f_b$ (float, fractional remainder).

**Steps:**

1. **Clip and Initialize:**
   Clip the continuous quota to the valid range $[0, n_b]$ and initialize the integer budget by flooring:
   $$ \hat{K}_b = \min\{n_b, \max(0, \tilde{K}_b)\} $$
   $$ K_b \leftarrow \lfloor \hat{K}_b \rfloor $$
   [G] If smoothing is disabled or $K=0$, return initial Top-K counts directly.

2. **Compute Deficit and Residuals:**
   Calculate the remaining budget $H$ and the fractional residuals $f_b$:
   $$ H = K - \sum_{b=1}^B K_b $$
   $$ f_b = \hat{K}_b - K_b $$
   Note: $f_b \in [0, 1)$.

3. **Iterative Allocation:**
   While $H > 0$:
   a. Identify eligible blocks: those where $K_b < n_b$.
   b. Sort eligible blocks by descending $f_b$. Break ties by ascending block index (to ensure determinism).
   c. Determine count to update: $c = \min(H, \text{number of eligible blocks})$.
   d. For the first $c$ blocks in the sorted list:
      $$ K_b \leftarrow K_b + 1 $$
      $$ f_b \leftarrow f_b - 1 $$
      (Note: $f_b$ becomes negative, effectively removing it from future consideration as it is now full or has excess).
   e. Update $H \leftarrow H - c$.

**Edge Cases:**
- **Clipping Deficit:** If $\tilde{K}_b > n_b$, clipping reduces the sum of floored budgets significantly, creating a large $H$. This requires multiple passes to redistribute the deficit.
- **Zero Quota:** Blocks with $\tilde{K}_b < 1$ may have $K_b=0$ initially. They can receive units if they have capacity and high fractional remainders from other blocks' clipping, though typically low-score blocks lose Gaussians.
- **Tie Breaking:** Global index tie-breaking ensures deterministic behavior when fractional remainders are equal.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def capacity_handling(blocks, K):
    """
    blocks: list of dicts with keys 'id', 'nb' (capacity), 'K_tilde' (continuous quota)
    K: int, target total retained count
    Returns: list of dicts with updated 'Kb' (integer budget)
    """
    B = len(blocks)
    
    # Step 1: Clip and Initialize
    for b in blocks:
        # Clip to [0, nb]
        K_hat = min(b['nb'], max(0, b['K_tilde']))
        # Floor to integer
        b['Kb'] = int(K_hat)
        # Store fractional part for sorting
        b['fb'] = K_hat - b['Kb']
        
    # Step 2: Compute Deficit
    current_sum = sum(b['Kb'] for b in blocks)
    H = K - current_sum
    
    # If no deficit (e.g. K=0 or exact integers), return
    if H <= 0:
        return blocks

    # Step 3: Iterative Allocation
    while H > 0:
        # Identify eligible blocks: Kb < nb
        eligible = [b for b in blocks if b['Kb'] < b['nb']]
        
        if not eligible:
            # Should not happen if K <= sum(nb), but guard against infinite loop
            break
            
        # Sort by descending fb, then ascending id
        eligible.sort(key=lambda x: (-x['fb'], x['id']))
        
        # Number of blocks to increment
        count = min(H, len(eligible))
        
        for i in range(count):
            b = eligible[i]
            b['Kb'] += 1
            b['fb'] -= 1  # Reduce fractional remainder
            
        H -= count
        
    return blocks
```

## Subtopics

### Block Loading Badges
Determines which blocks are loaded based on the budgets assigned by Capacity Handling. It uses the integer budgets $K_b$ to prioritize chunk loading during streaming. [Block Loading Badges](6011-block-loading-badges.md)

## Operating characteristics
- **Time Complexity:** $O(B \log B)$ per pass due to sorting. In the worst case (heavy clipping), multiple passes are required, but each pass reduces $H$. The number of passes is bounded by the maximum clipping depth.
- **Memory Complexity:** $O(B)$ to store budgets and residuals.
- **Strengths:** Guarantees exact sum $K$ and capacity constraints. Deterministic via tie-breaking. Handles extreme disparities in continuous quotas caused by clipping.
- **Limitations:** Iterative nature can be slower if many blocks are clipped. Not given in the sources: performance impact on very large $B$ with high clipping rates.

## Background
Capacity Handling is part of the SplatQ method for interaction-aware pruning of 3D Gaussian Splatting. It addresses the issue that continuous importance scores (from logarithmic smoothing of diagonal scores) do not naturally sum to integers or respect block capacities. The method was introduced in [S1, unofficial].

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
