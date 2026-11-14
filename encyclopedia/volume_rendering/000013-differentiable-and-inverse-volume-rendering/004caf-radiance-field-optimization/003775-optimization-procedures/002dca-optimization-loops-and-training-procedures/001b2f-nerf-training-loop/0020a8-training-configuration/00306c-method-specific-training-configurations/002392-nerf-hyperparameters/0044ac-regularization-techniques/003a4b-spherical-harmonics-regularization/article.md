# Spherical Harmonics Regularization

Also called: SH coefficient masking, SH curriculum training, SH degree regularization

## Overview
Spherical Harmonics (SH) Regularization is a training strategy that restricts the maximum degree of Spherical Harmonics during early optimization to prevent high-frequency overfitting and improve coarse geometry reconstruction [S2]. It stabilizes optimization in harsh input conditions by allowing the model to learn diffuse reflectance (low-frequency SH) before view-direction-sensitive components (high-frequency SH) [S1]. Unlike methods that modify the loss function directly, this approach acts as a representation restriction, gradually increasing the effective capacity of the radiance field [S1, S2].

## How it works
**Inputs and Outputs**
*   **Input:** Current training epoch/iteration $t$, total training duration $T_{total}$ (or number of early epochs $T_{early}$), and the set of SH coefficients $\mathbf{c}$ for each voxel.
*   **Output:** A binary mask $\mathbf{m}$ or a maximum allowed degree $d_{max}$ applied to the SH coefficients.

**Data Structures**
*   **SH Coefficients:** Stored as a vector per voxel. For degree $L$, there are $(L+1)^2$ coefficients per color channel (RGB). The coefficients are indexed by degree $l \in [0, L_{max}]$ and order $m \in [-l, l]$.
*   **Mask:** A boolean array or integer bitmask corresponding to the coefficient indices. If $\mathbf{m}_i = 0$, coefficient $i$ is frozen or zeroed out during the gradient update.

**Parameters**
*   $L_{max}$: The maximum SH degree used in the representation (typically 2 or 3 for Plenoxels) [S1].
*   $T_{early}$: The number of early epochs for curriculum scheduling. Sources specify 5 epochs [S1].
*   $r$: The reduction rate of the masking constraint per epoch. Sources specify $1/5$ [S1].

**Steps**
1.  **Initialize Mask:** At the start of training ($t=0$), set the maximum allowed SH degree $d_{max} = 0$ (or restrict to degree 1, depending on implementation details not fully specified, but effectively masking degrees 2 and 3 as per [S1]). All coefficients for $l > d_{max}$ are masked.
2.  **Schedule Update:** For each epoch $t$ during the early phase ($t < T_{early}$):
    *   Calculate the current allowed degree or mask rate.
    *   According to [S1], the "rate of SH masking" decreases by $1/5$ per epoch. This implies that at epoch $t$, the fraction of high-frequency coefficients that remain masked is reduced.
    *   Specifically, [S1] states: "We apply SH masking to the coefficient of SH of degrees 2 and 3. We decrease the rate of SH masking by 1/5 per epoch during the early five epochs... After the early five epochs, we update SH of all degrees with full rate, i.e., no SH masking."
    *   This implies a linear schedule where the mask is fully active for degrees 2 and 3 at $t=0$, and is removed completely by $t=5$.
