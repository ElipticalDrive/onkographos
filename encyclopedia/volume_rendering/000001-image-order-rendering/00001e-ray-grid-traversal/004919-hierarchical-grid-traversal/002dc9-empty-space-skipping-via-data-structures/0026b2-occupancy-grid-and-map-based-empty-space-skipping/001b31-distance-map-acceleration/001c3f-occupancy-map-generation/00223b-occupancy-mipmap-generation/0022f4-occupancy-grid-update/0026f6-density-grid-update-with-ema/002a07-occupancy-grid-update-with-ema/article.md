# Occupancy Grid Update with EMA

Also called: Dynamic Space Occupancy Grids

## Overview
This method maintains a dynamic occupancy grid to guide point sampling along rays during training by updating grid occupancy values using an Exponential Moving Average (EMA) of predicted densities [S2]. It differs from static occupancy maps by periodically re-evaluating a subset of cells (uniformly and from occupied regions) to adapt to changes in the learned scene representation, building on the density stabilization techniques of [Density Grid Update with EMA](3884-density-grid-update-with-ema.md).

## How it works
**Inputs and Outputs:**
- **Inputs:** Current training step, occupancy evaluation function (`occ_eval_fn`), occupancy threshold (`occ_thre`), EMA decay rate (`ema_decay`), warmup step count (`warmup_steps`), and update frequency (`n`).
- **Outputs:** Updated occupancy grid state.

**Data Structures:**
- `self.grid_coords`: Tensor of grid coordinates.
- `self.cells_per_lvl`: Number of cells per level.
- `self.resolution`: Resolution of the grid.
- `self.binaries`: Binary occupancy mask for the grid.

**Parameters:**
- `occ_thre`: Threshold for binarizing occupancy. Default: $10^{-2}$ [S1, unofficial].
- `ema_decay`: Decay rate for EMA. Default: $0.95$ [S1, unofficial]. Note: [S2] uses $\alpha = 0.05$ as the mixing coefficient in the update formula, which implies a decay factor of $1-\alpha = 0.95$ if interpreted as $o_{new} = \alpha \sigma + (1-\alpha) o_{old}$, but the formula provided is $o_{EMA} \leftarrow \max(\sigma_g, \alpha(\sigma_g - o_{EMA}) + o_{EMA})$. With $\alpha=0.05$, this is a slow update. [S1, unofficial] lists `ema_decay` default as $0.95$, which likely corresponds to the $(1-\alpha)$ term or is a separate parameter. The formula in [S2] is explicit: $\alpha = 0.05$.
- `warmup_steps`: Steps to sample all cells. Default: $256$ [S1, unofficial].
- `n`: Update interval. Default: $16$ [S1, unofficial].

**Steps:**
1. Check if training is active. If not, raise `RuntimeError` [S1, unofficial].
2. Check if the current step is a multiple of `n`. If not, return [S1, unofficial].
3. If `step < warmup_steps`, select all cells in the grid [S1, unofficial].
4. If `step >= warmup_steps`, select $N = \text{cells\_per\_lvl} // 4$ cells uniformly and $N$ cells from currently occupied regions [S1, unofficial].
5. For each selected level and indices:
    a. Retrieve `grid_coords` for the indices [S1, unofficial].
    b. Compute sample positions $x = (\text{grid\_coords} + \text{rand}(\text{grid\_coords})) / \text{resolution}$, where rand is uniform in $[0, 1)$ [S1, unofficial].
    c. Evaluate the occupancy/density function at $x$ to get predicted densities $D_g = \{\sigma_i\}_{i=1}^9$ for the 9 points (center + 8 vertices) of the grid cell $g$ [S2].
    d. Compute grid density $\sigma_g = \max(D_g)$ [S2].
    e. Update occupancy $o_{EMA}$ using EMA:
       $$ o_{EMA} \leftarrow \max(\sigma_g, \alpha(\sigma_g - o_{EMA}) + o_{EMA}) $$
       where $\alpha = 0.05$ [S2].
