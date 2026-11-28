# Priority-Based Content Allocation

Also called: allocateContentTargetsByChildCount, distributeRemainingByPriority

## Overview
Priority-Based Content Allocation is a deterministic method for distributing a fixed content budget among child nodes in a chunk-based spatial partitioning hierarchy. It ensures that the total allocated content targets sum exactly to the available budget while respecting minimum constraints and prioritizing children with higher remaining capacity or count. It builds on [Deterministic Integer Allocation](16520-deterministic-integer-allocation.md) by adding a priority-based tie-breaking mechanism for distributing the residual budget after minimums are satisfied.

## How it works
**Inputs:**
- `node.count`: The total expected Gaussian count for the current node.
- `totalTarget`: The target total content budget for this level.
- `children`: A list of child objects, each having:
  - `child.count`: Expected Gaussian count for the child.
  - `minimumContentTargetForVirtualBudget(child)`: The minimum allocation required for the child.
  - `contentTargetOverride`: The field to store the final allocation.

**Outputs:**
- Updated `contentTargetOverride` for each child.

**Data Structures:**
- `allocation`: Integer array of size $N$ (number of children), initialized to minimums.
- `capacity`: Integer array of size $N$, where `capacity[i] = max(0, children[i].count - allocation[i])`.
- `remainder`: Float scalar, tracking the fractional part of the proportional distribution.

**Steps:**
1. **Filter:** Filter children to only those with `count > 0`. If none, return.
2. **Calculate Minimums:** Compute `minTotal` as the sum of `minimumContentTargetForVirtualBudget(child)` for all filtered children.
3. **Determine Total Budget:** Compute `total` as `max(minTotal, min(node.count, floor(totalTarget)))`.
4. **Initialize:**
   - Set `allocation[i] = minimumContentTargetForVirtualBudget(children[i])` for all $i$.
   - Set `capacity[i] = max(0, children[i].count - allocation[i])`.
   - Set `remainder = 0.0`.
5. **Proportional Distribution:**
   - Compute `remaining = total - sum(allocation)`.
   - Compute `denominator = sum(children[i].count for all i)`.
   - For each child $i$:
     - `exact = (remaining * children[i].count) / denominator`
     - `extra = min(capacity[i], floor(exact))`
     - `allocation[i] += extra`
     - `capacity[i] -= extra`
     - `remainder += exact - extra`
