# Cone Loss


## Overview
The Cone Loss is a geometric regularization term used in ZipSplat to ground 3D Gaussian tokens to their specific source image patches, preventing decoded Gaussians from drifting arbitrarily in 3D space [S1, unofficial]. Unlike frustum-based constraints that merely ensure visibility, this loss explicitly ties each token’s geometry to the observation that produced it, ensuring an observation-grounded scene representation [S1, unofficial]. It operates alongside the GeoCE Loss, which aligns token similarity with 3D proximity so that clusters form by surface rather than by viewpoint, while the Cone Loss anchors individual tokens to their source observations [S1, unofficial].

## How it works
**Inputs and Outputs**
*   $K$: Total number of scene tokens [S1, unofficial].
*   $\ngs$: Number of Gaussians per token [S1, unofficial].
*   $\gmean_{kj}$: The 3D mean position of the $j$-th Gaussian belonging to token $k$ [S1, unofficial].
*   $\pi_k$: The projection function mapping 3D points into the 2D image plane of token $k$'s source view [S1, unofficial].
*   $\*{u}_k$: The 2D center coordinate of the source image patch for token $k$ [S1, unofficial].
*   Output: A scalar loss value $\loss_{\mathrm{cone}}$ representing the total reprojection error [S1, unofficial].

**Data Structures**
*   Token indices $k \in \{1, \dots, K\}$ [S1, unofficial].
*   Gaussian indices $j \in \{1, \dots, \ngs\}$ [S1, unofficial].
*   Coordinates are measured in "patch units" (where the patch width/height is normalized to 1 unit) [S1, unofficial].

**Algorithm**
The loss is computed by summing a robust penalty $\rho$ over all Gaussians of all tokens [S1, unofficial]. The penalty is based on the Euclidean distance between the projected Gaussian center and the source patch center [S1, unofficial].

1. For each token $k$ from $1$ to $K$: [S1, unofficial]
2. For each Gaussian $j$ from $1$ to $\ngs$: [S1, unofficial]
3. Project the Gaussian mean $\gmean_{kj}$ into the source view of token $k$ using $\pi_k$ to obtain the 2D point $\mathbf{p} = \pi_k(\gmean_{kj})$ [S1, unofficial].
4. Check if the Gaussian projects behind the camera plane of the source view [S1, unofficial].
    * If yes, assign a sentinel distance $d = 20$ patch units [S1, unofficial].
    * If no, compute the Euclidean distance in patch units: $d = \| \mathbf{p} - \*{u}_k \|$ [S1, unofficial].
5. Apply the hinged Cauchy penalty function $\rho(d)$ to $d$ [S1, unofficial].
6. Accumulate $\rho(d)$ into the total loss [S1, unofficial].

**Formula**
The total Cone Loss is defined as [S1, unofficial]:

$$
\loss_{\mathrm{cone}} = \sum_{k=1}^{K} \sum_{j=1}^{\ngs} \rho \left( \left\| \pi_k(\gmean_{kj}) - \*{u}_k \right\| \right)
$$

Where:
*   $\pi_k(\gmean_{kj})$ is the 2D projection of the $j$-th Gaussian of token $k$ into its source view [S1, unofficial].
*   $\*{u}_k$ is the center of the source patch for token $k$ [S1, unofficial].
*   The norm $\|\cdot\|$ is the Euclidean distance measured in patch units [S1, unofficial].
*   $\rho$ is a robust penalty function (hinged Cauchy) [S1, unofficial].

**Robust Penalty Function $\rho$**
The function $\rho(d)$ is a hinged Cauchy function of the distance $d$ in patch units [S1, unofficial]. It has the following properties:
1.  **Hinge (Zero-cost region):** For $d \le 2$, $\rho(d) = 0$ [S1, unofficial]. This allows Gaussians to extend up to 2 patch units past their source observation, accommodating flat surfaces or occluded regions [S1, unofficial].
2.  **Smooth Ramp:** For $2 < d \le 4$, the penalty ramps smoothly with $C^1$ continuity [S1, unofficial].
3.  **Saturation:** For $d > 4$, the penalty saturates to limit the influence of distant outliers [S1, unofficial].

