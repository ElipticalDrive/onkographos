# Partial Pre-Integration


## Overview
Partial Pre-Integration is a GPU-accelerated cell projection technique that improves the accuracy of ray integration approximation in volume rendering by utilizing a 2D lookup table ($\psi$ table) that is independent of the transfer function [S1]. Unlike Full Pre-Integration, which requires recomputing the lookup table whenever the transfer function changes, Partial Pre-Integration allows for interactive transfer function updates with minimal overhead [S1]. It builds on [Volume Raycasting Discretization](831-volume-raycasting-discretization.md) by replacing per-sample ray marching with a single fragment shader lookup, trading some computational speed for reduced under-sampling errors [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar field values at the entry ($s_f$) and exit ($s_b$) points of a voxel cell, the geometric thickness of the cell along the ray ($l$), and a transfer function $T(s)$ that maps scalar values to RGBA colors.
*   **Output:** The final RGBA color contribution of the cell to the image.

**Data Structures:**
*   **2D $\psi$ Table:** A pre-compiled 2D texture or array. The dimensions and content are fixed and independent of the visualization attributes or transfer function [S1]. The table stores pre-computed integration results based on optical thickness and absorption properties derived from the scalar values.

**Steps:**
1.  **Retrieve Colors:** Sample the transfer function $T$ at the entry scalar value $s_f$ and the exit scalar value $s_b$ to obtain their associated RGBA colors, denoted as $C_f$ and $C_b$ [S1].
2.  **Compute Indices:** Use the retrieved colors $C_f$, $C_b$, and the cell thickness $l$ to compute the indices for accessing the 2D $\psi$ table [S1]. The specific mapping from $(C_f, C_b, l)$ to table coordinates is determined by the pre-integration formulation, typically involving the optical thickness and absorption coefficients derived from the alpha components of $C_f$ and $C_b$ [S1].
3.  **Lookup and Compose:** Retrieve the pre-computed values from the $\psi$ table using the computed indices [S1]. Use these values to compute the final fragment color for the cell, effectively performing the integration of the volume rendering integral over the cell span [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partial_pre_integration(s_f, s_b, l, transfer_function, psi_table):
    """
    Computes the color contribution of a voxel cell using Partial Pre-Integration.
    
    Args:
        s_f (float): Scalar value at the entry point of the cell.
        s_b (float): Scalar value at the exit point of the cell.
        l (float): Thickness of the cell along the ray.
        transfer_function (function): Maps scalar s to RGBA color.
        psi_table (2D array): Pre-compiled lookup table independent of transfer function.
        
    Returns:
        tuple: Final RGBA color for the cell.
    """
    # Step 1: Retrieve RGBA colors associated with sf and sb from the transfer function
    C_f = transfer_function(s_f)
    C_b = transfer_function(s_b)
    
    # Step 2: Compute indices into the 2D psi table
    # The indices are derived from the colors and thickness.
    # In standard partial pre-integration, this involves calculating
    # optical thickness and absorption parameters from C_f and C_b.
    # Let alpha_f, alpha_b be the alpha components of C_f, C_b.
    # Let color_f, color_b be the RGB components.
    
    # Note: The exact formula for index computation depends on the specific
    # definition of the psi table by Moreland et al. [S1].
    # Typically, indices are based on the optical thickness tau and absorption.
    # For implementation, one must map (C_f, C_b, l) to 2D coordinates (u, v).
    
    # Placeholder for the specific index calculation logic defined by Moreland et al.
    # This part is not explicitly detailed in the provided notes beyond "compute the indices".
    # A common approach involves normalizing the alpha values and thickness.
    u = compute_index_u(C_f, C_b, l)
    v = compute_index_v(C_f, C_b, l)
    
    # Step 3: Retrieve values from the psi table and use them to compute the final fragment color
    psi_value = psi_table[u, v]
    
    # The final color is computed using the psi value and the retrieved colors.
    # Again, the exact composition formula is part of the Moreland et al. method.
    final_color = compute_final_color(psi_value, C_f, C_b)
    
    return final_color

def compute_index_u(C_f, C_b, l):
    # Implementation detail not fully specified in notes.
    # Requires mapping color/thickness to table dimension.
    return 0 # Placeholder

def compute_index_v(C_f, C_b, l):
    # Implementation detail not fully specified in notes.
    return 0 # Placeholder

def compute_final_color(psi_value, C_f, C_b):
    # Implementation detail not fully specified in notes.
    return (0, 0, 0, 0) # Placeholder
```

**Parameters:**
*   **$\psi$ Table Resolution:** Not given in the sources.
*   **Thickness ($l$):** The geometric length of the ray segment within the voxel.

**Edge Cases and Pitfalls:**
*   **Under-sampling:** Partial Pre-Integration is less prone to under-sampling errors compared to simple average scalar methods [S1].
*   **Performance:** It is slower than the simple average scalar method (original PT by Shirley and Tuchman) [S1].
*   **Transfer Function Changes:** Unlike Full Pre-Integration, the $\psi$ table does not need to be recomputed when the transfer function changes. However, the lookup texture (if implemented as a texture) may need to be reloaded if the implementation relies on transfer-function-dependent textures, though the notes state the $\psi$ table itself is pre-compiled and independent [S1].

## Subtopics

### [Hybrid Volume and Isosurface Rendering with Partial Pre-Integration](1311-hybrid-volume-and-isosurface-rendering-with-partial-pre-inte.md)
A GPU-accelerated cell projection method that combines direct volume rendering with multiple isosurfaces. It uses tetrahedral discretization, requires no cell connectivity, and offers lower memory usage than hardware-accelerated ray casting variants.

### [Semi-Analytical Fragment Shader Preintegration](1379-semi-analytical-fragment-shader-preintegration.md)
Improves accuracy by adaptively subdividing ray segments within a cell based on local extrema of scalar and shading polynomials, allowing lookup-based integration over smaller intervals.

## Operating characteristics
*   **Time Complexity:** Slower than the simple average scalar method (original PT) [S1].
*   **Memory Complexity:** Requires storage for the 2D $\psi$ table. The table is pre-compiled and independent of visualization attributes [S1].
*   **Strengths:** Less prone to under-sampling errors than simple averaging; allows interactive transfer function updates without full table recomputation [S1].
*   **Limitations:** Slower than basic averaging methods [S1].
*   **When to Use:** When interactive transfer function exploration is required and under-sampling artifacts are a concern, and performance is acceptable [S1].
*   **When Not to Use:** When maximum performance is critical and interactive transfer function changes are not needed (use Full Pre-Integration or simple averaging) [S1].

## Background
Partial Pre-Integration was introduced by Moreland et al. [S1]. It was developed to overcome the disadvantage of Full Pre-Integration, where the pre-integration table must be recomputed every time the transfer function changes, which can be an expensive operation depending on table resolution [S1]. The 2D $\psi$ table used in this method was computed by Moreland et al. and made available for downloading [S1]. An implementation using fragment shaders was described in a prototype programmed in C++ using OpenGL 2.0 with GLSL [S1].

## Sources
[S1] GPU-Based Cell Projection for Interactive Volume Rendering, https://graphics.tudelft.nl/~marroquim/publications/pdfs/marroquim-sibgrapi2006.pdf, fetched 2026-10-27
