# Wavelet-based Temporal Feature Extraction and Enhancement

Also called: Multiscale time-feature enhancement, Time-feature extraction and enhancement, Wavelet-based temporal feature extraction

## Overview
A compression-domain rendering technique for time-resolved volume data that uses wavelet multiresolution analysis to detect temporal singularities, retain time-varying structures, and suppress static regions to reduce memory usage and enhance feature visualization [S1]. It distinguishes itself from sibling methods like [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md) by explicitly leveraging Lipschitz regularity analysis to separate irregular temporal behavior from smooth static backgrounds, rather than merely projecting coefficients for visibility.

## How it works
**Inputs:** A time-resolved volume sequence $V(t, x, y, z)$ where $t$ is the time index and $(x, y, z)$ are spatial coordinates.
**Outputs:** A compressed multiresolution representation and/or a rendered volume where static regions are suppressed and time-varying features are enhanced.

### Data Structures
The method operates on wavelet coefficients. For a sequence with $T$ time steps and spatial resolution $N \times N \times N$, the data is decomposed into:
1.  **Spatial Wavelet Coefficients:** For each time step $t$, the volume is decomposed into approximation coefficients $A_{j}$ and detail coefficients $D_{j}$ across spatial scales $j$.
2.  **Temporal Coefficients:** The sequence of coefficients (either approximations or details) at specific spatial locations is treated as a 1D temporal signal.
3.  **Lipschitz Map:** A binary or scalar map indicating the presence of singularities (Lipschitz regularity $\alpha \in [0, 2]$) for each coefficient.

### Parameters
*   **Wavelet Basis:** Cubic B-Spline wavelets with 2 vanishing moments [S1]. The support of the basis function is 6 [S1].
*   **Lipschitz Threshold:** $\alpha = 2$. Coefficients contributing to singularities with regularity up to 2 are retained [S1].
*   **Coarsest Levels for Temporal Blurring:** 2 or 3 out of 7 total levels [S1].
*   **Filter Threshold:** Filter entries below a certain threshold are neglected during decomposition and reconstruction [S1].

### Steps

1.  **Boundary Handling:** Apply inverse mirroring at the start and end points of the temporal signal to handle boundary problems caused by the infinite support of the wavelet filters [S1].
2.  **Wavelet Decomposition:** Perform the wavelet transform on the temporal signal of each coefficient. Neglect filter entries below the specified threshold [S1].
3.  **Singularity Detection:** Analyze the Lipschitz regularity of the temporal signal. Detect sharp variation points (peaks, discontinuities, turbulent structures) by examining the regularity between 0 and 2 [S1].
4.  **Feature Extraction/Enhancement Decision:**
    *   *Extraction:* Retain all wavelet coefficients that contribute to detected singularities in time up to a Lipschitz regularity of 2 [S1].
    *   *Enhancement:* Reconstruct only structures that behave irregularly over time. Suppress regions where no singularities are determined in their time evolution [S1].
