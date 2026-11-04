# DONeRF Depth Oracle Filtering

Also called: Classified Depth Oracle, Depth smoothing filter, Neighborhood filter

## Overview
DONeRF Depth Oracle Filtering is a preprocessing step applied to the training targets of the depth oracle network in DONeRF to improve surface detection consistency and reduce false negatives [S1]. It distinguishes itself from sibling methods like [Hexagonal Multisampling Pattern Derivation](3498-hexagonal-multisampling-pattern-derivation.md) by operating on the classification targets in image and depth space rather than deriving sampling patterns directly, and from [Hierarchical Sampling for Hemispherical Wavefronts](4691-hierarchical-sampling-for-hemispherical-wavefronts.md) by focusing on smoothing discontinuities in the oracle's one-hot encoding targets [S1].

## How it works
The method filters the ground-truth classification targets generated for the depth oracle network. The oracle predicts a likelihood for each discretized segment along a ray, where a high value indicates the segment should receive samples and a low value indicates it should be skipped [S1].

### Inputs and Outputs
- **Input**: The unfiltered one-hot classification target $C_{x,y}(z)$ for each pixel $(x,y)$ and discretized depth segment $z$.
- **Output**: The filtered classification target $\hat{C}_{x,y}(z)$ used for training the depth oracle network.
- **Note**: Filtering is applied only to training targets; no filtering is performed during inference [S1].

### Data Structures
The classification target is a 3D tensor or array indexed by pixel coordinates $(x,y)$ and depth segment index $z$.
- $C_{x,y}(z)$: Binary value (0 or 1).
- $d_s$: Depth of the first surface along the ray.
- $d_z, d_{z+1}$: Discretization boundaries for segment $z$.

### Formulas and Definitions
The unfiltered target is a one-hot encoding [S1]:
$$
C_{x,y}(z) = \begin{cases} 1, & \text{if } d_z \leq d_s < d_{z+1} \\ 0, & \text{otherwise} \end{cases}
$$
where $d_s$ is the depth of the first surface, and $d_z, d_{z+1}$ are the boundaries of the discretized segment $z$ [S1].

The filtering consists of two stages: a 2D spatial neighborhood filter and a 1D depth-axis smoothing filter [S1].

