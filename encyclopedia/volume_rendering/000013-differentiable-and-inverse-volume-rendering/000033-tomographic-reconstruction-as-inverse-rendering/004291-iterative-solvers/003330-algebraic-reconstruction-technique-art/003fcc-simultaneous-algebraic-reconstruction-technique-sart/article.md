# Simultaneous Algebraic Reconstruction Technique (SART)


## Overview
Simultaneous Algebraic Reconstruction Technique (SART) is an iterative algebraic method for solving the linear system $Ax = p$ in X-ray tomography, where $A$ is the system matrix, $x$ is the unknown volume, and $p$ is the measured sinogram [S1, unofficial]. It solves the inverse rendering problem of reconstructing a 3D volume from 2D projections by aggregating updates from a full projection image (or subset of rays) and applying them simultaneously [S1, unofficial]. Unlike its sibling [Algebraic Reconstruction Technique (ART)](6944-algebraic-reconstruction-technique-art.md), which updates the volume one ray at a time, SART processes all rays in a projection image concurrently, providing faster convergence and better stability against noise [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** System matrix $A \in \mathbb{R}^{m \times n}$ (where $m$ is the number of rays and $n$ is the number of voxels), projection vector $p \in \mathbb{R}^m$ (sinogram), relaxation parameter $\alpha$ (typically $0 < \alpha < 2$), and total iterations $T$ [S1, unofficial].
*   **Output:** Reconstructed volume vector $x \in \mathbb{R}^n$ [S1, unofficial].

**Data Structures:**
*   The system matrix $A$ is not stored explicitly due to memory constraints (e.g., a $512^3$ volume with 360 1.5-megapixel projections requires ~280 GB for single-precision $A$) [S1, unofficial]. Instead, $A$ is accessed implicitly via geometric ray tracing functions during forward and backprojection [S1, unofficial].
*   $a_{ij}$ represents the contribution of voxel $j$ to ray $i$, computed as the intersection length or weight [S1, unofficial].
*   $x^{(t)}$ is the volume estimate at iteration $t$.
*   $\hat{p}^{(t)}$ is the estimated projection vector at iteration $t$.
*   $c^{(t)}$ is the normalized correction factor vector at iteration $t$.

**Algorithm Steps:**
1.  **Initialize:** Set the initial volume estimate $x^{(0)} = 0$ [S1, unofficial].
2.  **Iterate:** For each iteration $t = 1$ to $T$:
    3.  **Loop over Projections:** For each projection image $S$ in the set of all projection images $\mathcal{S}_{all}$:
        4.  **Forward Projection:** For each ray $i$ in the current subset $S$, compute the estimated projection:
            $$ \hat{p}_i^{(t)} = \sum_{k} a_{ik} x_k^{(t)} $$
            where the sum is over all voxels $k$ contributing to ray $i$ [S1, unofficial].
        5.  **Correction:** For each ray $i$ in the current subset $S$, compute the normalized correction factor:
            $$ c_i^{(t)} = \frac{p_i - \hat{p}_i^{(t)}}{\sum_{k} a_{ik}} $$
            The denominator $\sum_{k} a_{ik}$ is the sum of contributions of ray $i$ to all voxels [S1, unofficial].
        6.  **Backprojection:** Update all voxels $j = 1 \dots n$ using the aggregated corrections from the current subset $S$:
            $$ x_j^{(t+1)} = x_j^{(t)} + \alpha \frac{\sum_{i \in \mathcal{S}} c_i^{(t)} a_{ij}}{\sum_{i \in \mathcal{S}} a_{ij}} $$
            The numerator aggregates the weighted corrections from all rays $i$ in subset $S$ that pass through voxel $j$. The denominator normalizes by the total weight of rays in $S$ passing through voxel $j$ [S1, unofficial].
    7.  End loop over projections.
8.  End loop over iterations.
9.  Return $x^{(T)}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sart(A_getter, p, alpha, T, num_voxels, num_rays, projection_sets):
    """
    A_getter: function(i, j) -> float, returns a_ij. 
              Called implicitly for forward/backprojection.
    p: array of shape (num_rays,)
    alpha: float, relaxation parameter (0 < alpha < 2)
    T: int, number of iterations
    num_voxels: int, n
    num_rays: int, m
    projection_sets: list of lists, where each inner list contains indices of rays 
                     for one projection image.
    """
    # Initialize volume estimate
    x = [0.0] * num_voxels
    
    for t in range(1, T + 1):
        # Create a temporary copy for the update to ensure consistency within iteration
        # Note: Standard SART updates x in place using the current x(t) for all rays in S
        # before moving to the next S. The formula uses x^(t) for forward projection
        # and updates x^(t+1) based on aggregated corrections.
        # However, the standard algorithm description implies x is updated sequentially 
        # across subsets S within the same iteration t, or that x^(t) is fixed for the 
        # forward projection of all S. 
        # The source [S1, unofficial] says: "x_j^(t+1) = x_j^(t) + ...". This implies x^(t) is the 
        # value at the start of the iteration t. 
        # But the loop structure in Algorithm 1 shows x_j^(t+1) being updated inside 
        # the loop over S. This suggests that for the first S, x is updated to an 
        # intermediate state, and subsequent S use this intermediate state? 
        # Or does it mean x^(t) is used for ALL forward projections in iteration t, 
        # and the updates are accumulated into x^(t+1)?
        # Let's look at Eq 2.1: x_j^(t+1) = x_j^(t) + alpha * ...
        # The RHS uses c_i^(t). c_i^(t) uses p_i - p_hat_i^(t). 
        # p_hat_i^(t) uses x_k^(t).
        # If x is updated in place inside the loop over S, then for the second S, 
        # x is no longer x^(t). 
        # Standard SART implementations usually accumulate the backprojection term 
        # for all rays in the iteration before updating x. 
        # However, Algorithm 1 in [S1, unofficial] writes:
        # for all S:
        #   compute p_hat, c
        #   for j: x_j^(t+1) = x_j^(t) + ...
        # This notation x_j^(t+1) suggests a separate buffer or that the update is 
        # based on x^(t). If it were in-place, it would likely be x_j = x_j + ...
        # Given the "simultaneous" nature and the formula structure, it is safest 
        # to assume x^(t) is fixed for the forward projection of all rays in iteration t,
        # and the backprojection updates are accumulated. 
        # But Algorithm 1 explicitly puts the x update inside the S loop.
        # Let's stick to the literal Algorithm 1 structure from [S1, unofficial], but note that 
        # x_j^(t) on the RHS must refer to the value at the START of iteration t.
        
        # To implement strictly as per Algorithm 1 in [S1, unofficial]:
        # We need to store x^(t) before the S loop.
        x_prev = x[:] # Copy of x at start of iteration t
        
        for S in projection_sets:
            # Forward Projection for rays in S
            # Compute p_hat_i for i in S
            p_hat = [0.0] * len(S)
            ray_sum_denom = [0.0] * len(S) # Sum of a_ik for each ray i
            
            for idx, i in enumerate(S):
                # Compute p_hat_i = sum_k a_ik * x_k^(t)
                # And sum_k a_ik
                p_hat_val = 0.0
                ray_sum = 0.0
                for k in range(num_voxels):
                    a_ik = A_getter(i, k)
                    p_hat_val += a_ik * x_prev[k]
                    ray_sum += a_ik
                p_hat[idx] = p_hat_val
                ray_sum_denom[idx] = ray_sum

            # Correction
            c = [0.0] * len(S)
            for idx, i in enumerate(S):
                p_i = p[i]
                c[idx] = (p_i - p_hat[idx]) / ray_sum_denom[idx]

            # Backprojection
            # Update x_j^(t+1)
            # We accumulate the update term for each voxel
            # Note: The formula is x_j^(t+1) = x_j^(t) + alpha * (sum_{i in S} c_i a_ij) / (sum_{i in S} a_ij)
            # This implies the update is specific to the subset S.
            # If we have multiple S in one iteration t, do we add them up?
            # Algorithm 1 says: for all S: ... x_j^(t+1) = ...
            # This suggests that x is updated sequentially for each S.
            # However, the notation x_j^(t+1) is confusing if updated multiple times.
            # Usually, SART iterates over subsets (Ordered Subsets). 
            # If S_all is the set of ALL projection images, and we loop over them,
            # then each S is a full projection image.
            # The update equation divides by sum_{i in S} a_ij.
            # If we update x in place, the next S uses the updated x.
            # But the forward projection uses x^(t).
            # So we must use x_prev for forward projection, but update x in place?
            # Or update a new x_next?
            # Let's assume x is updated in place, but forward projection always uses x_prev.
            
            # Precompute voxel sums for denominator of backprojection
            voxel_ray_sum = [0.0] * num_voxels
            for idx, i in enumerate(S):
                for k in range(num_voxels):
                    a_ik = A_getter(i, k)
                    voxel_ray_sum[k] += a_ik
            
            # Apply update
            for j in range(num_voxels):
                if voxel_ray_sum[j] == 0:
                    continue
                numerator = 0.0
                for idx, i in enumerate(S):
                    a_ij = A_getter(i, j)
                    numerator += c[idx] * a_ij
                x[j] = x_prev[j] + alpha * (numerator / voxel_ray_sum[j])
                
        # End of iteration t
        # x is now x^(t+1)
        
    return x
```

**Edge Cases and Pitfalls:**
*   **Memory:** The system matrix $A$ is too large to store explicitly. Implementations must be "matrix-free," computing $a_{ij}$ on the fly using ray-voxel intersection geometry [S1, unofficial].
*   **Under-determined Systems:** If $m < n$ (fewer rays than voxels), SART is assumed to solve the minimum norm problem $\min_x \|x\|_2^2$ subject to $Ax = p$, similar to ART, though no formal convergence proof exists for the general case [S1, unofficial].
*   **Relaxation Parameter:** $\alpha$ should typically be in the range $(0, 2)$ for stability [S1, unofficial].

## Subtopics

### [SART Proximal Operator](10122-sart-proximal-operator.md)
Modifies SART to incorporate a proximal operator for regularization and prior information, iterating on auxiliary variables $y$ and $x$. Choose when prior knowledge or regularization is needed.

### [Motion-Compensated Algebraic Reconstruction Technique](7422-motion-compensated-algebraic-reconstruction-technique.md)
Variant for dynamic tomography. Choose when object motion is present.

### [Direct Linear Inversion (ART)](8506-direct-linear-inversion-art.md)
Single-ray update variant. Choose when memory is extremely constrained or for theoretical comparison.

## Operating characteristics
*   **Time Complexity:** Each iteration involves a forward projection and a backprojection for all rays. Complexity is proportional to $T \times m \times (\text{avg voxels per ray})$.
*   **Memory Complexity:** $O(n + m)$ for storing $x$, $p$, and intermediate buffers. $O(1)$ for $A$ (matrix-free) [S1, unofficial].
*   **Performance:** Faster convergence and better noise resilience than ART [S1, unofficial]. Amenable to parallelization on graphics hardware [S1, unofficial].
*   **Strengths:** Intuitive geometric interpretation, better stability against noise, faster convergence than ART [S1, unofficial].
*   **Limitations:** No formal convergence proof for the general case [S1, unofficial]. High computational cost per iteration compared to analytical methods like FDK [S1, unofficial].
*   **When to use:** When high-quality reconstruction is needed and computational resources allow for iterative methods; when noise resilience is critical [S1, unofficial].
*   **When not to use:** When real-time performance is required and analytical methods suffice; when memory for $x$ and $p$ is extremely limited (though less critical than ART's sequential nature).

## Background
SART is based on the Algebraic Reconstruction Technique (ART), which dates back to Kaczmarz in 1937 [S1, unofficial]. It was developed to improve upon ART by aggregating updates from multiple rays, providing better stability and convergence [S1, unofficial]. It is implemented in the Reconstruction ToolKit (RTK) and the ASTRA toolbox (though ASTRA lacks 3D SART support according to the text) [S1, unofficial].

## Sources
[S1] Industrial Computed Tomography using Proximal Algorithm, https://doi.org/10.25781/kaust-th10i, fetched 2026-10-27 (unofficial)
