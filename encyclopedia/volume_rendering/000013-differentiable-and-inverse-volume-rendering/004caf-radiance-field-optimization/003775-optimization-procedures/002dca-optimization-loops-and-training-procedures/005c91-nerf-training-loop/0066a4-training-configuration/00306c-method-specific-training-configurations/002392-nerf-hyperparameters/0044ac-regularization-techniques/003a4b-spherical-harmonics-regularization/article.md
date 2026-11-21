# Spherical Harmonics Regularization

Also called: SH coefficient masking, SH curriculum training, SH degree regularization

## Overview
Spherical Harmonics (SH) Regularization is a training strategy that restricts the maximum degree of Spherical Harmonics during the early stages of optimization to prevent high-frequency overfitting and stabilize geometry reconstruction [S1, S2]. It addresses the problem where high-frequency SH coefficients cause inaccurate geometry and "floaters" (noise artifacts) when the model attempts to learn view-dependent effects before the coarse geometry is established [S2]. It sets itself apart from general density constraints by specifically targeting the frequency content of the radiance representation, acting as a curriculum learning mechanism for view-dependent appearance [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid representation of the scene where each voxel stores opacity and Spherical Harmonics coefficients for color/radiance. The SH coefficients are typically organized by degree $l$ and order $m$.
*   **Output:** A trained radiance field with stabilized geometry and reduced high-frequency artifacts, achieved by gradually unlocking high-frequency SH coefficients during training.

**Data Structures:**
The radiance field uses a voxel grid. Each voxel $v$ contains:
*   Opacity $\alpha_v$ (scalar).
*   SH Coefficients $c_{l,m}$ for color channels. For a maximum degree $L$, there are $(L+1)^2$ coefficients per color channel. The coefficients are indexed by degree $l \in \{0, \dots, L\}$ and order $m \in \{-l, \dots, l\}$.

**Parameters:**
*   $L_{max}$: The maximum SH degree supported by the representation (e.g., $L_{max}=3$ or $4$).
*   $E_{curriculum}$: The number of epochs for the curriculum schedule. Sources specify $E_{curriculum}=5$ [S1].
*   $L_{start}$: The initial maximum SH degree allowed. Sources imply starting with low degrees (e.g., $l=0$ or $l=1$) to capture diffuse reflectance [S1].
*   Masking Rate Decay: The rate at which the masking constraint is relaxed. Sources specify a decay of $1/5$ per epoch for the first 5 epochs [S1].

**Steps:**
1.  **Initialization:** Initialize the voxel grid with SH coefficients. Set the current maximum allowed SH degree $L_{curr} = L_{start}$ (typically $0$ or $1$) [S1, S2].
2.  **Masking Definition:** Define a binary mask $M_{l,m}$ for each SH coefficient. If $l > L_{curr}$, then $M_{l,m} = 0$ (masked); otherwise, $M_{l,m} = 1$ [S1].
3.  **Gradient Application:** During backpropagation, compute gradients $\nabla c_{l,m}$ for all coefficients. Apply the mask to the gradients or the coefficients themselves:
    $$ c_{l,m}^{new} = c_{l,m}^{old} + \eta \cdot M_{l,m} \cdot \nabla c_{l,m} $$
    where $\eta$ is the learning rate. This ensures that coefficients with $l > L_{curr}$ do not update [S1].
4.  **Curriculum Schedule:** For each epoch $e$:
    *   If $e < E_{curriculum}$: Gradually increase $L_{curr}$. Sources describe decreasing the "rate of SH masking" by $1/5$ per epoch. This implies a linear relaxation of the constraint. Specifically, if the mask initially blocks all coefficients above a certain degree, the threshold for $L_{curr}$ increases. A precise interpretation from [S1] is that the masking is relaxed such that by epoch 5, full SH degrees are allowed.
    *   If $e \ge E_{curriculum}$: Set $L_{curr} = L_{max}$. All SH coefficients are updated with full rate [S1].
5.  **Optimization:** Continue optimizing opacity, SH coefficients, and other parameters (e.g., white balance) using the standard reconstruction loss [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_harmonics_regularization(
    voxel_grid, 
    max_sh_degree: int, 
    curriculum_epochs: int = 5,
    learning_rate: float
):
    """
    Applies SH regularization during training.
    
    Args:
        voxel_grid: Object containing SH coefficients c[l][m] for each voxel.
        max_sh_degree: The maximum SH degree L_max (e.g., 3).
        curriculum_epochs: Number of epochs for the curriculum (E_curriculum).
        learning_rate: Base learning rate for SH coefficients.
        
    Returns:
        Updated voxel_grid after one training step.
    """
    # Determine current max degree based on epoch
    # Note: The exact epoch counter is assumed to be available in the training loop context.
    # Here we assume 'current_epoch' is passed or tracked externally.
    # For the purpose of this self-contained function, we define the logic for a given epoch.
    
    # Placeholder for current epoch from outer loop
    # current_epoch = ... 
    
    # Calculate current allowed degree
    # Source [S1]: "decrease the rate of SH masking by 1/5 per epoch during the early five epochs"
    # This implies a linear ramp-up. 
    # If we start at degree 0 (only l=0), and end at degree L_max after 5 epochs.
    # A simple interpretation: L_curr = floor(current_epoch * (max_sh_degree / curriculum_epochs))
    # However, [S1] mentions masking degrees 2 and 3 specifically in the context of HDR-Plenoxels.
    # Let's implement a general linear schedule from 0 to max_sh_degree.
    
    if current_epoch < curriculum_epochs:
        # Linear interpolation: at epoch 0, degree 0; at epoch 5, degree max_sh_degree
        # Using float division for smoothness, then clamping
        progress = current_epoch / curriculum_epochs
        current_max_degree = int(progress * max_sh_degree)
    else:
        current_max_degree = max_sh_degree
        
    # Apply masking to gradients or coefficients
    for voxel in voxel_grid:
        for l in range(0, max_sh_degree + 1):
            for m in range(-l, l + 1):
                # Get gradient for this coefficient
                grad = get_gradient(voxel, l, m)
                
                # Apply mask: if l > current_max_degree, mask is 0
                if l > current_max_degree:
                    masked_grad = 0.0
                else:
                    masked_grad = grad
                    
                # Update coefficient
                voxel.c[l][m] += learning_rate * masked_grad
                
    return voxel_grid
```

**Edge Cases and Pitfalls:**
*   **Precision:** SH coefficients can vary in magnitude. Masking high-frequency terms prevents them from dominating the loss gradient early on, which can cause instability if not masked [S1].
*   **Boundary Conditions:** The transition from masked to unmasked coefficients must be smooth to avoid sudden changes in radiance representation. The linear schedule in [S1] helps mitigate this.
*   **Degenerate Input:** In scenes with very high-frequency view-dependent effects (glossy materials), early masking might slow down convergence for those specific materials, but it stabilizes the overall geometry [S1].

## Operating characteristics
*   **Time Complexity:** Adds negligible computational overhead. The masking operation is a simple conditional check per coefficient per voxel per step [S2].
*   **Memory Complexity:** No additional memory required beyond storing the SH coefficients and the epoch counter [S2].
*   **Performance:** Improves geometry accuracy and reduces floaters, especially in few-shot or harsh input conditions [S1, S2]. It stabilizes optimization by decoupling geometry learning (low frequency) from appearance learning (high frequency) [S1].
*   **Strengths:** Simple to implement, no extra hyperparameters beyond the curriculum length, effective at preventing high-frequency overfitting [S1, S2].
*   **Limitations:** Requires a fixed curriculum schedule; may not adapt well to scenes with varying frequency content across different regions [S1].

## Background
Spherical Harmonics Regularization was introduced in the context of Plenoxels and related voxel-based radiance field methods. It was inspired by frequency regularization techniques used in FreeNeRF (positional encoding) and FourieRF (Fourier coefficients) [S2]. The specific application to SH coefficients was detailed in HDR-Plenoxels [S1] and F2Plenoxels [S2]. It is used in few-shot view synthesis and high-dynamic-range radiance field reconstruction [S1, S2].

## Sources
[S1] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S2] F2Plenoxels: fast voxel radiance fields without neural networks for few-shot view synthesis, https://doi.org/10.1007/s44443-026-00491-2, fetched 2026-10-26