5.  **Temporal Blurring (Optional):** To examine object movement, reconstruct each time step using only the coarsest resolution levels (e.g., 2 or 3 out of 7) [S1]. This applies low-pass filtering in the time domain, projecting global information from neighboring time steps into the current volume [S1].
6.  **Reconstruction:** Reconstruct the volume from the selected coefficients. For enhancement, static regions are suppressed [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavelet_temporal_feature_extraction(volume_sequence, wavelet_basis='cubic_bspline', 
                                        lipschitz_threshold=2.0, 
                                        coarsest_levels=3, 
                                        total_levels=7,
                                        filter_threshold=1e-6):
    """
    Performs wavelet-based temporal feature extraction and enhancement.
    
    Args:
        volume_sequence: List of 3D arrays (t, x, y, z)
        wavelet_basis: Wavelet type (cubic B-Spline with 2 vanishing moments)
        lipschitz_threshold: Max Lipschitz regularity to consider as singularity
        coarsest_levels: Number of coarsest levels to use for temporal blurring
        total_levels: Total number of wavelet levels
        filter_threshold: Threshold for neglecting filter entries
        
    Returns:
        enhanced_volume: The reconstructed volume with enhanced temporal features
    """
    t_steps = len(volume_sequence)
    enhanced_volume = None
    
    # Step 1: Boundary Handling
    # Apply inverse mirroring at start and end of temporal signal
    # This is handled implicitly during the 1D temporal transform on coefficients
    # or by padding the temporal signal with mirrored values.
    
    # Step 2 & 3: Decomposition and Singularity Detection
    # We iterate over spatial locations or blocks. 
    # For efficiency, this is often done block-wise.
    
    singularity_map = {} # Stores (t, x, y, z) -> is_singular
    
    for t in range(t_steps):
        # Spatial Wavelet Transform for current time step
        spatial_coeffs = spatial_wavelet_transform(volume_sequence[t], wavelet_basis)
        
        # For each coefficient location (x, y, z) and scale j
        for x, y, z in spatial_coords:
            for j in range(total_levels):
                # Get temporal signal for this coefficient
                temporal_signal = [spatial_coeffs[t][x][y][z][j] for t in range(t_steps)]
                
                # Apply temporal wavelet transform
                # Neglect filter entries below threshold
                temporal_coeffs = temporal_wavelet_transform(temporal_signal, wavelet_basis, filter_threshold)
                
                # Analyze Lipschitz regularity
                # Detect sharp variation points (peaks, discontinuities)
                # Regularity alpha in [0, 2]
                alpha = compute_lipschitz_regularity(temporal_coeffs)
                
                # Step 4: Decision
                is_singular = alpha <= lipschitz_threshold
                singularity_map[(t, x, y, z, j)] = is_singular
                
    # Step 5 & 6: Reconstruction
    # Initialize reconstructed volume
    reconstructed_volume = create_empty_volume_like(volume_sequence[0])
    
    for t in range(t_steps):
        # Determine which coefficients to keep based on singularity map
        # For enhancement: keep only singular coefficients
        # For blurring: keep only coarsest levels (optional mode)
        
        keep_mask = {}
        for (tt, x, y, z, j), is_singular in singularity_map.items():
            if tt == t:
                if mode == 'enhancement':
                    keep_mask[(x, y, z, j)] = is_singular
                elif mode == 'blurring':
                    # Keep only coarsest levels
                    # Coarsest levels are typically the lowest indices or highest scales
                    # Assuming j=0 is finest, j=total_levels-1 is coarsest
                    # Or vice versa. Let's assume j=0 is coarsest for this logic
                    # The source says "coarsest levels... out of 7".
                    # Usually, level 0 is coarsest.
                    if j < coarsest_levels:
                        keep_mask[(x, y, z, j)] = True
                    else:
                        keep_mask[(x, y, z, j)] = False
                else: # Full reconstruction
                    keep_mask[(x, y, z, j)] = True
        
        # Reconstruct spatial volume from kept coefficients
        reconstructed_volume[t] = inverse_spatial_wavelet_transform(keep_mask, wavelet_basis)
        
    return reconstructed_volume

def compute_lipschitz_regularity(temporal_coeffs):
    """
    Computes the local Lipschitz regularity from wavelet coefficients.
    This is a placeholder for the specific algorithm described in [S1].
    Typically involves analyzing the decay of coefficients across scales.
    """
    # Not given in the sources. [G]
    # Standard approach: alpha = -lim_{j->inf} log2(|W(j)|) / j
    # Here we return a dummy value for implementation completeness.
    return 1.5

def spatial_wavelet_transform(volume, wavelet_basis):
    """
    Performs 3D spatial wavelet transform.
    """
    # Not given in the sources. [G]
    # Standard 3D wavelet transform implementation.
    return {}

def temporal_wavelet_transform(signal, wavelet_basis, threshold):
    """
    Performs 1D temporal wavelet transform, neglecting small filter entries.
    """
    # Not given in the sources. [G]
    # Standard 1D wavelet transform.
    return {}

def inverse_spatial_wavelet_transform(keep_mask, wavelet_basis):
    """
    Reconstructs volume from selected wavelet coefficients.
    """
    # Not given in the sources. [G]
    # Standard inverse wavelet transform.
    return {}
```

### Edge Cases and Pitfalls
*   **Static Region Blurring:** On coarser resolution levels, static regions are influenced by coefficients belonging to turbulent areas. This causes smooth approximations to contribute to static regions, blurring structures away from the center [S1].
*   **Filter Cutoff:** Since filter sequences are infinite, boundary problems and errors due to necessary cutoff of sequences must be taken into consideration [S1]. Inverse mirroring is used to mitigate this [S1].

## Operating characteristics
*   **Time Complexity:** High numerical complexity due to the rendering integral evaluation based on wavelet expansion, a reconstruction step for multiresolution representation, and a complex pre-processing step for singularity and local Lipschitz regularity detection [S1].
*   **Memory Complexity:** Significant memory savings. For a 512 MB sequence, compression to 74 MB saved 150 MB (Note: 512 - 74 = 438 MB saved, but source states 150 MB saved for time evolutions specifically, implying other data remains or the 512 MB was the uncompressed total and 74 MB is the compressed total, with 150 MB being the specific saving from time-feature extraction part) [S1].
*   **Scalability:** The approach can be easily implemented on distributed memory, parallel computers. The volume is divided into subblocks distributed over nodes. Wavelet transforms are performed individually, with global communication based on filter sequence length. Singularity detection and compression are done locally for equivalent subblocks in subsequent time steps [S1].
*   **Strengths:** Discriminates time-varying structures from static parts, enhancing understanding of the data. Good localization of varying structures [S1].
*   **Limitations:** High computational cost for pre-processing and rendering. Potential blurring of static regions due to influence from turbulent coefficients at coarse levels [S1].

## Background
*   **Origin:** Proposed in "Compression domain rendering of time-resolved volume data" [S1].
*   **History:** The method builds on multiresolution analysis and wavelet theory. It applies cubic B-Spline wavelets with 2 vanishing moments, similar to those used in [Cardinal B-spline wavelets](11756-cardinal-b-spline-wavelets.md).
*   **Usage:** Applied to stochastic fractal cloud sequences and moving high-density object sequences [S1].

## Sources
[S1] Compression domain rendering of time-resolved volume data, https://ieeexplore.ieee.org/document/480809/, fetched 2026-10-29