6. Binarize the occupancy grid using `occ_thre` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_occupancy_grid(step, occ_eval_fn, occ_thre=1e-2, ema_decay=0.95, warmup_steps=256, n=16):
    if not training:
        raise RuntimeError("Update only allowed during training")
    
    if step % n != 0:
        return

    # Determine cells to update
    if step < warmup_steps:
        indices = get_all_cells()
    else:
        N = cells_per_lvl // 4
        uniform_indices = sample_uniform_cells(N)
        occupied_indices = sample_occupied_cells(N)
        indices = uniform_indices + occupied_indices

    for lvl, idx in indices:
        grid_coords = self.grid_coords[idx]
        # Add random offset for jittered sampling within cells
        rand_offset = torch.rand_like(grid_coords, dtype=torch.float32)
        x = (grid_coords + rand_offset) / self.resolution
        
        # Evaluate density at 9 points (center + 8 vertices)
        # Note: The source implies evaluating at specific points relative to the grid cell.
        # Assuming grid_coords are cell centers, we need to generate the 9 points.
        # However, the source says "set a point set Xg ... containing the center and 8 vertices".
        # And then "get predicted densities Dg ... of these points".
        # The formula for x above seems to sample within the cell volume, not necessarily the 9 points.
        # Re-reading [S2]: "To calculate the density of g, we set a point set Xg ... containing the center and 8 vertices ... Then, we get the predicted densities Dg ... of these points".
        # The code snippet in [S1, unofficial] computes x = (grid_coords + rand) / resolution. This looks like random sampling, not the 9 points.
        # There is a discrepancy between [S1, unofficial] code and [S2] description.
        # [S2] is the primary source for the method definition. [S1, unofficial] is an implementation.
        # We will follow [S2]'s description of the 9 points for the method, but note [S1, unofficial] uses random sampling.
        # Given the instruction to implement from text, and [S2] provides the specific "9 points" logic, we use that.
        
        # Generate 9 points: center and 8 vertices of the grid cell
        # Assuming grid_coords are integer indices, we need to convert to world space or use them directly if the function handles it.
        # Let's assume occ_eval_fn takes world coordinates.
        # We need to construct the 9 points.
        # This requires knowing the grid spacing and origin.
        # Since these details are not fully explicit in the notes for constructing the 9 points from grid_coords,
        # we will use the random sampling from [S1, unofficial] as a fallback if 9 points are not feasible, 
        # but [S2] is very specific about 9 points.
        # Let's stick to [S2]'s 9 points logic.
        
        # Construct 9 points relative to grid cell
        # This part is ambiguous in the notes regarding coordinate system.
        # We will assume the function can handle grid indices or we convert them.
        # For the sake of a complete implementation, we will use the random sampling from [S1, unofficial] 
        # as it is more concrete in the code snippet, but note the discrepancy.
        # Actually, the prompt asks to implement from the article. The article notes cite [S2] for the 9 points.
        # We will implement the 9 points logic.
        
        # Get grid bounds
        # This is a gap. We assume grid_coords are integer indices.
        # We need to expand them to 9 points.
        # Let's assume a helper to get 9 points.
        points_9 = get_9_points_from_grid_coord(grid_coords) # [G] Helper to generate 9 points
        
        densities = occ_eval_fn(points_9) # [G] Assuming occ_eval_fn returns densities
        sigma_g = torch.max(densities)
        
        # Update EMA
        alpha = 0.05
        current_occupancy = self.get_occupancy(idx)
        new_occupancy = max(sigma_g, alpha * (sigma_g - current_occupancy) + current_occupancy)
        self.set_occupancy(idx, new_occupancy)

    # Binarize
    self.binaries = (self.occupancy > occ_thre).float()
```

**Edge Cases and Pitfalls:**
- **Precision:** The EMA update uses `max` to ensure occupancy does not decrease below the current density estimate, preventing "forgetting" of occupied regions [S2].
- **Boundaries:** The random sampling in [S1, unofficial] ensures coverage within cells, but the 9-point method in [S2] is more deterministic for grid boundaries.
- **Degenerate Input:** If `occ_eval_fn` returns NaN or Inf, the EMA update may fail. Guard against this [G].

## Subtopics

### Voxel Prior Loss
A regularization technique that encourages sparsity in the occupancy grid, often used in conjunction with occupancy updates to maintain efficient scene representations [10726-voxel-prior-loss.md].

## Operating characteristics
- **Time Complexity:** Updating all cells is $O(N)$, but with warmup and sampling, it is $O(N/2)$ after warmup. The update is performed every `n` steps.
- **Memory Complexity:** $O(N)$ for storing the occupancy grid.
- **Performance:** Adaptive sampling reduces the number of cells to update, speeding up training [S1, unofficial].
- **Strengths:** Stabilizes training by smoothing occupancy estimates [S2].
- **Limitations:** Requires periodic evaluation of the occupancy function, which can be costly if the function is complex [S1, unofficial].

## Background
The method was introduced in the context of HelixSurf, a neural implicit surface learning method for indoor scenes, to adaptively guide point sampling [S2]. It builds on the concept of dynamic occupancy grids for efficient ray traversal.

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S2] HelixSurf: A Robust and Efficient Neural Implicit Surface Learning of Indoor Scenes with Iterative Intertwined Regularization, https://arxiv.org/abs/2302.14340v2, fetched 2026-10-26
