# Adaptive Pruning

Also called: Triangle Pruning based on Volume Rendering Weight

## Overview
Adaptive Pruning is a sparsification technique that reduces floaters and overfitting by removing primitives that explain only small amounts of data within a single training view [S1]. It distinguishes itself from opacity-only methods by evaluating the actual volume rendering blending weight ($T \cdot o$) accumulated across all training views, ensuring that only primitives with significant visual impact are retained [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ primitives (triangles or splats) with attributes (position, shape, opacity, color) and a set of training views [S1][S2, unofficial].
*   **Output:** A filtered set of primitives $\mathcal{P}'$ with compacted attribute buffers [S2, unofficial].

**Data Structures:**
Primitive attributes are stored in contiguous arrays of size $N$ [G]. Tracking arrays of size $N$ store `max_weight` (initialized to $0$) and `view_count` (initialized to $0$) [G]. A boolean `alive` mask of size $N$ determines primitive validity [S2, unofficial].

**Steps:**
1.  Initialize `max_weight` and `view_count` arrays to zero for all $N$ primitives [G].
2.  For each training view, rasterize the primitives to compute the volume rendering blending weight $w = T \cdot o$ (transmittance times opacity) and the pixel coverage count for each primitive [S1].
3.  Update `max_weight[i] = max(max_weight[i], w_i)` and increment `view_count[i]` if the primitive covers more than one pixel in the current view [S1].
4.  After processing all views, construct the `alive` mask where `alive[i]` is true if `max_weight[i] >= τ_prune` AND `view_count[i] >= 2` [S1].
5.  If the number of alive primitives is less than $N$, filter all attribute buffers using the `alive` mask to remove invalid primitives [S2, unofficial].
6.  Re-initialize the optimizers and learning rate scheduler for the remaining primitives to align gradient states with the new buffer indices [S2, unofficial].

**Parameters:**
*   $\tau_{prune}$: A user-defined threshold for the maximum volume rendering blending weight below which a primitive is pruned [S1].
*   Pruning Frequency: Pruning executes every 200 steps if `step > 0`, `step % 200 == 0`, and `prune_threshold > 0` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_pruning(primitives, training_views, tau_prune, step, prune_threshold):
    # Step 0: Check execution condition from [S2, unofficial]
    if step <= 0 or step % 200 != 0 or prune_threshold <= 0:
        return primitives

    N = len(primitives['positions'])
    max_weight = [0.0] * N
    view_count = [0] * N

    # Step 1-3: Accumulate weights and coverage across all training views [S1]
    for view in training_views:
        # [G] Standard volume rendering rasterization.
        # Returns arrays of size N: blending weights (T*o) and pixel coverage counts.
        weights, coverage = compute_rasterization_weights(primitives, view)
        for i in range(N):
            if weights[i] > max_weight[i]:
                max_weight[i] = weights[i]
            if coverage[i] > 1:
                view_count[i] += 1

    # Step 4: Generate alive mask [S1]
    alive = [max_weight[i] >= tau_prune and view_count[i] >= 2 for i in range(N)]
    num_alive = sum(alive)

    # Step 5: Filter buffers [S2, unofficial]
    if num_alive < N:
        for key in primitives:
            primitives[key] = [primitives[key][i] for i in range(N) if alive[i]]
        # Step 6: Reset optimization state [S2, unofficial]
        # [G] Clears optimizer buffers (momentum/variance) and resets scheduler
        # to align gradient states with the new compacted primitive indices.
        reinitialize_optimizers(primitives)

    return primitives

def compute_rasterization_weights(primitives, view):
    # [G] Standard volume rendering pass. Returns arrays of size N:
    # blending weights (T*o) and pixel coverage counts.
    pass

def reinitialize_optimizers(primitives):
    # [G] Re-initializes optimizer state and learning rate scheduler
    # for the remaining primitives.
    pass
```

**Edge Cases and Pitfalls:**
*   **Optimizer State Mismatch:** Failing to re-initialize the optimizer after pruning leaves stale gradient states (e.g., momentum buffers) indexed to removed primitives, causing incorrect updates for remaining primitives [S2, unofficial].
*   **Degenerate Coverage:** The "more than one pixel" coverage check must correctly handle primitives that project to sub-pixel areas or single pixels, which should not count as significant coverage [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Effectively removes floaters and overfitted primitives that have low visual impact, improving reconstruction quality and reducing memory usage [S1].
*   **Limitations:** Requires processing all training views to accumulate accurate weights and coverage counts, which adds computational overhead during training [S1].
*   **When to Use:** Use when the scene contains significant noise or overfitting, leading to the emergence of spurious primitives that do not correspond to real geometry [S1].

## Background
Adaptive Pruning was introduced in the context of Triangle Splatting for Real-Time Radiance Field Rendering [S1]. It was developed as an alternative to manual heuristics for densification and pruning, aiming to automatically reduce representation power where it is not needed [S1].

## Sources
[S1] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S2] SpaceCollectors/dust3r-3dgs, https://github.com/SpaceCollectors/dust3r-3dgs, fetched 2026-10-27 (unofficial)
