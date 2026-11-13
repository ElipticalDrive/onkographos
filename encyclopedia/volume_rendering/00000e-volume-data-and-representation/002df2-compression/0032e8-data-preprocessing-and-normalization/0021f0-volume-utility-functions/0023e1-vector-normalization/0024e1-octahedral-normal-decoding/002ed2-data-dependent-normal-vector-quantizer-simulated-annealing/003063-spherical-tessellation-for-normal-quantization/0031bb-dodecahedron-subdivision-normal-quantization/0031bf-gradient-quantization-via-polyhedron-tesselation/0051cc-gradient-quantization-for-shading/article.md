# Gradient Quantization for Shading

Also called: Gradient Vector Quantization for Table Lookup

## Overview
Gradient Quantization for Shading reduces the precision of gradient direction vectors to a limited bit depth (typically 12–16 bits) to enable shading calculations via a single table lookup, trading geometric accuracy for rendering performance [S1]. Unlike [Octahedral Direction Encoding for Gradient Storage](10730-octahedral-direction-encoding-for-gradient-storage.md), which maps directions to a fixed geometric structure, this method relies on the quantization itself to cluster voxels for view-dependent skipping optimizations in Maximum Intensity Projection (MIP) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** High-precision gradient vectors $\nabla(P)$ for each voxel $P$ in the volume dataset.
*   **Output:** Quantized gradient indices used to index a precomputed shading lookup table (LUT).

**Data Structures:**
*   **Quantized Gradient Index:** An integer value derived from the quantized direction. The bit depth determines the size of the LUT.
*   **Shading Lookup Table (LUT):** A table where the index is the quantized gradient direction and the value is the precomputed shading intensity or color contribution. The size of the LUT is $2^{\text{bit\_depth}}$.

**Parameters:**
*   **Bit Depth:** 12–16 bits per gradient vector [S1]. 12 bits is sufficient for scanned datasets to provide visually satisfactory results [S1].
*   **Exponent $n$:** Used in the intensity equation term $(1 - |\nabla(P) \cdot V|)^n$. For 12-bit quantization, an exponent around 4 is recommended to avoid disturbing artifacts (too bright/dark contributions) caused by quantization error amplification near contours [S1].

**Steps:**
1.  **Gradient Reconstruction:** Compute the gradient vector $\nabla(P)$ for each voxel $P$. High-quality reconstruction methods (e.g., linear regression) are recommended over central differences to avoid staircase artifacts, despite the subsequent quantization [S1].
2.  **Quantization:** Quantize the direction of $\nabla(P)$ to the specified bit depth. This maps the continuous direction space to a discrete set of indices [S1].
3.  **LUT Precomputation:** Precompute the shading values for all possible quantized gradient directions. The effort for computing this table is negligible [S1].
4.  **Shading Lookup:** During rendering, for each voxel, use the quantized gradient index to perform a single table lookup to determine the shading contribution [S1].

**Formulas:**
The intensity contribution depends on the viewing direction $V$ and the gradient magnitude $|\nabla(P)|$. The view-dependent term is:
$$ (1 - |\nabla(P) \cdot V|)^n $$
where $n$ is the exponent [S1].

