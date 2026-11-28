# Logit-based Root Finding (L-Tracing)

Also called: L-Tracing, Root Finding Surface Points, secant method for implicit surfaces

## Overview
L-Tracing is a root-finding method for determining the intersection distance $d$ of a ray with an implicit surface defined by a neural network outputting logits [S1, unofficial]. It solves the problem of locating the zero-crossing of a signed distance-like function in a differentiable rendering pipeline by combining a coarse grid search with a secant method refinement [S1, unofficial]. It is distinguished from standard Signed Distance Function (SDF) intersection methods by its specific handling of neural network logits, where the convention is positive outside the surface and negative inside, and by its use of a cost-matrix-based sign-change detection to efficiently identify the first entry point along the ray [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `rays_o`: Tensor of ray origins, shape `[B, N_rays, 3]` or `[N_rays, 3]`.
    *   `rays_d`: Tensor of ray directions, shape `[B, N_rays, 3]` or `[N_rays, 3]`.
    *   `near`: Scalar or tensor specifying the near plane distance.
    *   `far`: Scalar or tensor specifying the far plane distance.
    *   `surface_query_fn`: A callable neural network that takes 3D points and returns logits.
    *   `logit_tau`: Scalar bias subtracted from network output to center logits.
    *   `N_steps`: Integer, number of coarse samples along the ray (default 256).
    *   `N_secant_steps`: Integer, number of secant iterations (passed to refinement).
    *   `fill_inf`: Boolean, how to handle rays with no intersection.
*   **Outputs:**
    *   `pt_pred`: Tensor of 3D intersection points, shape `[B, N_rays, 3]`.
    *   `d_pred`: Tensor of intersection distances, shape `[B, N_rays]`.

**Data Structures**
*   `t`: Linear space tensor of shape `[B, N_rays, N_steps, 1]` with values in $[0, 1]$.
*   `d_proposal`: Tensor of proposed distances, shape `[B, N_rays, N_steps, 1]`.
*   `p_proposal`: Tensor of query points, shape `[B, N_rays, N_steps, 3]`.
*   `val`: Tensor of logits, shape `[B, N_rays, N_steps]`.
*   `sign`: Sign matrix, shape `[B, N_rays, N_steps]`.
*   `cost_matrix`: Matrix for finding the first sign change, shape `[B, N_rays, N_steps]`.

**Parameters**
*   `N_steps`: Default 256. Controls coarse grid resolution.
*   `logit_tau`: Bias term. Not given in the sources.
*   `N_secant_steps`: Not given in the sources.

**Algorithm Steps**
1.  **Batching:** If `batched` is False, unsqueeze `rays_o` and `rays_d` at dimension 0 to ensure shape `[1, N_rays, 3]`.
2.  **Tensor Conversion:** Convert scalar `near` and `far` to tensors matching the shape of `rays_o` if they are not already tensors.
3.  **Coarse Sampling Grid:** Create `t` as a linear space from 0.0 to 1.0 with `N_steps` elements. Reshape `t` to `[B, N_rays, N_steps, 1]`.
4.  **Propose Distances:** Compute proposal distances:
    $$d_{proposal} = near \times (1 - t) + far \times t$$
    Compute query points:
    $$p_{proposal} = rays_o + d_{proposal} \times rays_d$$
5.  **Network Query:** Query the surface network at $p_{proposal}$ to get logits $val_{raw}$. Center the values:
    $$val = surface\_query\_fn(p_{proposal}) - logit\_tau$$
    Shape of `val` is `[B, N_rays, N_steps]`.
6.  **Initial Occupancy Check:** Check if the first sample point (index 0) is inside the surface. The convention is positive outside, negative inside. If $val[..., 0] > 0$, the first point is outside (not occupied). Let $mask_{0\_not\_occupied}$ be the boolean mask where $val[..., 0] > 0$.
7.  **Sign Change Detection:**
    *   Construct a sign matrix for adjacent pairs. For $i < N_{steps}-1$:
        $$sign[i] = \text{sign}(val[i] \times val[i+1])$$
        For the last element, set $sign[N_{steps}-1] = 1$.
    *   Create a cost matrix by multiplying the sign matrix by descending indices $[N_{steps}, N_{steps}-1, \dots, 1]$.
    *   Find the minimum cost and its index for each ray. A negative minimum cost indicates a sign change occurred.
8.  **Masking Valid Rays:**
    *   Identify rays with at least one sign change: $mask_{sign\_change} = (values < 0)$.
    *   Verify the sign change is from positive (outside) to negative (inside). Check if the value at the identified index is positive: $mask_{pos\_to\_neg} = val[..., indices] > 0$.
    *   Combine masks: $mask = mask_{sign\_change} \text{ AND } mask_{pos\_to\_neg} \text{ AND } mask_{0\_not\_occupied}$.
9.  **Secant Method Refinement:**
    *   Extract bounds for masked rays:
        *   $d_{high}, f_{high}$: Distance and value at the index where sign change was detected.
        *   $d_{low}, f_{low}$: Distance and value at the next index (clamped to $N_{steps}-1$).
    *   Run the secant method using $f_{low}, f_{high}, d_{low}, d_{high}$, ray origins/directions, and the network function to find refined depth $d_{pred}$ for masked rays.
10. **Compute Points:**
    *   For masked rays: $pt_{pred} = rays_o + d_{pred} \times rays_d$.
    *   For unmasked rays: Initialize $pt_{pred}$ to ones (placeholder).
11. **Handle No Intersection:**
    *   If `fill_inf` is false, set depth to `far`.
    *   If `fill_inf` is true, set depth to `np.inf`.
    *   If the first point was occupied ($mask_{0\_not\_occupied} == 0$), set depth to 0.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def l_tracing(rays_o, rays_d, near, far, surface_query_fn, logit_tau, 
              N_steps=256, N_secant_steps=10, fill_inf=False, batched=False):
    """
    L-Tracing: Logit-based Root Finding for implicit surfaces.
    
    Args:
        rays_o: Ray origins [B, N_rays, 3] or [N_rays, 3]
        rays_d: Ray directions [B, N_rays, 3] or [N_rays, 3]
        near: Near plane distance (scalar or tensor)
        far: Far plane distance (scalar or tensor)
        surface_query_fn: Neural network returning logits for 3D points
        logit_tau: Bias to subtract from logits
        N_steps: Number of coarse samples
        N_secant_steps: Number of secant iterations
        fill_inf: If True, set no-intersection depth to inf, else far
        batched: If False, assume single batch dimension
        
    Returns:
        pt_pred: Intersection points [B, N_rays, 3]
        d_pred: Intersection distances [B, N_rays]
    """
    # 1. Batching
    if not batched:
        rays_o = rays_o.unsqueeze(0)
        rays_d = rays_d.unsqueeze(0)
    
    B, N_rays, _ = rays_o.shape
    
    # 2. Tensor Conversion
    if not isinstance(near, torch.Tensor):
        near = torch.full((B, N_rays), near, device=rays_o.device)
    if not isinstance(far, torch.Tensor):
        far = torch.full((B, N_rays), far, device=rays_o.device)
        
    # 3. Coarse Sampling Grid
    t = torch.linspace(0.0, 1.0, N_steps, device=rays_o.device).view(1, 1, N_steps, 1)
    t = t.expand(B, N_rays, N_steps, 1)
    
    # 4. Propose Distances
    d_proposal = near * (1 - t) + far * t
    d_proposal = d_proposal.squeeze(-1) # Shape [B, N_rays, N_steps]
    
    # Expand rays for broadcasting
    rays_o_exp = rays_o.unsqueeze(2).expand(B, N_rays, N_steps, 3)
    rays_d_exp = rays_d.unsqueeze(2).expand(B, N_rays, N_steps, 3)
    
    p_proposal = rays_o_exp + d_proposal.unsqueeze(-1) * rays_d_exp
    
    # 5. Network Query
    # surface_query_fn should handle batched points
    val_raw = surface_query_fn(p_proposal) # Shape [B, N_rays, N_steps]
    val = val_raw - logit_tau
    
    # 6. Initial Occupancy Check
    # Positive outside, negative inside. First point outside means val > 0
    mask_0_not_occupied = val[:, :, 0] > 0
    
    # 7. Sign Change Detection
    # Sign of product of adjacent values
    sign_matrix = torch.sign(val[:, :, :-1] * val[:, :, 1:])
    # Last element sign is 1
    last_sign = torch.ones(B, N_rays, 1, device=rays_o.device)
    sign_matrix = torch.cat([sign_matrix, last_sign], dim=-1)
    
    # Cost matrix: weight by descending index to find first sign change
    indices = torch.arange(N_steps, 0, -1, device=rays_o.device).view(1, 1, N_steps)
    cost_matrix = sign_matrix * indices
    
    # Find min cost and index
    min_cost, indices = torch.min(cost_matrix, dim=-1)
    
    # 8. Masking Valid Rays
    mask_sign_change = min_cost < 0
    # Check if value at index is positive (outside) -> entering surface
    # Gather values at detected indices
    # indices shape [B, N_rays], need to gather from val [B, N_rays, N_steps]
    val_at_index = torch.gather(val, 2, indices.unsqueeze(-1)).squeeze(-1)
    mask_pos_to_neg = val_at_index > 0
    
    mask = mask_sign_change & mask_pos_to_neg & mask_0_not_occupied
    
    # 9. Secant Method Refinement
    d_pred = torch.full((B, N_rays), float('inf'), device=rays_o.device)
    
    if mask.any():
        # Extract bounds
        # d_high, f_high at index
        d_high = torch.gather(d_proposal, 2, indices.unsqueeze(-1)).squeeze(-1)
        f_high = val_at_index
        
        # d_low, f_low at next index (clamped)
        next_indices = torch.clamp(indices + 1, max=N_steps - 1)
        d_low = torch.gather(d_proposal, 2, next_indices.unsqueeze(-1)).squeeze(-1)
        f_low = torch.gather(val, 2, next_indices.unsqueeze(-1)).squeeze(-1)
        
        # Run secant method for masked rays
        # This is a placeholder for the actual secant iteration loop
        # which would iterate N_secant_steps times updating d based on f_low, f_high
        d_pred[mask] = run_secant_method(
            d_low[mask], f_low[mask], d_high[mask], f_high[mask],
            rays_o[mask], rays_d[mask], surface_query_fn, N_secant_steps
        )
        
    # 10. Compute Points
    pt_pred = torch.ones(B, N_rays, 3, device=rays_o.device)
    if mask.any():
        pt_pred[mask] = rays_o[mask] + d_pred[mask].unsqueeze(-1) * rays_d[mask]
        
    # 11. Handle No Intersection
    if not fill_inf:
        d_pred[~mask] = far[~mask]
    else:
        d_pred[~mask] = float('inf')
        
    # If first point was occupied, depth is 0
    d_pred[~mask_0_not_occupied] = 0
    
    return pt_pred, d_pred

def run_secant_method(d_low, f_low, d_high, f_high, rays_o, rays_d, 
                      surface_query_fn, N_secant_steps):
    """
    Placeholder for secant method refinement.
    Iteratively refines the root between d_low and d_high.
    """
    d_curr = d_low.clone()
    f_curr = f_low.clone()
    d_prev = d_high.clone()
    f_prev = f_high.clone()
    
    for _ in range(N_secant_steps):
        # Secant step
        # d_next = d_curr - f_curr * (d_prev - d_curr) / (f_prev - f_curr)
        # Guard against division by zero
        denom = f_prev - f_curr
        denom = torch.where(denom == 0, torch.ones_like(denom) * 1e-8, denom)
        d_next = d_curr - f_curr * (d_prev - d_curr) / denom
        
        # Query surface at new point
        p_next = rays_o + d_next.unsqueeze(-1) * rays_d
        f_next = surface_query_fn(p_next) - logit_tau # logit_tau must be accessible or passed
        
        # Update bounds
        # If f_next has same sign as f_curr, replace curr with next
        # Otherwise replace prev with next
        # Standard secant method updates both points
        d_prev = d_curr.clone()
        f_prev = f_curr.clone()
        d_curr = d_next.clone()
        f_curr = f_next.clone()
        
    return d_curr
```

**Subtopics**
*   **Secant Method Refinement:** A root-finding algorithm that approximates the root by drawing a secant line between two points. It converges faster than bisection but requires derivative-like information (slope) from two previous points. Choose when the function is smooth and differentiable. [S1, unofficial]
*   **Coarse Grid Search:** A uniform sampling of the ray interval to detect sign changes. It is robust but computationally expensive if $N_{steps}$ is large. Choose when the surface topology is complex or unknown. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** $O(B \cdot N_{rays} \cdot N_{steps})$ for the coarse sampling and network query. The secant refinement is $O(B \cdot N_{rays} \cdot N_{secant\_steps})$, which is typically negligible compared to the network query.
*   **Memory Complexity:** $O(B \cdot N_{rays} \cdot N_{steps})$ to store the coarse samples and logits.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of rays and coarse steps. The neural network query is the bottleneck.
*   **Strengths:** Handles implicit surfaces defined by neural networks efficiently. The cost-matrix approach allows vectorized sign-change detection.
*   **Limitations:** Relies on the coarse grid to find the initial bracket. If the surface is very thin or the grid is too sparse, sign changes may be missed. Assumes the ray starts outside the surface.
*   **When to use:** When rendering neural implicit surfaces where the output is logits rather than signed distances, and the surface is expected to be entered from outside.
*   **When not to use:** When the ray origin is inside the surface, or when the surface is extremely thin relative to the coarse step size.

## Background
L-Tracing was introduced in the context of neural implicit surface rendering, specifically for methods that use neural networks to represent surfaces via logits. The method is part of the L-Tracing framework, which aims to efficiently trace rays through these implicit representations. The approach leverages the properties of neural network outputs (logits) and uses a combination of coarse sampling and secant refinement to locate intersections.

## Sources
[S1] ziyc/L-Tracing, https://github.com/ziyc/L-Tracing, fetched 2026-10-26 (unofficial)
