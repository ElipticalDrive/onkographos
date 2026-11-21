# Model-Based Local Transfer Functions

Also called: Model-based TFs, Morphon-based local TFs

## Overview
A visualization method for medical image volumes that applies multiple voxel-specific transfer functions simultaneously to different anatomical regions, determined by a deformable anatomical model registered to the volume [S1]. It allows a radiologist to review more information in a single visualization by automatically assigning transfer functions to anatomically relevant regions, solving the problem where different transfer functions are only relevant for disjoint anatomical regions [S1]. It sets itself apart from sibling methods like [Spatial Style Zones](6652-spatial-style-zones.md) by using a probabilistic anatomical model and non-rigid registration (Morphon) rather than fixed spatial zones or texture statistics [S1].

## How it works
**Inputs:**
*   Target volume $I_T$ (medical image data).
*   Source anatomical model $I_S$ (initialized as $I_M$).
*   Number of compartments $N$.
*   Display bit depth $bits_{display}$.
*   User-defined parameters: weighting values $a_i$, min/max bounds $\alpha_i, \beta_i$ for each compartment, and transition averaging filter settings [S1].

**Outputs:**
*   Display volume $I_{disp}$ suitable for visualization [S1].

**Data Structures:**
*   **Model Compartments:** $N$ probability fields $IC_i(\mathbf{x})$, where each voxel value is in $[0, 1]$. The sum of probabilities for all compartments at any location $\mathbf{x}$ is 1: $\sum_{i=1}^{N} IC_i(\mathbf{x}) = 1$ [S1].
*   **Combined Model:** $I_M(\mathbf{x}) = \sum_{i=1}^{N} a_i IC_i(\mathbf{x})$ [S1].
*   **Deformation Field:** $d(\mathbf{x})$, a vector field mapping source coordinates to target coordinates [S1].
*   **Deformed Compartments:** $\tilde{IC}_i(\mathbf{x})$, the registered probability fields [S1].
*   **Local Bounds:** $I_\alpha(\mathbf{x})$ and $I_\beta(\mathbf{x})$, computed per-voxel based on deformed probabilities [S1].

**Steps:**

1.  **Model Definition:** Define the anatomical model $I_M$ with $N$ compartments. Assign weighting values $a_i$ to combine compartments. The selection of $a_i$ depends on the registration method; for phase-based methods (like Morphon), boundaries between compartments in the model must have the same sign as in the target volume [S1]. Assign min ($\alpha_i$) and max ($\beta_i$) values to each compartment $i$ to control the local transfer functions [S1].

2.  **Registration:** Register the source model $I_S$ (where $I_S = I_M$) to the target volume $I_T$ using the Morphon algorithm [S1].
    *   The Morphon is a phase-based non-rigid registration algorithm [S1].
    *   Iteratively deform $I_S$ via a displacement field $d(\mathbf{x})$ such that the deformed source $I_D(\mathbf{x}) = I_S(\mathbf{x} + d(\mathbf{x}))$ becomes sufficiently similar to $I_T$ [S1].
    *   Perform registration over multiple scales: start at coarse scales to register large global displacements, then move to finer scales for smaller local deformations [S1].
    *   [G] The specific Morphon implementation details (e.g., gradient computation, optimization loop) are not provided in the source notes; assume a standard multi-scale phase-based registration routine is used [S1].

3.  **Deformation of Compartments:** Apply the estimated deformation field $d(\mathbf{x})$ to all model compartments using linear interpolation to obtain deformed compartments $\tilde{IC}_i(\mathbf{x})$ [S1]:
    $$ \tilde{IC}_i(\mathbf{x}) = IC_i(\mathbf{x} + d(\mathbf{x})) $$
    [S1]