**Pitfalls:**
*   **Quantization Error Amplification:** High values of $n$ result in very sharp and thin contours. Quantization errors for gradients close to a contour are amplified, leading to too bright or too dark voxel contributions [S1].
*   **Staircase Artifacts:** Using central differences for gradient reconstruction before quantization can introduce staircase artifacts [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase
def preprocess_volume(volume_data, bit_depth=12):
    """
    Quantize gradients and build shading LUT.
    """
    # 1. High-quality gradient reconstruction
    gradients = compute_gradients_high_quality(volume_data)
    
    # 2. Quantize gradient directions
    # Map direction to integer index based on bit_depth
    quantized_indices = quantize_directions(gradients, bit_depth)
    
    # 3. Precompute Shading LUT
    # Size is 2^bit_depth
    lut_size = 1 << bit_depth
    shading_lut = [0.0] * lut_size
    
    # Precompute shading for all possible quantized directions
    # This step is negligible in effort [S1]
    for i in range(lut_size):
        direction = index_to_direction(i, bit_depth)
        # Shading calculation depends on viewing direction V,
        # which is fixed for a frame or interpolated.
        # For simplicity, assume V is passed or averaged.
        shading_lut[i] = compute_shading(direction, V)
        
    return quantized_indices, shading_lut

def render_mip_with_quantization(volume_data, quantized_indices, shading_lut, V, n=4, epsilon=0.01):
    """
    Render using MIP with gradient quantization optimizations.
    """
    # Group voxels by quantized gradient direction
    groups = group_voxels_by_gradient(quantized_indices)
    
    final_image = []
    
    for group_direction, voxels in groups.items():
        # Sort voxels within group by gradient magnitude descending
        voxels.sort(key=lambda v: v.gradient_magnitude, reverse=True)
        
        # Skip group if representative intensity is below threshold
        # Representative can be the first voxel (highest magnitude)
        if not voxels:
            continue
            
        representative = voxels[0]
        intensity_rep = compute_intensity(representative, V, n)
        
        if intensity_rep < epsilon:
            continue # Skip entire group
            
        # Process voxels in group
        for voxel in voxels:
            intensity = compute_intensity(voxel, V, n)
            if intensity < epsilon:
                break # Stop processing group
            
            # Update MIP
            update_mip(final_image, voxel, intensity)
            
    return final_image

def compute_intensity(voxel, V, n):
    """
    Compute intensity based on gradient and viewing direction.
    """
    grad_mag = voxel.gradient_magnitude
    grad_dir = voxel.gradient_direction
    # Term: (1 - |grad_dir . V|)^n
    # Note: grad_dir is normalized, V is normalized
    dot_product = abs(dot(grad_dir, V))
    term = (1.0 - dot_product) ** n
    return grad_mag * term # Simplified intensity model
```

## Subtopics

### MIP Voxel Skipping by Gradient Grouping
An optimization for Maximum Intensity Projection (MIP) that groups voxels by quantized gradient direction to skip non-contributing groups efficiently [S1]. It exploits the view-dependence of the contour intensity term $(1 - |\nabla(P) \cdot V|)^n$ [S1].
*   **Steps:** Group voxels with the same or similar gradient direction (achieved by quantization). Sort voxels within each group by gradient magnitude in descending order. Skip an entire group if the intensity $s(P, V)$ of a representative voxel is below a threshold $\varepsilon$. Within a group, stop processing as soon as the first voxel with intensity $I(P, V)$ falls below $\varepsilon$ [S1].
*   **Performance:** Over 75% of all voxels can be skipped for typical datasets [S1].
*   **Pitfall:** Restricts compositing to maximum intensity selection. Blending is not feasible because voxels are projected in arbitrary spatial order (sorted by gradient, not depth) [S1].

## Operating characteristics
*   **Time Complexity:** Preprocessing is negligible due to the simplicity of the LUT computation [S1]. Rendering is accelerated by skipping non-contributing voxels.
*   **Memory Complexity:** Requires storage for the quantized gradient indices and the shading LUT ($2^{\text{bit\_depth}}$ entries).
*   **Measured Performance:** On a $256^2 \times 241$ dataset, MIP rendering took 130ms, while blending took 270ms (Table 1 in [S1]). MIP is faster due to more efficient skipping and simpler compositing [S1].
*   **Strengths:** Enables interactive frame rates for contour-based volume rendering. High skipping rates (75%+) for MIP.
*   **Limitations:** Quantization reduces accuracy. Blending is not supported with the MIP skipping optimization. High exponents $n$ cause artifacts with low bit depths [S1].
*   **When to use:** Interactive contour visualization of scanned datasets.
*   **When not to use:** When high-precision gradient reconstruction is required for blending, or when central differences are used (causes artifacts).

## Background
Originated in the context of fast visualization of object contours by non-photorealistic volume rendering [S1]. The method leverages the fact that voxels with high gradient magnitude but unfavorable orientation to the viewing direction contribute little to the final image, allowing for aggressive skipping when gradients are quantized and grouped [S1].

## Sources
[S1] Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering, https://doi.org/10.1111/1467-8659.00538, fetched 2026-11-01
