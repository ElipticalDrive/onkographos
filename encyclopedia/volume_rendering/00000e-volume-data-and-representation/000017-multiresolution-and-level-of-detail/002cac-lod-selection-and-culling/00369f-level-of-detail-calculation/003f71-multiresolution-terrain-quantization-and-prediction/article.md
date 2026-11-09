# Multiresolution Terrain Quantization and Prediction

Also called: Patch vertex classification, Quantization shift assignment

## Overview
This method assigns quantization precision to terrain vertices in a multiresolution hierarchy to minimize bitstream size while maintaining visual fidelity. It solves the problem of efficient compression for multiresolution terrain data by using specific shift values for different vertex types (Corner, Leg, Hypotenuse, Interior). It sets itself apart from generic quantization methods by leveraging the geometric connectivity of patch neighbors (specifically cousin patches and parent-child relationships) to derive shifts for boundary vertices from their children, thereby reducing the number of stored parameters [S1].

## How it works
**Inputs:** A multiresolution terrain mesh represented as a hierarchy of patches. Each patch contains vertices classified as Corners (C), Legs (L), Hypotenuses (H), or Interior (I). A target quantization error $\epsilon_q$ and the local height variation $\Delta z$ for each patch.
**Outputs:** A set of quantization shift values $k$ for each patch, and a prediction scheme that respects these shifts.

**Data Structures:**
*   **Patch:** Contains a list of vertices. Each vertex is classified as C, L, H, or I.
*   **Shift Map:** A mapping from patch ID to shift values. For each patch, three shifts are relevant: $k_L^{left}$, $k_L^{right}$, and $k_{HI}$.
*   **Bitstream:** Variable-length stream of quantized residuals.

**Vertex Classification:**
1.  Vertices on the finest level are assigned a shift of 0 (lossless).
2.  On coarser levels, vertices are classified:
    *   **Corner (C):** Vertices shared by multiple patches at different resolutions.
    *   **Leg (L):** Vertices on the legs of the triangular patch.
    *   **Hypotenuse (H):** Vertices on the hypotenuse of the triangular patch.
    *   **Interior (I):** Vertices strictly inside the patch.

**Quantization Shift Calculation:**
The shift $k$ determines the number of bits to discard. It is calculated as:

$$
k = \begin{cases} \lfloor \log_2 \frac{\epsilon_q}{\Delta z} \rfloor + 1 & \text{if } \epsilon_q \geq \Delta z \\ 0 & \text{otherwise} \end{cases}
$$

Where $\epsilon_q$ is the desired quantization error and $\Delta z$ is the local height variation.

**Shift Assignment Rules:**
1.  **Corners (C):** Left unquantized (shift $k=0$) because they are surrounded by patches at various resolutions, making a single consistent shift difficult to define without excessive storage [S1].
2.  **Legs (L):** The precision for an L vertex is dictated by the finer resolution children. An L vertex in a patch may be an L vertex in an adjacent patch at the same level or an H vertex in a patch one level below. Since children have smaller coarsening errors than parents, the shift is determined by the finest resolution neighbor. Specifically, the shifts for L vertices are derived from the H/I shifts of the patch's children [S1].
3.  **Hypotenuse (H) and Interior (I):** Only the two cousin patches forming a diamond must agree on quantization for H vertices. Since these cousin patches have the same coarsening error, a single shift $k_{HI}$ is used for the union of their H and I vertices. This $k_{HI}$ is stored with the patch [S1].

**Prediction Precision Handling:**
To avoid unwanted low-order bits on patch boundaries, prediction must be performed at a unified precision.
1.  Identify vertices involved in prediction.
2.  Reduce all involved vertices to the precision of the vertex being predicted by shifting out low-order bits.
3.  Perform prediction at this unified precision.
4.  Add the low-precision residual.
5.  Left shift the result to restore the original scale.

This adjustment is necessary only for Corner (C) and Leg (L) vertices in the first triangle of a patch due to regular connectivity and traversal order from diamond interior to boundaries (low to high precision) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_quantization_shifts(patch, epsilon_q, delta_z):
    """
    Calculate the quantization shift for a single patch's H/I vertices.
    """
    if epsilon_q < delta_z:
        return 0
    else:
        return floor(log2(epsilon_q / delta_z)) + 1