4.  **Compute Local Transfer Function Bounds:** For each voxel $\mathbf{x}$, compute the local minimum $I_\alpha(\mathbf{x})$ and maximum $I_\beta(\mathbf{x})$ bounds using the deformed compartment probabilities $\tilde{IC}_i(\mathbf{x})$ and the compartment-specific bounds $\alpha_i, \beta_i$ [S1]:
    $$ I_\alpha(\mathbf{x}) = \frac{\sum_{i=1}^{N} \alpha_i \tilde{IC}_i(\mathbf{x})}{\sum_{i=1}^{N} \tilde{IC}_i(\mathbf{x})} $$
    $$ I_\beta(\mathbf{x}) = \frac{\sum_{i=1}^{N} \beta_i \tilde{IC}_i(\mathbf{x})}{\sum_{i=1}^{N} \tilde{IC}_i(\mathbf{x})} $$
    [S1]
    Note: Since $\sum \tilde{IC}_i(\mathbf{x}) = 1$ (assuming interpolation preserves the sum or is normalized), the denominator is 1. If interpolation breaks the sum constraint, normalization is required [S1].

5.  **Transform Voxel Values:** Transform the input volume $I$ into the display volume $I_{disp}$ using the local bounds [S1].
    *   Calculate the slope for the display range:
        $$ I_{slope} = \frac{2^{bits_{display}} - 1}{I_\beta(\mathbf{x}) - I_\alpha(\mathbf{x})} $$
        [S1]
    *   Clamp the input voxel value $I(\mathbf{x})$ to the local range $[I_\alpha(\mathbf{x}), I_\beta(\mathbf{x})]$:
        $$ \tilde{I}(\mathbf{x}) = \min(\max(I(\mathbf{x}), I_\alpha(\mathbf{x})), I_\beta(\mathbf{x})) $$
        [S1]
    *   Map the clamped value to the display range:
        $$ I_{disp}(\mathbf{x}) = I_{slope} \cdot (\tilde{I}(\mathbf{x}) - I_\alpha(\mathbf{x})) $$
        [S1]

