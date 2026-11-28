# Inhomogeneity-Based Splitting

Also called: Inhomogeneity-Based Splitting for Training, Splitting during 3D Gaussian Learning

## Overview
Inhomogeneity-Based Splitting is a densification technique for 3D Gaussian Splatting that addresses the formation of "needle-like" artifacts and sparse point clouds in large, flat, untextured regions [S1, unofficial]. It sets itself apart from standard densification by explicitly measuring the ratio of principal scale eigenvalues to determine split necessity, rather than relying solely on position or opacity gradients [S1, unofficial].

## How it works
**Inputs:** A trained or partially trained set of 3D Gaussians, where each Gaussian $k$ is defined by center $\mu_k$, rotation $R_k$, scale $S_k$, opacity $\alpha_k$, and spherical harmonics coefficients. The method operates on the diagonalized scale/rotation matrix components, denoted as $\text{Tri}[i]$ for $i \in \{0, 1, 2\}$, which correspond to the squared scales along the principal axes [S1, unofficial].

**Outputs:** A modified set of Gaussians where those exceeding the inhomogeneity threshold are replaced by two new Gaussians.

**Data Structures:**
*   **Gaussian:** Standard 3DGS representation.
*   **Tri:** A vector of size 3 containing the diagonal elements of the covariance matrix's scale/rotation component, sorted or indexed by principal direction magnitude [S1, unofficial].

**Parameters:**
*   $\gamma_{th}$: Splitting threshold for the inhomogeneity ratio $\gamma$. Default value: 5 [S1, unofficial].
*   $\gamma_{th\_ij}$: Inhomogeneity threshold for pairwise eigenvalue ratios $y_{ij}$. Default value: 2 [S1, unofficial].
*   $R_m$: Regularization term in $y_{ij}$. Value not specified in the sources [S1, unofficial].
*   Iteration schedule: Check for splitting every 5000 iterations, starting from iteration 9999 and ending at 24999 [S1, unofficial].

**Steps:**
1.  **Compute Inhomogeneity Ratio ($\gamma$):**
    For each Gaussian, identify the maximum and second-maximum diagonal elements of the scale/rotation matrix:
    $$pd_m = \max_{i \in \{0,1,2\}} (\text{Tri}[i])$$
    $$pd_s = \text{second\_max}_{i \in \{0,1,2\}} (\text{Tri}[i])$$
    Calculate the ratio:
    $$\gamma = \frac{pd_m}{pd_s}$$
    [S1, unofficial]

2.  **Check Primary Threshold:**
    If $\gamma > \gamma_{th}$ (default 5), the Gaussian is a candidate for splitting [S1, unofficial].

3.  **Compute Pairwise Inhomogeneity ($y_{ij}$):**
    Calculate the degree of inhomogeneity between the $i$-th and $j$-th largest eigenvalues ($i < j$):
    $$y_{ij} = \frac{\text{Tri}[i]}{\text{Tri}[j] + R_m}$$
    where $(i, j) \in \{(0, 1), (1, 2), (0, 2)\}$ [S1, unofficial].

4.  **Check Pairwise Threshold:**
    If any $y_{ij} > \gamma_{th\_ij}$ (default 2), the Gaussian should be split at principal component $i$ [S1, unofficial].

5.  **Determine Split Plane Normal ($\mathbf{n}$):**
    Assume the split plane normal is parallel to the principal direction of the maximum eigenvalue:
    $$\mathbf{n} = R[\operatorname{argmax}(\operatorname{Tri}[i])]$$
    [S1, unofficial]

