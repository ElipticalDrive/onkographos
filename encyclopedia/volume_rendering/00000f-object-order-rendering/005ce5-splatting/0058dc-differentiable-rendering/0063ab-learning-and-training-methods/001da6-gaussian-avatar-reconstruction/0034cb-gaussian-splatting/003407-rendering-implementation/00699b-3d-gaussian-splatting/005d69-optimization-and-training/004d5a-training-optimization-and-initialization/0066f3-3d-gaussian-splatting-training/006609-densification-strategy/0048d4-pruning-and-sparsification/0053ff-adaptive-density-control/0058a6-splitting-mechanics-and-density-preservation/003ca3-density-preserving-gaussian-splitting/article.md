# Density-preserving Gaussian splitting

Also called: BOGausS splitting, SplitAdaptation

## Overview
Density-preserving Gaussian splitting is a densification strategy for 3D Gaussian Splatting that modifies child parameters to ensure the sum of their densities approximates the parent's density, thereby preventing loss function perturbations and Gaussian clustering [S1]. Unlike standard 3DGS splitting which may cause instability, this method discards cloning operations entirely and relies solely on splitting with learned opacity and size adaptations [S1].

## How it works
**Inputs:**
*   Parent Gaussian parameters: mean $\mu_{p}$, scale $\sigma_{p}$ (vector), rotation $R_{p}$, opacity $o_{p}$, and spherical harmonic coefficients $C_{p}$.
*   Splitting constant $\alpha = 0.3$ [S1].
*   Learned adaptation function $SplitAdaptation$ [S1].

**Outputs:**
*   Two child Gaussians with updated means, scales, opacities, and inherited rotation/SH coefficients.

**Data Structures:**
*   Gaussian parameters are stored as standard 3DGS attributes: 3D mean, 3D scale (log-space or linear), 4D quaternion rotation, scalar opacity, and SH coefficients.

**Steps:**
1.  **Axis Selection:** Identify the principal axis with the largest scale value $\sigma_{max}$ of the parent Gaussian [Parent].
2.  **Mean Shift:** Calculate the shift distance $\Delta = \alpha \cdot \sigma_{max}$ [S1]. Create two new means $\mu_{c1}$ and $\mu_{c2}$ by shifting the parent mean $\mu_{p}$ along the selected axis by $+\Delta$ and $-\Delta$ respectively [S1].
3.  **Parameter Adaptation:** Compute the child scale $\sigma_{c}$ and opacity $o_{c}$ using the learned offline function:
    $$(\sigma_{c}, o_{c}) = SplitAdaptation(\sigma_{p}, o_{p})$$
    This function ensures the resulting density of the sum of the two children is close to the density of the mother Gaussian [S1].
4.  **Inheritance:** Copy the parent's rotation $R_{p}$ and spherical harmonic coefficients $C_{p}$ to both children [Parent].
5.  **Cloning Discard:** Do not perform cloning operations; rely exclusively on splitting for densification [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def density_preserving_split(parent_gaussian, alpha=0.3):
    """
    Performs density-preserving splitting on a parent Gaussian.
    
    Args:
        parent_gaussian: Object containing mean (mu), scale (sigma), 
                         rotation (R), opacity (o), and SH coefficients (C).
        alpha: Constant for mean shift magnitude (default 0.3).
    
    Returns:
        List of two child Gaussian objects.
    """
    # 1. Axis Selection: Find axis with largest scale
    # sigma is typically a vector [sx, sy, sz]
    max_scale_idx = argmax(parent_gaussian.sigma)
    sigma_max = parent_gaussian.sigma[max_scale_idx]
    
    # 2. Mean Shift
    # Create basis vector for the selected axis
    axis_vector = [0, 0, 0]
    axis_vector[max_scale_idx] = 1.0
    
    shift_distance = alpha * sigma_max
    
    # Shift means by +shift and -shift along the axis
    mu_c1 = parent_gaussian.mu + (axis_vector * shift_distance)
    mu_c2 = parent_gaussian.mu - (axis_vector * shift_distance)
    
    # 3. Parameter Adaptation
    # Apply learned offline adaptation function
    # Note: SplitAdaptation is a learned function, not a simple formula
    sigma_c, o_c = SplitAdaptation(parent_gaussian.sigma, parent_gaussian.o)
    
    # 4. Inheritance
    # Rotation and SH coefficients are copied directly
    R_c = parent_gaussian.R
    C_c = parent_gaussian.C
    
    # 5. Construct Children
    child1 = Gaussian(
        mu=mu_c1,
        sigma=sigma_c,
        R=R_c,
        o=o_c,
        C=C_c
    )
    
    child2 = Gaussian(
        mu=mu_c2,
        sigma=sigma_c,
        R=R_c,
        o=o_c,
        C=C_c
    )
    
    return [child1, child2]
```

**Edge Cases and Pitfalls:**
*   **Loss Perturbation:** Original splitting operations cause perturbations in the loss function. This method mitigates this by adapting parameters [S1].
*   **Clustering:** The method prevents clusters of Gaussians, a problem observed in other splitting strategies [S1].
*   **Opacity Variation:** Unlike some similar mechanisms, this method explicitly accounts for opacity variations during adaptation [S1].

## Operating characteristics
*   **Complexity:** The splitting operation itself is $O(1)$ per Gaussian. The adaptation function $SplitAdaptation$ is learned offline, so its runtime cost during training is negligible (typically a lookup or simple neural network inference) [S1].
*   **Performance:** By discarding cloning operations, the method simplifies the densification process and reduces the risk of inconsistent results in indoor and outdoor scenes [S1]. It replaces opacity reset with pruning based on an opacity threshold (e.g., 1% of total Gaussians) [S1].
*   **Strengths:** Prevents loss perturbations and Gaussian clustering; handles opacity variations [S1].
*   **Limitations:** Requires offline learning of the adaptation function; discards cloning which may limit some forms of densification flexibility [S1].

## Background
This method is part of the BOGausS (Better Optimized Gaussian Splatting) framework [S1]. It addresses issues identified in original 3DGS splitting, such as loss perturbations and clustering, by modifying the splitting operation to preserve density [S1]. It builds on ideas from [16] and contrasts with [17] by incorporating opacity variations. The approach is inspired by rate-distortion optimization principles for prioritizing splits [S1].

## Sources
[S1] BOGausS: Better Optimized Gaussian Splatting, https://doi.org/10.23919/eusipco63237.2025.11226626, fetched 2026-10-26
