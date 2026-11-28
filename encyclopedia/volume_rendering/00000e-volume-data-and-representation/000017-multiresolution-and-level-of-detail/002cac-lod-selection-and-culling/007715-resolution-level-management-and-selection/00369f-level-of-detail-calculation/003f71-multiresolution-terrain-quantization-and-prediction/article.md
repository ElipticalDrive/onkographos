# Multiresolution Terrain Quantization and Prediction

Also called: Patch vertex classification, Quantization shift assignment

## Overview
This method assigns quantization precision to terrain vertices in a multiresolution hierarchy to minimize bitstream size while maintaining visual fidelity [S1]. It sets itself apart from generic quantization methods by leveraging the geometric connectivity of patch neighbors to derive shifts for boundary vertices from their children, thereby reducing the number of stored parameters [S1].

## How it works
**Inputs:** A multiresolution terrain hierarchy and a target quantization error $\epsilon_q$ [S1].
**Outputs:** Quantization shift values $k$ for each vertex type and a prediction scheme that respects these shifts [S1].

**Data Structures:**
*   **Patch:** Contains vertices classified as Corners (C), Legs (L), Hypotenuses (H), or Interior (I) [S1].
*   **Bitstream:** Variable-length stream storing quantized residuals [S1].

**Vertex Classification:**
1.  Assign shift $k=0$ to all vertices on the finest level for lossless coding [S1].
2.  Classify vertices on non-finest levels as boundary (C, L, H) and interior (I) [S1].
3.  Leave Corner (C) vertices unquantized (full precision) because they are surrounded by patches at multiple resolutions [S1].

**Quantization Shift Calculation:**
The shift $k$ determines the number of bits to discard and is calculated as:

$$k = \begin{cases} [\log_2 \frac{\epsilon_q}{\Delta z}] + 1 & \text{if } \epsilon_q \geq \Delta z \\ 0 & \text{otherwise} \end{cases}$$

Where $\epsilon_q$ is the target quantization error, $\Delta z$ is the local height variation, and $[\cdot]$ denotes rounding to the nearest integer [S1].

**Shift Assignment Rules:**
1.  **Legs (L):** An L vertex in a patch may be an L vertex in an adjacent patch at the same level or an H vertex in a patch one level below [S1]. The precision is dictated by the finer resolution children to ensure coarsening errors are no larger [S1].
2.  **Hypotenuse (H) and Interior (I):** Only the two cousin patches forming a diamond must agree on quantization for H vertices [S1]. Since they have the same coarsening error, the same shift is used for the union of their H and I vertices [S1].
3.  **Storage:** Three shift values are needed per patch for the decoder: two for the legs (L) and one for the remaining H and I vertices [S1]. Only the H/I shift is stored with the patch; the two L shifts are derived as the H/I shifts of its children [S1].

**Prediction Precision Handling:**
1.  Vertices involved in prediction must be reduced to the precision of the vertex being predicted to avoid unwanted low-order bits on patch boundaries [S1].
2.  Shift out low-order bits, perform prediction at unified precision, add low-precision residual, then left shift result [S1].
3.  This adjustment is necessary only for C and L vertices in the first triangle of a patch due to regular connectivity and traversal order from diamond interior to boundaries (low to high precision) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_quantization_shift(epsilon_q, delta_z):
    """Computes the quantization shift k for a patch's H/I vertices."""
    if epsilon_q < delta_z:
        return 0
    return round(math.log2(epsilon_q / delta_z)) + 1

def assign_quantization_shifts(patches):
    """
    Assigns quantization shifts to all vertices in a multiresolution terrain hierarchy.
    patches: list of patch objects with attributes: children, vertices, epsilon_q, delta_z
    """
    for patch in patches:
        # 1. Calculate H/I shift for the current patch
        patch.k_HI = compute_quantization_shift(patch.epsilon_q, patch.delta_z)
        
        # 2. Assign shifts to vertices based on classification
        for v in patch.vertices:
            if v.type == 'C':
                v.k = 0  # Corners left unquantized (full precision)
            elif v.type in ('H', 'I'):
                v.k = patch.k_HI
            elif v.type == 'L':
                # L shifts are derived from the H/I shifts of the patch's children
                if patch.children:
                    v.k = patch.children[0].k_HI if v.is_left_leg else patch.children[1].k_HI
                else:
                    v.k = 0  # Finest level vertices get shift 0

def adjust_prediction_precision(vertex_val, neighbor_vals, shift):
    """
    Adjusts prediction precision to avoid low-order bit artifacts on boundaries.
    Shifts out low-order bits, performs prediction, adds residual, and left shifts.
    """
    # Shift out low-order bits to unify precision
    shifted_vertex = vertex_val >> shift
    shifted_neighbors = [n >> shift for n in neighbor_vals]
    
    # Perform prediction at unified precision (standard graphics operation [G])
    predicted = sum(shifted_neighbors) // len(shifted_neighbors)
    
    # Extract and average low-order residuals
    mask = (1 << shift) - 1
    residual_vertex = vertex_val & mask
    residual_neighbors = [n & mask for n in neighbor_vals]
    avg_residual = sum(residual_neighbors) // len(residual_neighbors)
    
    # Left shift result to restore scale
    return (predicted + avg_residual) << shift
```

**Parameters:**
*   $\epsilon_q$: Target quantization error [S1].
*   $\Delta z$: Local height variation for each patch [S1].

**Edge Cases and Pitfalls:**
*   **Precision Artifacts:** If prediction does not unify precision, unwanted low-order bits appear on patch boundaries [S1].
*   **Boundary Agreement:** L vertices require agreement between four patches (two children, two parents); the method resolves this by deferring to the finest resolution (children) [S1].

## Subtopics

### [GPU Geometry Shader Decoder](10009-gpu-geometry-shader-decoder.md)
The GPU Geometry Shader Decoder is a method for decompressing multiresolution terrain height data directly on the graphics hardware, bypassing the rasterizer to avoid hardware attribute limits. It solves the problem of rendering massive terrain datasets by performing variable-length decompression in parallel on the GPU, emitting only height values to a buffer to avoid the 1,024 scalar attribute limit. Unlike methods that cache decompressed geometry or emit full vertex attributes, this approach decodes patches on-the-fly and renders them immediately, optimizing for throughput by maximizing shared memory utilization and minimizing vertex program threads.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Only three shift values per patch are stored (one H/I shift, two L shifts derived) [S1].
*   **Strengths:** Minimizes bitstream size by exploiting geometric redundancy and neighbor agreement [S1].
*   **Limitations:** Requires specific patch hierarchy connectivity; precision artifacts occur if prediction does not unify precision [S1].
*   **When to Use:** Multiresolution terrain rendering where bitstream size and decoder simplicity are prioritized [S1].
*   **When Not to Use:** Not given in the sources.

## Background
This method was introduced in the context of on-the-fly decompression and rendering of multiresolution terrain [S1]. It builds on the concept of using bit shifts for quantization and leverages the specific connectivity of triangular patch hierarchies [S1].

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