**Parameters**
*   Weight $\lambda_{\mathrm{cone}} = 0.01$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Behind-view Gaussians:** Gaussians projecting behind the source view receive a fixed sentinel distance of 20 patches [S1, unofficial]. This provides a large penalty value but no lateral gradient, preventing optimization from pushing these Gaussians laterally into the view [S1, unofficial].
*   **Depth Independence:** The loss constrains only the lateral position relative to the viewing cone [S1, unofficial]. Depth is left to be determined by the rendering loss [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cone_loss(gmeans, projections, patch_centers, num_tokens, num_gs):
    """
    Computes the Cone Loss for ZipSplat.
    
    Args:
        gmeans: Array of shape (num_tokens, num_gs, 3), 3D means of Gaussians.
        projections: Array of shape (num_tokens, num_gs, 2), 2D projected coordinates.
        patch_centers: Array of shape (num_tokens, 2), centers of source patches.
        num_tokens: Integer K.
        num_gs: Integer n_gs.
        
    Returns:
        Scalar loss value.
    """
    total_loss = 0.0
    for k in range(num_tokens):
        for j in range(num_gs):
            # Get projected 2D point and patch center
            proj_2d = projections[k, j]
            u_k = patch_centers[k]
            
            # Check if Gaussian projects behind the camera plane
            # [G] The source does not specify the exact visibility check implementation.
            # Replace with camera-space z-depth test (e.g., z <= 0 in right-handed convention).
            is_behind = False  # Placeholder for implementer's camera model
            
            if is_behind:
                d = 20.0
            else:
                # Euclidean distance in 2D patch units
                dx = proj_2d[0] - u_k[0]
                dy = proj_2d[1] - u_k[1]
                d = math.sqrt(dx * dx + dy * dy)
            
            # Apply hinged Cauchy penalty
            # [G] The exact algebraic coefficients for the "hinged Cauchy" function are not provided in the sources.
            # Standard Cauchy robust loss: L(x) = c^2 * log(1 + (x/c)^2).
            # We implement a piecewise function matching the described behavior:
            # hinge at 2, C1 smooth ramp to 4, saturation past 4.
            if d <= 2.0:
                rho = 0.0
            elif d <= 4.0:
                # Smooth C1 ramp from 0 at d=2 to saturation value at d=4
                t = (d - 2.0) / 2.0  # normalized to [0, 1]
                rho = t * t          # ramps to 1.0 at d=4
            else:
                # Saturated past 4 patches
                rho = 1.0
            
            total_loss += rho
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** Linear in the number of tokens $K$ and the number of Gaussians per token $\ngs$ [S1, unofficial].
*   **Memory Complexity:** Minimal, requiring only scalar accumulation beyond the input arrays [S1, unofficial].
*   **Strengths:** Prevents Gaussian drift from source observations and allows Gaussians to extend up to 2 patch units to cover flat surfaces or occluded regions without penalty [S1, unofficial].
*   **Limitations:** Does not constrain depth, relying on the rendering loss for depth accuracy, and provides no lateral gradient for Gaussians behind the view plane [S1, unofficial].
*   **When to use:** When training ZipSplat or similar token-based Gaussian splatting methods that require observation-grounded geometry [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The Cone Loss was introduced in the ZipSplat paper by Veicht et al. (2026) [S1, unofficial]. It addresses the issue that without geometric supervision, decoded Gaussians in token-based representations can drift from their source patches, leading to merging errors where tokens from different surfaces are combined [S1, unofficial]. It is part of a suite of geometric priors including the GeoCE Loss and TokenBA, designed to improve novel view synthesis performance on datasets like DL3DV and RealEstate10K [S1, unofficial].

## Sources
[S1] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
