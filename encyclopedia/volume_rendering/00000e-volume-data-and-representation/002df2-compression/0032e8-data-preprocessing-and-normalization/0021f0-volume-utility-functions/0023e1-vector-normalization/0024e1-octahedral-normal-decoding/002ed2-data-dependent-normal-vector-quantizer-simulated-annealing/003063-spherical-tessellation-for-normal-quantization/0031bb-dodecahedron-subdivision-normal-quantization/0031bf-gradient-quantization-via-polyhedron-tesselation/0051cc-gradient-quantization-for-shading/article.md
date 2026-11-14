# Gradient Quantization for Shading

Also called: Gradient Quantization via Polyhedron Tesselation (Shading Variant)

## Overview
Gradient Quantization for Shading is a method that quantizes gradient direction vectors to a limited bit depth (typically 12–16 bits) to enable shading computation via a single table lookup, trading accuracy for rendering performance [S1]. It distinguishes itself from its parent method, [Gradient Quantization via Polyhedron Tesselation](6591-gradient-quantization-via-polyhedron-tesselation.md), by focusing specifically on the optimization of the shading term in non-photorealistic volume rendering, and from its sibling [Octahedral Direction Encoding for Gradient Storage](10730-octahedral-direction-encoding-for-gradient-storage.md) by utilizing a polyhedral tessellation approach rather than octahedral projection [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset where each voxel has a scalar value and a computed gradient vector $\nabla(P)$.
*   **Output:** A quantized representation of the gradient direction for each voxel, used to index a pre-computed shading lookup table.

**Data Structures:**
*   **Quantized Gradient Index:** An integer value derived from the gradient direction, stored per voxel. The bit depth is typically 12–16 bits [S1].
*   **Shading Lookup Table:** A pre-computed table indexed by the quantized gradient direction, containing the shading contribution for that direction relative to a light source or viewing vector.

**Steps:**
1.  **Gradient Reconstruction:** Compute the gradient vector $\nabla(P)$ for each voxel. High-quality reconstruction methods (e.g., linear regression) are recommended over central differences to avoid staircase artifacts [S1].
2.  **Quantization:** Quantize the direction of $\nabla(P)$ to a limited bit depth (12–16 bits) using the tessellation scheme defined in [Gradient Quantization via Polyhedron Tesselation](6591-gradient-quantization-via-polyhedron-tesselation.md). For scanned datasets, 12 bits is sufficient for visually satisfactory results [S1].
3.  **Lookup Table Pre-computation:** Pre-compute the shading values for all possible quantized gradient directions. The effort for this is negligible [S1].
4.  **Shading Computation:** During rendering, retrieve the quantized gradient index for the current voxel and perform a single table lookup to obtain the shading contribution.

**Formulas:**
The intensity contribution of a voxel depends on the viewing direction $V$ and the gradient magnitude $|\nabla(P)|$. The contour intensity term includes:
$$ (1 - |\nabla(P) \cdot V|)^n $$
where $n$ is an exponent controlling the sharpness of the contour [S1].

**Parameters:**
*   **Bit Depth:** 12–16 bits per gradient vector. 12 bits is recommended for scanned datasets [S1].
*   **Exponent $n$:** For 12-bit quantization, an exponent around 4 is recommended. Higher values amplify quantization errors for gradients close to a contour, causing disturbing bright/dark artifacts [S1].

**Edge Cases and Pitfalls:**
*   **Quantization Error Amplification:** High values of $n$ combined with gradient quantization can result in too bright or too dark voxel contributions for gradients near contours [S1].
*   **Staircase Artifacts:** Using central differences for gradient reconstruction can introduce staircase artifacts; high-quality reconstruction is recommended even with quantization [S1].
*   **Empty Space:** Voxels with $g(|\nabla(P)|) = 0$ can be skipped using established empty space encoding techniques [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase
def preprocess_volume(volume_data):
    """
    Computes gradients, quantizes them, and prepares for shading lookup.
    """
    # 1. Gradient Reconstruction
    # Use high-quality method (e.g., linear regression) to avoid staircase artifacts [S1]
    gradients = compute_gradients_high_quality(volume_data)
    
    # 2. Quantization
    # Quantize direction to 12-16 bits using polyhedron tessellation [S1]
    # The specific tessellation logic is defined in [Gradient Quantization via Polyhedron Tesselation](6591-gradient-quantization-via-polyhedron-tesselation.md)
    quantized_indices = quantize_gradient_directions(gradients, bit_depth=12)
    
    # 3. Store indices for fast access
    return quantized_indices

def compute_shading_lookup_table(bit_depth=12):
    """
    Pre-compute shading values for all quantized gradient directions.
    """
    num_entries = 2 ** bit_depth
    table = [0.0] * num_entries
    
    # Iterate over all possible quantized directions
    for i in range(num_entries):
        # Reconstruct direction from index (inverse of quantization)
        direction = reconstruct_direction_from_index(i)
        
        # Compute shading contribution
        # Example: simple diffuse shading with light vector L
        # shading = max(0, dot(direction, L))
        # This depends on the specific shading model, but the key is single lookup [S1]
        shading_value = compute_shading_for_direction(direction)
        table[i] = shading_value
        
    return table

# Rendering Phase
def render_voxel_shading(voxel_gradient_index, shading_table, exponent_n=4):
    """
    Compute shading for a single voxel using table lookup.
    """
    # 1. Table Lookup
    shading_factor = shading_table[voxel_gradient_index]
    
    # 2. Apply contour intensity term
    # Note: The exact intensity formula involves viewing direction V and gradient magnitude
    # I(P, V) depends on |grad(P)| and the term (1 - |grad(P) . V|)^n [S1]
    # Here we assume shading_factor incorporates the directional component
    # and the magnitude component is handled separately or within the table if pre-computed for magnitude ranges.
    # Based on [S1], the table lookup replaces the expensive dot product calculation.
    
    return shading_factor
```

## Subtopics

### [Dual Access Gradient Quantization](14819-dual-access-gradient-quantization.md)
Dual Access Gradient Quantization is a data structure and algorithm for accelerating gradient-dependent volume rendering, specifically silhouette extraction and shading. It solves the problem of efficiently storing and retrieving quantized gradient vectors for every voxel, reducing memory usage from 12 bytes (three 32-bit floats) to 2 bytes per voxel while maintaining fast access times. It distinguishes itself from standard quantization by providing "dual access": it supports immediate lookup for pre-computed indices and allows fast approximation for arbitrary normalized vectors via a pre-filled index table, minimizing expensive inner product calculations during rendering.

## Operating characteristics
*   **Time Complexity:** Shading computation is reduced to a single table lookup, significantly accelerating the rendering process compared to computing dot products for every voxel [S1].
*   **Memory Complexity:** Quantization reduces the storage requirement for gradient directions. The parent method uses tessellation vertices; this variant specifies 12–16 bits per vector [S1].
*   **Performance:** For typical datasets, over 75% of voxels can be skipped using optimizations like MIP Voxel Skipping by Gradient Grouping, which relies on this quantization [S1].
*   **Strengths:** Enables interactive frame rates for non-photorealistic volume rendering by replacing expensive shading calculations with lookups [S1].
*   **Limitations:** Quantization introduces errors, particularly for gradients near contours when high exponents are used. Blending is not feasible with certain optimizations (like MIP Voxel Skipping) due to arbitrary spatial ordering [S1].
*   **When to Use:** When interactive rendering of non-photorealistic contour-based volume data is required, and visual fidelity can tolerate minor quantization artifacts.
*   **When Not to Use:** When high-precision gradient-based shading is critical, or when blending compositing operations are required in conjunction with MIP skipping optimizations [S1].

## Background
This method originates from the work on "Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering" [S1]. It builds on the polyhedral tessellation approach to quantize gradient directions, aiming to optimize the shading term in volume rendering. The technique was developed to address the computational cost of gradient-dependent shading in large volumetric datasets, enabling interactive frame rates.

## Sources
[S1] Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering, https://doi.org/10.1111/1467-8659.00538, fetched 2026-11-01