3.  **Apply Mask:** During the backward pass, zero out gradients for any coefficient $c_{l,m}$ where $l > d_{max}(t)$ or where the mask $\mathbf{m}$ is active.
4.  **Unmask:** For $t \ge T_{early}$, set $d_{max} = L_{max}$. All coefficients are updated normally.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_harmonics_regularization(epoch, total_epochs, max_degree, early_epochs=5):
    """
    Determines the maximum SH degree allowed for gradient updates.
    
    Args:
        epoch: Current training epoch (0-indexed).
        total_epochs: Total number of epochs (unused in this specific schedule).
        max_degree: The maximum SH degree of the model (e.g., 2 or 3).
        early_epochs: Number of epochs for the curriculum (default 5).
        
    Returns:
        current_max_degree: The highest SH degree whose gradients are not masked.
    """
    # If training has passed the early phase, allow all degrees
    if epoch >= early_epochs:
        return max_degree
    
    # Early phase: Gradually increase allowed degree
    # Source [S1] specifies masking degrees 2 and 3 initially, 
    # and decreasing the masking rate by 1/5 per epoch for 5 epochs.
    # This implies a linear transition from masking high frequencies to none.
    
    # Calculate progress through the early phase (0.0 to 1.0)
    progress = epoch / early_epochs
    
    # Determine the threshold degree. 
    # At epoch 0, we mask degrees > 1 (allowing only 0 and 1).
    # At epoch 4 (last early epoch), we mask nothing.
    # Linear interpolation:
    # allowed_degree = 1 + progress * (max_degree - 1)
    # However, SH degrees are integers. We floor or round.
    
    # More precise interpretation of [S1]:
    # "Decrease the rate of SH masking by 1/5 per epoch"
    # Let M be the set of masked coefficients. Initially M contains all l >= 2.
    # Each epoch, 1/5 of the "masking weight" is removed.
    # Since there are 5 epochs, at epoch 5, 5/5 = 100% of masking is removed.
    
    # Implementation: Allow degree l if l <= floor(1 + progress * (max_degree - 1))
    # Or simply: Allow degree 0 and 1 for first few epochs, then 2, etc.
    
    # Given max_degree is typically 2 or 3 in Plenoxels:
    # If max_degree == 2:
    #   Epoch 0-1: Allow l=0,1
    #   Epoch 2-3: Allow l=0,1,2 (partial or full unmasking?)
    #   Epoch 4: Allow all
    
    # Simplified linear schedule for max allowed degree:
    current_max_degree = int(1 + progress * (max_degree - 1))
    
    # Ensure it doesn't exceed max_degree
    current_max_degree = min(current_max_degree, max_degree)
    
    return current_max_degree

def apply_sh_mask(gradient, current_max_degree, max_degree):
    """
    Zeros out gradients for SH coefficients with degree > current_max_degree.
    """
    # Assuming gradient is a flat array or structured by (l, m)
    # Iterate through coefficients and zero out if l > current_max_degree
    for l in range(current_max_degree + 1, max_degree + 1):
        for m in range(-l, l + 1):
            gradient[l, m] = 0
    return gradient
```

**Edge Cases and Pitfalls**
*   **Precision:** Ensure that the transition between epochs does not cause discontinuous jumps in gradient magnitude that destabilize training. Linear interpolation of the "mask rate" helps mitigate this [S1].
*   **Boundary Conditions:** At $t=0$, if $L_{max}=0$, no regularization is needed. If $L_{max} < 2$, the masking of degrees 2 and 3 is vacuous.
*   **Harsh Conditions:** The method is specifically designed for "harsh condition input cases" (e.g., sparse views, low light) where high-frequency components might overfit to noise [S1].

## Operating characteristics
*   **Time Complexity:** Negligible. The mask application is a simple conditional check or zeroing operation during the backward pass. It does not add forward-pass computation.
*   **Memory Complexity:** Negligible. No additional data structures are required beyond the existing SH coefficients.
*   **Strengths:** Stabilizes optimization in early stages, preventing high-frequency overfitting to noise in sparse data [S1, S2]. Improves coarse geometry reconstruction [S2].
*   **Limitations:** Slows down the convergence of high-frequency details (glossy materials) until the mask is removed [S1]. Not suitable if the scene is known to be purely diffuse and low-frequency, as it adds unnecessary constraint.
*   **When to Use:** Few-shot view synthesis, sparse input views, or scenes with potential high-frequency artifacts (floaters) [S1, S2].
*   **When Not to Use:** Dense input views with abundant high-frequency detail where early overfitting is less likely.

## Background
*   **Origin:** The method was introduced in **HDR-Plenoxels** (2022) as "SH coefficient masking" to stabilize optimization for high dynamic range radiance fields [S1]. It was later adopted and refined in **F2Plenoxels** (2026) as "SH degree regularization" to improve geometry accuracy in few-shot settings [S2].
*   **Inspiration:** Inspired by frequency regularization techniques in FreeNeRF (positional encoding) and FourieRF (Fourier coefficients), but applied directly to SH coefficients in a voxel grid representation [S2].
*   **History:** Developed as a response to the instability of neural radiance fields and voxel-based methods when trained on limited data, where high-frequency SH coefficients tend to fit noise rather than signal [S1, S2].

## Sources
[S1] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S2] F2Plenoxels: fast voxel radiance fields without neural networks for few-shot view synthesis, https://doi.org/10.1007/s44443-026-00491-2, fetched 2026-10-26
