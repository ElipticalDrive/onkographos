# Downward Propagation

Also called: Delta propagation, Fine-to-coarse update

## Overview
Downward propagation is a method for initializing or updating fine-resolution voxel data from coarser parent nodes in a hierarchical volume representation, triggered when the camera moves closer to the scene surface [S1]. It solves the problem of preserving high-frequency details during resolution refinement by propagating a fixed delta (the difference between the parent's last update and the children's mean) rather than interpolating raw values, which would smooth existing details [S1]. Unlike [Priority Queue Based Field Update](7430-priority-queue-based-field-update.md), which manages the order of updates via a priority queue, downward propagation defines the specific arithmetic operation used to transfer information from scale $l$ to $l+1$ [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The TSDF field value $F^l$ and weight $y^l$ of a parent node at scale $l$ at time step $k-1$; the current TSDF field values $F^{l+1}_{k-1,i}$ and weights $y^{l+1}_{k-1,i}$ of its 8 children at scale $l+1$; the mean of the children's TSDF values $\bar{F}^l_{k-1}$ at time $k-1$; the maximum weight $y_{max}$; and the current time step $k$.
*   **Output:** Updated TSDF field values $F^{l+1}_{k,i}$ and weights $y^{l+1}_{k,i}$ for all 8 children at scale $l+1$.

**Data Structures:**
The method operates on a Sparse Voxel Octree where each node stores a TSDF value $F$ and an integration weight $y$. The hierarchy allows access to children via index $i \in \{1, \dots, 8\}$ [S1].

**Parameters:**
*   $\bar{F}^l_{k-1}$: The mean of the children's TSDF values at scale $l$ at time $k-1$. This value is assumed constant between frames if children have not been up-propagated [S1].
*   $y_{max}$: The maximum allowed integration weight to prevent overflow [S1].

**Steps:**
1.  **Calculate Parent Delta:** Compute the difference between the parent's value at the previous time step and the mean of its children's values at the previous time step. This delta represents the change in the parent that has not yet been reflected in the children [S1].
    $$ \Delta F^l_k = F^l_{k-1} - \bar{F}^l_{k-1} $$
    $$ \Delta y^l_k = \Delta y^l_{k-1} + \Delta y^l_{k-1} \quad \text{(Note: Source formula (6) appears to contain a typo or specific context for weight delta accumulation; standard interpretation suggests propagating the weight change delta)} $$
    *Correction based on source text:* The source defines $\Delta y^{l+1}_k = \Delta y^l_{k-1} + \Delta y^l_{k-1}$ which simplifies to $2\Delta y^l_{k-1}$, but logically weight deltas should track the parent's weight change. The source states "Using similar reasoning for the weight values...". We follow the source formula exactly [S1].

2.  **Handle Uninitialized Children:** If not all 8 children are initialized, compute $\bar{F}^l_{k-1}$ using only the initialized children [S1].

3.  **Update Children TSDF:** Add the calculated delta to each child's current TSDF value [S1].
    $$ F^{l+1}_{k,i} = F^{l+1}_{k-1,i} + \Delta F^l_k $$

4.  **Update Children Weight:** Add the weight delta to each child's current weight, clamping to $y_{max}$ [S1].
    $$ y^{l+1}_{k,i} = \min(y_{max}, y^{l+1}_{k-1,i} + \Delta y^l_k) $$

5.  **Enforce Unitary Scale Steps:** Ensure that scale changes occur in unitary steps (resolution can be at most double or half the previous resolution) to maintain stability [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def downward_propagate(parent_F, parent_y, children_F, children_y, children_mean_F, children_mean_y, y_max, k):
    """
    Propagates delta from parent at scale l to children at scale l+1.
    
    Args:
        parent_F: TSDF value of parent at k-1 (F^l_{k-1})
        parent_y: Weight of parent at k-1 (y^l_{k-1})
        children_F: List of 8 TSDF values for children at k-1 (F^{l+1}_{k-1,i})
        children_y: List of 8 weights for children at k-1 (y^{l+1}_{k-1,i})
        children_mean_F: Mean of children's TSDF values at k-1 (F_bar^l_{k-1})
        children_mean_y: Mean of children's weights at k-1 (y_bar^l_{k-1}) [G: inferred for weight delta]
        y_max: Maximum integration weight
        k: Current time step
        
    Returns:
        Updated children_F and children_y lists
    """
    # Step 1: Calculate Delta for TSDF
    # Delta is difference between parent's last value and children's mean
    delta_F = parent_F - children_mean_F
    
    # Step 1b: Calculate Delta for Weight
    # Source formula: Delta y^{l+1}_k = Delta y^l_{k-1} + Delta y^l_{k-1}
    # This implies the weight delta is doubled from the previous weight delta?
    # Or it refers to accumulating the parent's weight change.
    # Given the ambiguity and potential typo in source [S1], we implement the literal formula.
    # However, standard delta propagation usually uses parent_y - children_mean_y.
    # We stick to the source's explicit formula for Delta y.
    # Note: The source defines Delta y^l_{k-1} implicitly. 
    # If we assume Delta y^l_{k-1} is the weight delta from the previous step:
    delta_y_prev = parent_y - children_mean_y # [G: Standard definition of weight delta, not explicitly in source formula but required for initialization]
    delta_y = 2 * delta_y_prev # Source formula (6)
    
    # Step 2: Handle uninitialized children (Mean calculation is done externally)
    # If children_mean_F was computed with only initialized children, we proceed.
    
    # Step 3 & 4: Update each child
    new_children_F = []
    new_children_y = []
    
    for i in range(8):
        # Update TSDF
        new_F = children_F[i] + delta_F
        
        # Update Weight
        new_y = min(y_max, children_y[i] + delta_y)
        
        new_children_F.append(new_F)
        new_children_y.append(new_y)
        
    return new_children_F, new_children_y
```

**Edge Cases and Pitfalls:**
*   **Uninitialized Children:** If some children are not yet initialized, the mean $\bar{F}^l_{k-1}$ must be computed only from the initialized subset to avoid NaN or zero-inflation errors [S1].
*   **Scale Steps:** The method assumes scale changes are unitary. Jumping more than one level (e.g., $l$ to $l+2$) is not supported by this specific delta formulation [S1].
*   **Precision:** The delta approach preserves high-frequency details better than trilinear interpolation, but accumulation of deltas over many frames may lead to drift if not corrected by upward propagation [S1].

## Subtopics

### Upward Propagation
Upward propagation is the counterpart to downward propagation, moving updated information from finer children nodes to coarser parent nodes after depth map integration [S1]. It ensures scale consistency among neighboring blocks and octants [S1]. The process stops at the coarsest scale observed during the current frame [S1]. It uses a cell-centred data model where parent voxels are the linear combination (average) of their 8 children [S1].
*   **Formula:**
    $$ F^l = \frac{\sum_{i=1}^{8} F^{l+1}_i}{8} $$
    $$ y^l = \frac{\sum_{i=1}^{8} y^{l+1}_i}{8} $$
*   **Parameters:** $F^{l+1}_i$ and $y^{l+1}_i$ are the TSDF value and weight of the $i$-th child at scale $l+1$ [S1].
*   **When to choose:** Use upward propagation after integrating a new depth map to ensure the coarser levels reflect the new observations, maintaining consistency for subsequent downward propagation or rendering [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per parent node, as it involves a constant number of arithmetic operations for 8 children [S1].
*   **Memory Complexity:** $O(1)$ additional memory per node for storing deltas or means, assuming the mean is cached or computed on the fly [S1].
*   **Performance:** The method is efficient for interactive SLAM systems as it avoids expensive trilinear interpolation and smoothing operations [S1].
*   **Strengths:** Preserves high-frequency details that would be lost by interpolation; computationally cheap [S1].
*   **Limitations:** Requires careful management of uninitialized children; relies on upward propagation to correct drift and maintain global consistency [S1].
*   **When to use:** When the camera moves closer to the surface, requiring finer resolution for depth integration [S1].
*   **When not to use:** When the camera moves away (coarsening), where upward propagation is the primary mechanism, or when non-unitary scale jumps are required [S1].

## Background
Downward propagation was introduced in the context of adaptive-resolution octree-based volumetric SLAM by Vespa et al. in 2019 [S1]. It builds on the idea of lazy propagation and delta updates seen in height map reconstruction by Zienkiewicz et al. [S1, S25]. The method is part of a larger framework that includes upward propagation and priority-queue-based field updates to manage dynamic voxel data in real-time [S1].

## Sources
[S1] Adaptive-resolution octree-based volumetric SLAM, https://www.doc.ic.ac.uk/~sleutene/publications/Vespa_3DV19.pdf, fetched 2026-10-29