6. **Sort by Priority:** Sort the active allocations based on the following criteria (descending order for all):
   - `remainder` (fractional part from step 5, though typically this is handled by a single pass or specific tie-breaking logic; the source implies sorting by the *potential* remainder or a derived priority metric. Given the source text "Sort active allocations by priority: remainder descending, then child count descending, then child slot ascending", we interpret this as sorting the *entries* by their calculated `exact` value's fractional part, or simply by the `exact` value itself if `remainder` is not stored per-entry. However, the source explicitly lists "remainder descending". Let's assume `remainder` here refers to the fractional component `exact - floor(exact)` for each entry, or the source implies a specific priority queue. Given the ambiguity, we follow the literal sort keys:
     1. `remainder` (fractional part of `exact` for that child) descending.
     2. `child.count` descending.
     3. `child.slot` (index) ascending.
   - *Note:* The source says "remainder descending". In standard proportional allocation, one might distribute the integer part of `exact` and then use the fractional part to decide who gets the +1. The source mentions `remainder` as a scalar in step 4, but then sorts by "remainder descending". This likely refers to the fractional part of the proportional share for each child. Let's define `frac[i] = exact - floor(exact)`.
   - Sort indices $i$ by: `frac[i]` descending, then `children[i].count` descending, then `i` ascending.
7. **Assign Final Allocations:**
   - The source says "Assign final allocations to children: `entry.child.contentTargetOverride = entry.allocation`". This implies the sorted list is used to distribute the *remaining* integer budget if `total` was not fully distributed by the floor operation. However, the source text for step 5 already updates `allocation` with `extra`. The sorting step likely handles the distribution of the *last* few units if `remaining` is not perfectly divisible or if `total` needs to be strictly met.
   - *Correction based on source:* The source says "Sort active allocations by priority... Assign final allocations". It does not explicitly state a second loop. It is possible the "remainder" in the sort key refers to the *unassigned* budget or the fractional part. Given the deterministic nature, the most robust interpretation is:
     - The `allocation` array is updated with `floor(exact)` equivalents? No, step 5 uses `floor(exact)` for `extra`.
     - Let's re-read carefully: `exact = (remaining * entry.child.count) / denominator`. `extra = min(entry.capacity, floor(exact))`. `remainder = exact - extra`.
     - The `remainder` variable in step 4 is a scalar. The sort key "remainder descending" likely refers to the fractional part of the *proportional share* for each child.
     - The final allocation is simply the `allocation` array after step 5? Or does the sort determine who gets the *remaining* integer units if `sum(allocation) < total`?
     - The source says "Assign final allocations...". It does not show a loop. It is possible the `allocation` is already correct, and the sort is for debugging or a secondary purpose? No, "Priority-Based" implies the sort drives the allocation.
     - *Alternative Interpretation:* The `remainder` scalar in step 4 is the total fractional loss. The sort is used to distribute the *remaining* budget if `total` > `sum(allocation)` after step 5. But step 5 uses `floor(exact)`. The sum of `floor(exact)` might be less than `remaining`. The difference is at most $N$. The sort by fractional part is the standard way to distribute these last units.
     - *Revised Step 6/7:*
       - After step 5, `sum(allocation)` might be less than `total`. The difference `diff = total - sum(allocation)` is distributed.
       - Sort children by `frac[i]` descending, then `count` descending, then `index` ascending.
       - For the first `diff` children in the sorted list, `allocation[i] += 1`.
       - *However*, the source does not explicitly show this loop. It just says "Sort... Assign".
       - *Strict Source Adherence:* The source says: "Sort active allocations by priority: remainder descending, then child count descending, then child slot ascending. Assign final allocations to children: `entry.child.contentTargetOverride = entry.allocation`."
       - This implies the `allocation` array is already fully computed, and the sort might be for *which* child gets which *slot* if there are multiple slots? Or perhaps the `allocation` variable in the source is a list of objects with `allocation` and `remainder` fields?
       - Let's assume the source implies that the `allocation` is determined by the sort order if there are ties or specific priority rules. But the formula `extra = min(capacity, floor(exact))` is deterministic.
       - *Gap Filling [G]:* To ensure the total sums to `total`, if `sum(allocation) < total` after step 5, the remaining units are distributed to the children with the highest fractional parts `frac[i] = exact - floor(exact)`. If `sum(allocation) > total`, it's an error. The sort order defines the tie-breaking for these last units.
       - *Implementation:*
         1. Calculate `frac[i] = exact - floor(exact)`.
         2. Sort indices by `frac[i]` desc, `count` desc, `index` asc.
         3. `diff = total - sum(allocation)`.
         4. For `k` from 0 to `diff - 1`: `allocation[sorted_indices[k]] += 1`.
         5. Assign `contentTargetOverride`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def allocate_content(node, children):
    # Filter children with count > 0
    active_children = [c for c in children if c.count > 0]
    if not active_children:
        return

    # Calculate minimums
    min_totals = [c.minimum_content_target_for_virtual_budget() for c in active_children]
    min_total = sum(min_totals)
    
    # Determine total budget
    total = max(min_total, min(node.count, floor(node.total_target)))
    
    # Initialize allocations
    allocations = list(min_totals)
    capacities = [max(0, c.count - a) for c, a in zip(active_children, allocations)]
    
    # Proportional distribution
    remaining = total - sum(allocations)
    denominator = sum(c.count for c in active_children)
    
    if denominator == 0:
        return # Should not happen due to filter
        
    exact_shares = [(remaining * c.count) / denominator for c in active_children]
    
    # Distribute integer parts
    for i, (c, exact) in enumerate(zip(active_children, exact_shares)):
        extra = min(capacities[i], floor(exact))
        allocations[i] += extra
        capacities[i] -= extra
        
    # Handle remaining fractional units
    # Calculate fractional parts
    frac_shares = [exact - floor(exact) for exact in exact_shares]
    
    # Create indices and sort by priority
    indices = list(range(len(active_children)))
    # Sort key: frac desc, count desc, index asc
    indices.sort(key=lambda i: (-frac_shares[i], -active_children[i].count, i))
    
    # Distribute remaining units
    current_sum = sum(allocations)
    diff = total - current_sum
    
    # Assign to top 'diff' children in sorted order
    for k in range(diff):
        if k < len(indices):
            allocations[indices[k]] += 1
            
    # Assign final allocations
    for i, c in enumerate(active_children):
        c.content_target_override = allocations[i]
```

## Subtopics

### Instance Lifecycle and Zombie Recycling
This variant manages the lifecycle of instances in a registry to handle capacity constraints. It tracks match/miss counters for each instance. If an instance is unmatched for $\theta_{miss}$ frames, it is marked as a zombie. When the registry nears capacity $J$, zombies are recycled (longest-missing first), followed by all-zero columns. Recycling zeroes the evidence column and returns the ID to the pool. Parameters: $J$ (Maximum registry capacity), $\theta_{miss}$ (Window for missing state before becoming zombie) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ due to sorting, where $N$ is the number of children.
- **Memory Complexity:** $O(N)$ for storing allocations and capacities.
- **Strengths:** Deterministic, respects minimum constraints, prioritizes larger children for proportional distribution.
- **Limitations:** Requires sorting, which may be costly for very large numbers of children. Precision issues with floating-point arithmetic in proportional distribution may require careful handling of remainders.
- **When to use:** When a fixed budget must be distributed among children with varying counts and minimum requirements, and priority based on size or fractional remainder is desired.

## Background
The method is described in the context of 3D Gaussian Splatting and 3D Tiles conversion, specifically in the `lzy-0618/3DGS-PLY-3DTiles-Converter` repository [S1, unofficial]. It is part of a chunk-based world streaming and lifecycle management system.

## Sources
[S1] lzy-0618/3DGS-PLY-3DTiles-Converter, https://github.com/lzy-0618/3DGS-PLY-3DTiles-Converter, fetched 2026-10-27 (unofficial)
[S2] DIGS: Distribution-Informed Gaussian Splatting for Training-Free Open-Vocabulary 3D Segmentation, https://doi.org/10.48550/arxiv.2610.16177, fetched 2026-10-26 (unofficial)