1. **2D Neighborhood Filter (Radial)**:
   Applies a radial filter to include values from neighboring rays with a contribution weighted by their Euclidean distance [S1]. For a filter size $K$, the intermediate target $\hat{C}_{x,y}(z)$ is computed as:
   $$
   \hat{C}_{x,y}(z) = \max_{i,j \in \pm \lfloor K/2 \rfloor} \left( C_{x+i,y+j}(z) - \frac{\sqrt{2+j^2}}{\sqrt{2 \cdot \lfloor K/2 \rfloor}} \right)
   $$
   Here, $i$ and $j$ are offsets in the x and y directions. The term $\frac{\sqrt{2+j^2}}{\sqrt{2 \cdot \lfloor K/2 \rfloor}}$ represents the penalty for distance. Note that the source formula uses $j$ in the numerator's square root but implies a radial distance; however, the text specifies "Euclidean distance" and gives examples for distances 0, 1, 2, 3 pixels. The provided formula in the source [S1] Eq. 7 uses $j$ in the numerator, which appears to be a typo for radial distance $\sqrt{i^2+j^2}$ or similar, but we must follow the source's explicit formula. The source states: "rays at a distance of 3, 2, 1, 0 pixels contribute 0, ≈ 0.30, ≈ 0.65, 1". Let's verify the source formula's behavior. If $K=5$, $\lfloor K/2 \rfloor = 2$. Denominator is $\sqrt{2 \cdot 2} = 2$.
   - Distance 0 ($i=0,j=0$): $1 - 0/2 = 1$.
   - Distance 1 ($i=1,j=0$ or $0,1$): $\sqrt{1}/2 = 0.5$. Contribution $1-0.5=0.5$. Source says ≈0.65.
   - Distance 2 ($i=2,j=0$ or $0,2$ or $1,1$): $\sqrt{2}/2 \approx 0.707$. Contribution $1-0.707=0.293$. Source says ≈0.30.
   - Distance 3 ($i=3,j=0$ etc, but $K=5$ limits to $\pm 2$). The source mentions distance 3 contributing 0. This implies the filter might be larger or the formula in the text has a specific interpretation. The source formula (7) uses $\sqrt{2+j^2}$. This seems inconsistent with "Euclidean distance" if $i$ is ignored. However, the source explicitly provides Eq. 7. We will use Eq. 7 as the definition, noting the source's example values [S1].
   
   For multiple depth values with the same discretization result (which shouldn't happen in one-hot, but may occur if smoothing creates overlaps or if the max is taken over multiple $z$), the maximum result is taken to ensure the largest value is placed at the actual surface [S1].

2. **1D Depth Smoothing Filter**:
   Applies a 1D triangle filter along the depth axis with kernel size $Z$ [S1]:
   $$
   \hat{C}(z) = \min \left( \sum_{i=-\lfloor Z/2 \rfloor}^{\lfloor Z/2 \rfloor} \hat{C}(z+i) \frac{\lfloor Z/2 \rfloor + 1 - |i|}{\lfloor Z/2 \rfloor + 1}, 1 \right)
   $$
   This acts as label smoothing, blurring depth values to help the oracle predict consistent outputs at depth discontinuities [S1].

### Parameters
- **Filter Size $K$**: Size of the 2D neighborhood filter. Example: $K=5$ [S1].
- **Kernel Size $Z$**: Size of the 1D depth filter. Example: $Z=5$ [S1].
- **Loss Function**: Binary Cross-Entropy (BCE) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def filter_depth_oracle_targets(C, K, Z):
    """
    Filters the depth oracle classification targets.
    
    Args:
        C: 3D array of shape (H, W, D) containing binary classification targets.
           C[x, y, z] is 1 if the surface is in segment z, else 0.
        K: Integer, size of the 2D spatial filter (must be odd).
        Z: Integer, size of the 1D depth filter (must be odd).
        
    Returns:
        filtered_C: 3D array of shape (H, W, D) with filtered targets.
    """
    H, W, D = C.shape
    filtered_C = np.zeros_like(C)
    
    half_K = K // 2
    half_Z = Z // 2
    
    # Precompute radial weights for the 2D filter
    # The source formula uses sqrt(2+j^2) in the numerator, which seems specific.
    # However, standard radial distance is sqrt(i^2 + j^2).
    # The source text says "Euclidean distance" and gives specific values for K=5.
    # We implement the source's explicit formula (Eq. 7) but note the potential ambiguity.
    # Source Eq 7: max( C[x+i, y+j] - sqrt(2+j^2) / sqrt(2*half_K) )
    # Note: The source formula in the text has 'j' in the numerator, not 'i' or 'sqrt(i^2+j^2)'.
    # We follow the source's explicit equation.
    
    denom_2d = np.sqrt(2 * half_K)
    
    for x in range(H):
        for y in range(W):
            for z in range(D):
                max_val = -np.inf
                
                # 2D Neighborhood Filter
                for i in range(-half_K, half_K + 1):
                    for j in range(-half_K, half_K + 1):
                        # Boundary check
                        nx = x + i
                        ny = y + j
                        if 0 <= nx < H and 0 <= ny < W:
                            # Source formula: sqrt(2+j^2)
                            # Note: This formula in the source might be a typo for radial distance,
                            # but we replicate it as written in [S1] Eq. 7.
                            weight = np.sqrt(2 + j**2) / denom_2d
                            val = C[nx, ny, z] - weight
                            if val > max_val:
                                max_val = val
                
                # Clamp to [0, 1] implicitly by the nature of max over binary - weight?
                # The source doesn't explicitly clamp the 2D step, but the 1D step clamps to 1.
                # We assume the result is the intermediate value.
                intermediate_val = max_val
                
                # 1D Depth Smoothing Filter
                sum_val = 0.0
                for i in range(-half_Z, half_Z + 1):
                    nz = z + i
                    if 0 <= nz < D:
                        # Triangle kernel weight
                        weight = (half_Z + 1 - abs(i)) / (half_Z + 1)
                        sum_val += intermediate_val * weight # Note: intermediate_val is scalar per (x,y,z)
                        # Wait, the 1D filter is applied to the result of the 2D filter.
                        # The 2D filter produces a value for each (x,y,z).
                        # The 1D filter sums over z for a fixed (x,y).
                        pass
                
                # Correction: The 1D filter should sum the *filtered* values from neighboring depths.
                # Let's restructure the loops for correctness.
                
            # Recompute for clarity
            # Step 1: Compute 2D filtered values for all z at (x,y)
            step1_vals = np.zeros(D)
            for z in range(D):
                max_val = -np.inf
                for i in range(-half_K, half_K + 1):
                    for j in range(-half_K, half_K + 1):
                        nx = x + i
                        ny = y + j
                        if 0 <= nx < H and 0 <= ny < W:
                            weight = np.sqrt(2 + j**2) / denom_2d
                            val = C[nx, ny, z] - weight
                            if val > max_val:
                                max_val = val
                step1_vals[z] = max_val
            
            # Step 2: Apply 1D triangle filter on step1_vals
            sum_val = 0.0
            for i in range(-half_Z, half_Z + 1):
                nz = z + i # This z is the current z in the outer loop? No, we are computing for a specific z.
                # We need to compute the final value for each z.
                pass
            
            # Let's rewrite the inner loop properly
            final_vals = np.zeros(D)
            for z in range(D):
                # 1D filter centered at z
                sum_val = 0.0
                for i in range(-half_Z, half_Z + 1):
                    nz = z + i
                    if 0 <= nz < D:
                        weight = (half_Z + 1 - abs(i)) / (half_Z + 1)
                        sum_val += step1_vals[nz] * weight
                final_vals[z] = min(sum_val, 1.0)
            
            filtered_C[x, y, :] = final_vals
            
    return filtered_C
```

### Edge Cases and Pitfalls
- **Filter Size**: Using an even larger filter size than recommended (e.g., $K > 5$) may reduce overall quality because it places many samples in empty space rather than on the surface [S1].
- **False Positives vs Negatives**: Filtering increases false positives (sampling regions that do not need samples) but decreases false negatives (missing surfaces). This is acceptable because the local raymarching network handles empty space (no image contribution), while missing surfaces clearly reduces image quality [S1].
- **Inference**: No filtering is done during inference; the filter is strictly for training target generation [S1].

## Operating characteristics
- **Performance Gain**: The neighborhood filter adds 1.3 dB–1.8 dB to PSNR, and the depth smoothing filter adds 0.1 dB, measured on test scenes with configuration $K=5, Z=5, I=128$ compared to the baseline [S1].
- **Cost**: The filtering adds no inference cost as it is applied only to training targets [S1].
- **Scaling**: The method scales with the filter sizes $K$ and $Z$. Larger filters increase the computational cost of target generation but may degrade quality if too large [S1].

## Background
DONeRF (Depth Oracle Networks) was introduced to enable real-time rendering of compact neural radiance fields [S1]. The depth oracle filtering technique was developed to address the difficulty of training the oracle to predict consistent outputs at depth discontinuities by providing a multi-class target that is filtered in image-space and along the depth axis [S1].

## Sources
[S1] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
