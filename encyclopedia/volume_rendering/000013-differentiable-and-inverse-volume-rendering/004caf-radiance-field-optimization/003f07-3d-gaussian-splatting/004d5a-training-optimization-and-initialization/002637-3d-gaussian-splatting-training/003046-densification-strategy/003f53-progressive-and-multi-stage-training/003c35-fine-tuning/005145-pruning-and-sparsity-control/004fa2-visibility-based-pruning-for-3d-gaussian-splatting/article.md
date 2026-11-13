# Visibility-Based Pruning for 3D Gaussian Splatting

Also called: Visibility Pruning within 3DGS, Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations

## Overview
Visibility-Based Pruning removes occluded 3D Gaussian kernels during the optimization pipeline to create a compact, "hard" geometry representation by concentrating alpha values into remaining kernels [S1]. It distinguishes itself from sibling methods like [Gaussian Volume Mask](12710-gaussian-volume-mask.md) by explicitly computing geometric occlusion via a visibility mask against training views, rather than relying solely on scalar thresholds of opacity or gradient magnitude [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Set of $N$ 3D Gaussians $M$, set of $K$ training viewpoints $V$, thickness parameter $\tau$, Minimum Viewpoint Exposure (MVE) threshold $\nu$, and regularization hyperparameter $\beta$.
*   **Outputs:** Pruned set of Gaussians $M'$ and updated visibility mask $O$.

**Data Structures:**
*   **Gaussian Representation:** Each Gaussian $i$ is an ellipsoid defined by a covariance matrix $\Sigma$, decomposed into a scaling matrix $S_i$ and a rotation matrix $R_i$ [S1]. The position is defined by the centroid $P[i]$ [S1].
*   **Visibility Mask:** A binary matrix $O$ of size $N \times K$, where $O_{ij} = 1$ if Gaussian $i$ is visible from viewpoint $j$, and $0$ otherwise [S1].

**Algorithm Steps:**
1.  **Visibility Mask Computation:** For each viewpoint $j \in V$:
    a. Retrieve extrinsic matrix $d$ and intrinsic matrix $K$ for viewpoint $j$ [S1].
    b. Initialize a list $M$ to store points projected to each pixel [S1].
    c. For each Gaussian $i \in M$:
        i. Project centroid $P[i]$ to 2D coordinates $(u, v)$ using forward projection with $d$ and $K$ [S1].
        ii. Add $P[i]$ to the pixel-specific list $M_{u,v}$ [S1].
    d. For each pixel $(u, v)$ with non-empty $M_{u,v}$:
        i. Sort $M_{u,v}$ by depth $z_i$ (ascending) [S1].
        ii. Select the first $\tau$ points from the sorted list into $M^*$ [S1].
    e. For each Gaussian $i \in M$:
        i. If $P[i] \in M^*$, set $O_{ij} = 1$; else $O_{ij} = 0$ [S1].
2.  **Pruning Decision:** For each Gaussian $i$:
    a. Compute the sum of visible viewpoints: $sum_i = \sum_{k=1}^{K} O_{ik}$ [S1].
    b. If $sum_i \le \nu$, remove Gaussian $i$ from the scene [S1].
3.  **Regularization:** During training, add a regularization term to the loss function to prioritize the pruning of smaller Gaussians [S1]:
    $$ L_{reg} = \frac{\beta}{N} \sum_{i=1}^{N} \frac{1}{\|S_i\|} $$
    where $\|S_i\|$ is the norm of the scaling matrix of the $i$-th Gaussian [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ComputeVisibilityMask(points, viewpoints, thickness):
    """
    Computes the visibility mask O for a set of points and viewpoints.
    points: List of 3D centroids [N, 3]
    viewpoints: List of camera parameters containing extrinsic and intrinsic matrices [K]
    thickness: Integer tau, number of front-most points to consider visible
    Returns: O, a binary matrix [N, K]
    """
    N = len(points)
    K = len(viewpoints)
    O = [[0 for _ in range(K)] for _ in range(N)]
    
    for j in range(K):
        c = viewpoints[j]
        d = c.extrinsic_matrix
        K_mat = c.intrinsic_matrix
        
        # Group points by pixel
        pixel_points = {} # Key: (u, v), Value: List of (point, z)
        
        for i in range(N):
            p = points[i]
            # Forward projection: 3D to 2D
            u, v = forward_projection(p, d, K_mat)
            
            if (u, v) not in pixel_points:
                pixel_points[(u, v)] = []
            
            # Store point and its depth z
            z = p[2] # Assuming z is the depth component in camera space
            pixel_points[(u, v)].append((p, z))
            
        # Process each pixel
        for (u, v), plist in pixel_points.items():
            # Sort by depth z ascending
            plist.sort(key=lambda x: x[1])
            
            # Pick first 'thickness' points
            visible_points = plist[:thickness]
            visible_set = set(p for p, z in visible_points)
            
            # Mark visibility in mask
            for i in range(N):
                if points[i] in visible_set:
                    O[i][j] = 1
                    
    return O

def VisibilityPruningStep(gaussians, viewpoints, tau, nu):
    """
    Performs visibility-based pruning on a set of Gaussians.
    gaussians: List of Gaussian objects with attributes .centroid, .scaling_matrix
    viewpoints: List of camera parameters
    tau: Thickness parameter
    nu: MVE threshold
    Returns: Pruned list of Gaussians
    """
    N = len(gaussians)
    points = [g.centroid for g in gaussians]
    
    # Compute visibility mask
    O = ComputeVisibilityMask(points, viewpoints, tau)
    
    pruned_gaussians = []
    
    for i in range(N):
        # Sum visible viewpoints
        sum_visible = sum(O[i])
        
        # Check MVE threshold
        if sum_visible > nu:
            pruned_gaussians.append(gaussians[i])
            
    return pruned_gaussians

def ComputeLossWithRegularization(rendered, target, gaussians, beta, lambda_dssim):
    """
    Computes the total loss including visibility-based regularization.
    """
    # Standard 3DGS losses
    L1 = compute_l1_loss(rendered, target)
    DSSIM = compute_dssim_loss(rendered, target)
    
    # Regularization term
    N = len(gaussians)
    reg_sum = 0
    for g in gaussians:
        # Norm of scaling matrix S_i
        s_norm = norm(g.scaling_matrix)
        if s_norm > 0:
            reg_sum += 1.0 / s_norm
            
    L_reg = (beta / N) * reg_sum
    
    # Combined loss (assuming lambda is for DSSIM weight as per source context)
    # Source Eq 7: L = (1 - lambda)L1 + lambda LD-SSIM + L_reg
    # Note: Source uses lambda for DSSIM weight.
    lambda_val = 0.2 # Typical value from source
    total_loss = (1 - lambda_val) * L1 + lambda_val * DSSIM + L_reg
    
    return total_loss
```

**Parameters:**
*   $\beta$: Hyperparameter for the scaling regularization term, typically $1 \times 10^{-5}$ [S1].
*   $\nu$: MVE threshold parameter determining if a Gaussian is sufficiently occluded to be removed [S1].
*   $\tau$: Thickness parameter in visibility computation, defining how many points are picked from the sorted depth list [S1].
*   $\lambda$: Weight for D-SSIM term in loss, typically $0.2$ [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Forward projection must handle floating-point precision for pixel coordinates $(u, v)$ to correctly group points [S1].
*   **Boundaries:** Points projected outside the image bounds should be ignored or handled as not visible [S1].
*   **Degenerate Input:** If $\|S_i\| = 0$, the regularization term becomes undefined; a small epsilon should be added to the denominator [G].

## Operating characteristics
*   **Time Complexity:** Visibility mask computation is $O(N \cdot K \cdot \log N)$ due to sorting points per pixel, where $N$ is the number of Gaussians and $K$ is the number of views [S1].
*   **Memory Complexity:** Requires storing the $N \times K$ visibility mask $O$ [S1].
*   **Performance:** The method reduces the number of kernels during training, allowing remaining Gaussians to optimize shape and color attributes more effectively [S1]. It results in a "harder" geometry representation [S1].
*   **Strengths:** Explicitly handles occlusion; concentrates alpha values into visible kernels [S1].
*   **Limitations:** Requires visibility computation at pruning intervals, adding overhead to the training loop [S1].
*   **When to use:** When a compact, hard geometry representation is desired and occlusion is a significant factor in the scene [S1].

## Background
The method was proposed by Niedermayr and Lee in their work on "Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations" [S1]. It is designed to be embedded into the 3DGS optimization pipeline, specifically within the Adaptive Density Control block, performing visibility computation and removal jointly with densification steps [S1]. The approach was benchmarked against other methods like PlenOctrees and NeRF variants [S1].

## Sources
[S1] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