def assign_shifts_to_hierarchy(patches, epsilon_q):
    """
    Assign shifts to all patches in the hierarchy.
    patches: list of patch objects, each with children and neighbors.
    """
    # Process from finest to coarsest? Or just compute H/I shifts first.
    # The source implies H/I shifts are stored, L shifts are derived from children.
    
    for patch in patches:
        # 1. Calculate H/I shift for this patch
        # delta_z should be specific to the patch's geometry
        patch.k_HI = calculate_quantization_shifts(patch, epsilon_q, patch.delta_z)
        
        # 2. Corners are always 0 (unquantized)
        for vertex in patch.vertices:
            if vertex.type == 'C':
                vertex.k = 0
            elif vertex.type == 'L':
                # L shift is derived from children's H/I shifts
                # This requires the children to be processed or accessible
                # The source says: "the two L shifts are the H/I shifts of its children"
                # We assume children are available or computed in a prior pass
                pass 
            elif vertex.type in ['H', 'I']:
                vertex.k = patch.k_HI

    # Second pass to resolve L shifts based on children
    for patch in patches:
        if patch.children:
            # Left leg L shift comes from one child's H/I shift
            # Right leg L shift comes from the other child's H/I shift
            # The exact mapping depends on the triangle orientation
            patch.k_L_left = patch.children[0].k_HI
            patch.k_L_right = patch.children[1].k_HI
            
            # Apply to vertices
            for vertex in patch.vertices:
                if vertex.type == 'L':
                    # Determine if it's left or right leg
                    if vertex.is_left_leg:
                        vertex.k = patch.k_L_left
                    else:
                        vertex.k = patch.k_L_right

def predict_with_precision_adjustment(vertex, neighbors, shift):
    """
    Predict a vertex value, adjusting precision to avoid boundary artifacts.
    """
    # 1. Shift out low-order bits from neighbors and the vertex itself
    # to unify precision at 'shift'
    adjusted_neighbors = [n >> shift for n in neighbors]
    adjusted_vertex = vertex >> shift
    
    # 2. Perform prediction at unified precision
    predicted_value = predict(adjusted_neighbors)
    
    # 3. Add the low-precision residual (the bits that were shifted out)
    # This step is conceptual; in practice, the residual is stored
    # and added back during reconstruction.
    # For encoding: residual = (vertex - predicted_value) >> shift
    # For decoding: reconstructed = (predicted_value + residual) << shift
    
    return predicted_value
```

**Parameters:**
*   $\epsilon_q$: Target quantization error. Typical values depend on the application's visual tolerance.
*   $\Delta z$: Local height variation within a patch.

**Edge Cases and Pitfalls:**
*   **Precision Artifacts:** If prediction does not unify precision, unwanted low-order bits appear on patch boundaries [S1].
*   **Boundary Agreement:** L vertices require agreement between four patches (two children, two parents). The method resolves this by deferring to the finest resolution (children) [S1].

## Subtopics

### [GPU Geometry Shader Decoder](10009-gpu-geometry-shader-decoder.md)
The GPU Geometry Shader Decoder is a method for decompressing multiresolution terrain height data directly on the graphics hardware, bypassing the CPU to minimize latency and memory bandwidth usage. It solves the problem of rendering massive terrain datasets by performing variable-length decompression in parallel on the GPU, emitting only height values to a buffer to avoid hardware attribute limits. Unlike methods that cache decompressed geometry or emit full vertex attributes, this approach decodes patches on-the-fly and renders them immediately, optimizing for throughput by maximizing shared memory utilization and minimizing vertex program threads.

## Operating characteristics
*   **Time Complexity:** The shift calculation is $O(1)$ per patch. The assignment requires a traversal of the hierarchy. Prediction is $O(1)$ per vertex.
*   **Memory Complexity:** Only three shift values per patch are stored (one H/I shift, two L shifts derived). This is significantly less than storing shifts for every vertex.
*   **Strengths:** Minimizes bitstream size by exploiting geometric redundancy. Simple bit-shift operations are computationally cheap.
*   **Limitations:** Requires a specific patch hierarchy (diamonds). Precision artifacts can occur if prediction is not handled carefully.
*   **When to Use:** When rendering large multiresolution terrain datasets with limited bandwidth or storage.
*   **When Not to Use:** When lossless reconstruction is required for all vertices (though C vertices are lossless, H/I are quantized).

## Background
This method was introduced in the context of on-the-fly decompression and rendering of multiresolution terrain. It builds on the concept of using bit shifts for quantization and leverages the specific connectivity of triangular patch hierarchies. The approach was developed to enable efficient GPU-based rendering of massive terrain datasets by reducing the data transfer and processing overhead [S1].

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