6.  **Generate New Gaussians:**
    Simplify the split by setting the split plane to pass through the Gaussian center ($\mu_0 = 0$ relative offset). Set the scaling factors for the new Gaussians to $\sigma_0 = \sigma_1 = 0.5$ and the opacity weight $\alpha = 1/\sqrt{2}$. This sets the first term of the new covariance matrices $\Sigma_0$ and $\Sigma_1$ to 0 to speed up calculation [S1, unofficial].
    
    To ensure the new Gaussians are sufficiently separated and avoid overlapping positions, set the position offset parameter $\mu_0$ as:
    $$\mu_0 = 2 \cdot \text{Tri}[i]$$
    where $i$ is the principal component index identified in Step 4 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inhomogeneity_based_splitting(gaussians, gamma_th=5, gamma_th_ij=2, R_m=0.0):
    """
    Splits Gaussians based on inhomogeneity of their principal scales.
    gaussians: List of Gaussian objects with attributes:
               .tri: list of 3 floats (diagonal scale/rotation elements)
               .R: 3x3 rotation matrix
               .mu: 3D center position
               .alpha: opacity
               .scale: scale parameters
    """
    new_gaussians = []
    gaussians_to_split = []

    for g in gaussians:
        tri = g.tri
        # Step 1: Compute pd_m and pd_s
        sorted_tri = sorted(tri, reverse=True)
        pd_m = sorted_tri[0]
        pd_s = sorted_tri[1]
        
        # Avoid division by zero
        if pd_s == 0:
            new_gaussians.append(g)
            continue

        gamma = pd_m / pd_s
        
        # Step 2: Check primary threshold
        if gamma <= gamma_th:
            new_gaussians.append(g)
            continue
            
        # Step 3: Compute pairwise inhomogeneity
        split_component = -1
        for i in range(3):
            for j in range(i + 1, 3):
                # Use original indices from sorted_tri for i and j
                # Note: The source defines y_ij based on Tri[i] and Tri[j]
                # where i < j refers to rank order (0 is largest)
                val_i = sorted_tri[i]
                val_j = sorted_tri[j]
                y_ij = val_i / (val_j + R_m)
                
                # Step 4: Check pairwise threshold
                if y_ij > gamma_th_ij:
                    split_component = i
                    break
            if split_component != -1:
                break
        
        if split_component != -1:
            gaussians_to_split.append((g, split_component))
        else:
            new_gaussians.append(g)

    # Step 5 & 6: Perform splits
    for g, comp_idx in gaussians_to_split:
        tri = g.tri
        sorted_tri = sorted(tri, reverse=True)
        pd_val = sorted_tri[comp_idx]
        
        # Determine normal direction
        # argmax returns the index in the original tri array? 
        # Source says: n = R[argmax(Tri[i])]. Assuming comp_idx refers to the rank.
        # We need the actual index in the original tri array corresponding to the comp_idx-th largest value.
        # For simplicity, assuming tri is already ordered or we map rank to index.
        # Let's assume we find the index k in original tri such that tri[k] is the comp_idx-th largest.
        # This requires sorting with indices.
        indexed_tri = sorted(enumerate(tri), key=lambda x: x[1], reverse=True)
        original_idx = indexed_tri[comp_idx][0]
        
        # Normal is the column of R corresponding to the principal axis
        n = g.R[:, original_idx]
        
        # Offset calculation
        # Source: mu_0 as 2*Tri[i]. Tri[i] here likely refers to the scale value pd_val.
        offset = 2.0 * pd_val * n
        
        # Create two new Gaussians
        # Center: mu +/- offset/2 ? 
        # Source says: mu_0 = 2*Tri[i] to ensure new Gaussians are far apart.
        # And split plane passes through center (mu_0=0 relative).
        # Standard splitting usually creates two Gaussians at mu +/- delta.
        # If mu_0 is the offset vector, the new centers are mu + mu_0/2 and mu - mu_0/2?
        # Or is mu_0 the total distance? "ensure new Gaussians are far apart".
        # Let's assume the new centers are mu + offset/2 and mu - offset/2.
        
        g1 = copy_gaussian(g)
        g2 = copy_gaussian(g)
        
        g1.mu = g.mu + offset / 2.0
        g2.mu = g.mu - offset / 2.0
        
        # Scale and Opacity adjustments
        # Source: sigma_0 = sigma_1 = 0.5, alpha = 1/sqrt(2)
        # This implies scaling the existing scale by 0.5 and splitting opacity.
        g1.scale = g.scale * 0.5
        g2.scale = g.scale * 0.5
        g1.alpha = g.alpha / 1.41421356 # 1/sqrt(2)
        g2.alpha = g.alpha / 1.41421356
        
        new_gaussians.append(g1)
        new_gaussians.append(g2)

    return new_gaussians
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $pd_s$ is zero, the ratio $\gamma$ is undefined. The implementation must handle this, typically by skipping the split or treating it as non-inhomogeneous [S1, unofficial].
*   **Needle Artifacts:** Using a threshold of 10 results in fewer needles than 5, but 5 is chosen to balance densification and artifact removal [S1, unofficial].
*   **Position Overlap:** Failing to set the offset $\mu_0$ appropriately (e.g., setting it to 0) causes new Gaussians to overlap, leading to redundancy. The source specifies $\mu_0 = 2 \cdot \text{Tri}[i]$ to prevent this [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is computed in parallel and "hardly increases the training time" compared to original 3D Gaussian Splatting [S1, unofficial].
*   **Memory Complexity:** Increases memory usage by replacing single Gaussians with pairs, effectively doubling the count for split candidates.
*   **Performance:** On the NeRF synthetic dataset (ship object), the method achieves PSNR 37.1908, SSIM 0.9839, and LPIPS 0.0216, compared to baseline 3D GS PSNR 36.9356, SSIM 0.9825, LPIPS 0.0236 [S1, unofficial].
*   **Strengths:** Reduces "needle-like bugles" and blurring in flat untextured regions; produces more uniform and denser point clouds for downstream tasks like normal extraction [S1, unofficial].
*   **Limitations:** Requires careful tuning of $\gamma_{th}$; may introduce artifacts if thresholds are too high (e.g., 10 still leaves some needles) [S1, unofficial].

## Background
The method was proposed in "A New Split Algorithm for 3D Gaussian Splatting" [S1, unofficial]. It addresses limitations in standard 3D Gaussian Splatting where structural inhomogeneity leads to uneven point clouds and artifacts. The approach integrates with the 3D Gaussian learning process, specifically targeting the extraction of uniform point clouds for downstream applications.

## Sources
[S1] A New Split Algorithm for 3D Gaussian Splatting, https://arxiv.org/abs/2403.09143v1, fetched 2026-10-26 (unofficial)