6.  **Transition Handling (Optional):** Apply a user-controlled averaging filter to the transition regions between compartments to smooth disturbing features. The radiologist selects the filter type and size [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def model_based_local_tf(target_volume, source_model_compartments, num_compartments, 
                         weights, alpha_bounds, beta_bounds, bits_display, 
                         deformation_field, transition_filter=None):
    """
    Applies model-based local transfer functions.
    
    Args:
        target_volume: 3D array of input voxel values I_T.
        source_model_compartments: List of N 3D arrays IC_i, probabilities in [0,1].
        num_compartments: N.
        weights: List of N floats a_i.
        alpha_bounds: List of N floats alpha_i.
        beta_bounds: List of N floats beta_i.
        bits_display: Integer, display bit depth.
        deformation_field: 3D array of vectors d(x), same size as volume.
        transition_filter: Optional filter function for smoothing.
        
    Returns:
        I_disp: 3D array of display values.
    """
    
    # Step 1 & 3: Deform compartments
    # IC_i_tilde(x) = IC_i(x + d(x))
    # Assuming deformation_field is a vector field where field[x,y,z] = (dx, dy, dz)
    deformed_compartments = []
    for i in range(num_compartments):
        ic_i = source_model_compartments[i]
        # Linear interpolation of IC_i at x + d(x)
        # [G] Implementation of trilinear interpolation at deformed coordinates is assumed
        ic_i_tilde = interpolate_linear(ic_i, deformation_field)
        deformed_compartments.append(ic_i_tilde)
        
    # Step 4: Compute local bounds I_alpha and I_beta
    # Sum of probabilities should be 1, but we normalize to be safe
    prob_sum = sum(deformed_compartments)
    # Avoid division by zero
    prob_sum = max(prob_sum, 1e-6)
    
    I_alpha = numpy.zeros_like(target_volume)
    I_beta = numpy.zeros_like(target_volume)
    
    for i in range(num_compartments):
        I_alpha += alpha_bounds[i] * deformed_compartments[i] / prob_sum
        I_beta += beta_bounds[i] * deformed_compartments[i] / prob_sum
        
    # Step 5: Transform voxel values
    max_display_val = 2**bits_display - 1
    
    # Calculate slope per voxel
    # Handle case where I_beta == I_alpha to avoid division by zero
    denom = I_beta - I_alpha
    denom = numpy.where(denom == 0, 1e-6, denom)
    I_slope = max_display_val / denom
    
    # Clamp input value I to [I_alpha, I_beta]
    I_clamped = numpy.clip(target_volume, I_alpha, I_beta)
    
    # Map to display range
    I_disp = I_slope * (I_clamped - I_alpha)
    
    # Step 6: Transition handling
    if transition_filter is not None:
        # Apply filter to I_disp or the probability fields? 
        # Source says "averaging of the transition regions", likely applied to the 
        # resulting visualization or the bounds. We apply to I_disp as a post-process 
        # smoothing of the final image to handle registration imperfections.
        I_disp = apply_filter(I_disp, transition_filter)
        
    return I_disp

def interpolate_linear(volume, deformation_field):
    """
    Performs trilinear interpolation of 'volume' at coordinates defined by deformation_field.
    deformation_field has shape (H, W, D, 3).
    """
    H, W, D, _ = deformation_field.shape
    result = numpy.zeros((H, W, D))
    
    # [G] Standard trilinear interpolation logic
    # For each voxel (x,y,z), get target coordinate (x+dx, y+dy, z+dz)
    # Sample the 8 surrounding voxels in 'volume' and interpolate.
    # This is a placeholder for the actual interpolation kernel.
    for x in range(H):
        for y in range(W):
            for z in range(D):
                # Get deformed coordinate
                dx, dy, dz = deformation_field[x, y, z]
                tx = x + dx
                ty = y + dy
                tz = z + dz
                
                # Check bounds
                if not (0 <= tx < H-1 and 0 <= ty < W-1 and 0 <= tz < D-1):
                    result[x, y, z] = 0.0
                    continue
                    
                # Trilinear interpolation
                x0, x1 = int(tx), int(tx) + 1
                y0, y1 = int(ty), int(ty) + 1
                z0, z1 = int(tz), int(tz) + 1
                
                fx = tx - x0
                fy = ty - y0
                fz = tz - z0
                
                # Get 8 corner values
                v000 = volume[x0, y0, z0]
                v100 = volume[x1, y0, z0]
                v010 = volume[x0, y1, z0]
                v110 = volume[x1, y1, z0]
                v001 = volume[x0, y0, z1]
                v101 = volume[x1, y0, z1]
                v011 = volume[x0, y1, z1]
                v111 = volume[x1, y1, z1]
                
                # Interpolate
                v00 = v000 * (1-fx) + v100 * fx
                v10 = v010 * (1-fx) + v110 * fx
                v01 = v001 * (1-fx) + v101 * fx
                v11 = v011 * (1-fx) + v111 * fx
                
                v0 = v00 * (1-fy) + v10 * fy
                v1 = v01 * (1-fy) + v11 * fy
                
                result[x, y, z] = v0 * (1-fz) + v1 * fz
                
    return result
```

## Operating characteristics
*   **Time Complexity:** Dominated by the registration step (Morphon), which is iterative and multi-scale. The visualization step is linear $O(V)$ where $V$ is the number of voxels, involving per-voxel arithmetic operations [S1].
*   **Memory Complexity:** Requires storage for the target volume, the source model compartments ($N$ fields), the deformation field, and the deformed compartments. $O(N \cdot V)$ for compartments and $O(V)$ for deformation [S1].
*   **Strengths:** Allows efficient review of multiple anatomical regions with different transfer functions in a single visualization. Handles non-rigid anatomy via registration. Smooth transitions via probabilistic compartments reduce disturbing artifacts [S1].
*   **Limitations:** Relies on the accuracy of the registration algorithm. Imperfect registration can still introduce features in transition regions, requiring user-controlled averaging [S1]. Computationally expensive due to registration [S1].
*   **When to use:** Medical image visualization where anatomical structure is known and can be modeled, and efficient review of heterogeneous tissue types is required [S1].
*   **When not to use:** Real-time applications where registration time is prohibitive, or when no suitable anatomical model exists [S1].

## Background
*   **Origin:** Proposed by Forsberg et al. in "Model-Based Transfer Functions for Efficient Visualization of Medical Image Volumes" [S1].
*   **History:** The method utilizes the Morphon algorithm for phase-based non-rigid registration [S1]. It addresses the limitation of previous methods that did not utilize underlying human anatomy to control visualization [S1].
*   **Usage:** Medical image volume visualization, specifically for radiologists reviewing complex anatomical data [S1].

## Sources
[S1] Model-Based Transfer Functions for Efficient Visualization of Medical Image Volumes, https://doi.org/10.1007/978-3-642-21227-7_55, fetched 2026-10-30
