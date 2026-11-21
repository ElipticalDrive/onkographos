# LLO-Based Shear-Warp Rendering

Also called: New shear-warp algorithm

## Overview
LLO-Based Shear-Warp Rendering is a volume rendering method that combines the Shear-warp factorization with a Linear Level Octree (LLO) spatial data structure to optimize 3D visualization [S1]. It solves the performance bottleneck of hierarchical traversal in standard octree-based rendering by pre-computing display order arrays, allowing the algorithm to process each octant only once using 3D spatial coherency rather than the 1D coherency of pure shear-warp [S1]. Unlike pure shear-warp which requires storing three copies of Run-Length Encoded (RLE) data, this method stores only the 'black' (non-transparent) octants of a single LLO, reducing memory usage and enabling efficient supersampling [S1].

## How it works
The method decomposes the rendering pipeline into preprocessing, display order preparation, and the rendering loop, leveraging the LLO structure to avoid runtime tree traversal.

### Inputs and Outputs
*   **Input:** Classified volume data (voxel intensities), viewing parameters (view direction vector $V = (V_x, V_y, V_z)$, projection type), and a user-defined intensity variation threshold $T$.
*   **Output:** Final 2D image.

### Data Structures
**Linear Level Octree (LLO):**
The volume is converted into an LLO representation. Each leaf node (octant) stores:
1.  **Code Key:** A tuple $(L_w, x/2^{l_L}, y/2^{l_L}, z/2^{l_L})$ where $L_w$ is the ancestor level, $L$ is the current level, and $l_L = L - L_w$ [S1].
2.  **Vertex Voxels:** The 8 corner voxel values of the octant.
3.  **Gradients:** Precomputed gradients for all 8 vertices of the octant.
4.  **Size:** The smallest octant size is $2 \times 2 \times 2$ voxels [S1].

**Display Order Arrays:**
Four arrays are pre-computed, each storing pointers to leaf nodes ordered by display priority for one of the four distinctive view direction sign combinations. During rendering, one array is selected based on the current view direction [S1].

### Preprocessing
1.  **LLO Generation:** Traverse the volume and generate LLO leaf nodes based on the following criteria [S1]:
    *   Minimum octant size: $2 \times 2 \times 2$ voxels.
    *   Intensity variation constraint: Let $Max$ and $Min$ be the maximum and minimum intensity values in an octant. The octant is a leaf if $Max - Min \le T$, where $T$ is a user-defined threshold ($T \neq 0$).
    *   Transparency: If all voxels in an octant are transparent, it is marked as "white" and excluded from the LLO (or handled as empty).
2.  **Gradient Computation:** Compute and store gradients for all 8 vertices of each leaf octant.
3.  **Display Order Pre-computation:** Traverse the LLO once for each of the four distinctive view direction sign patterns. Store pointers to leaf nodes in four separate arrays ordered by display priority. The display order of child nodes (0-7) depends on the signs of the view direction vector components $(V_x, V_y, V_z)$ [S1]. There are 8 cases based on signs, but only 4 distinctive orders exist (others are inverse). The mapping is defined in Table 1 below.

**Table 1: Display Orders of Child Nodes Based on View Direction Signs**

| View Direction Signs $(V_x, V_y, V_z)$ | Display Order (Node Indices) |
| :--- | :--- |
| $(<0, <0, <0)$ | 7, 3, 5, 6, 1, 2, 4, 0 |
| $(>0, <0, <0)$ | 6, 2, 4, 7, 0, 3, 5, 1 |
| $(<0, >0, <0)$ | 5, 1, 4, 7, 0, 3, 6, 2 |
| $(>0, >0, <0)$ | 4, 0, 5, 6, 1, 2, 7, 3 |
| $(<0, <0, >0)$ | 3, 1, 2, 7, 0, 5, 6, 4 |
| $(>0, <0, >0)$ | 2, 0, 3, 6, 1, 4, 7, 5 |
| $(<0, >0, >0)$ | 1, 3, 6, 4, 0, 2, 5, 7 |
| $(>0, >0, >0)$ | 0, 1, 2, 4, 3, 5, 6, 7 |

*Note: The order for $(<0, >0, >0)$ is inferred from the pattern and the statement that there are only 4 distinctive orders and their inverses. The source explicitly lists 6 cases and states the remaining are inverse or derived [S1].*

### Rendering Loop
1.  **Select Array:** Determine the signs of the view direction vector $V$. Select the corresponding pre-computed display order array.
2.  **Iterate Octants:** Process each octant pointer in the selected array. For each octant:
    a.  **Transform:** Transform the octant into the sheared object space. If the octant is a child of a parent node, its transformation is computed efficiently from the parent's transformation [S1].
    b.  **Project:** Project the sheared octant onto the intermediate image plane.
    c.  **Composite:** Composite the projection into the intermediate image buffer. The intermediate image is run-length encoded (RLE) to skip opaque pixels quickly. Only non-transparent sub-volumes (octants) are passed to the renderer [S1].
    d.  **Interpolate Samples:** Compute shaded samples for the octant. Since all samples within an octant are interpolated using the same 8 vertices, incremental computation is used to reduce cost. The precomputed vertex gradients are used for efficient gradient interpolation [S1].
3.  **Finalization:** After all octants are processed, warp the intermediate image into the final image space using an inexpensive bilinear filter [S1].

### Multi-modality Support
Multiple modalities can be encoded into the LLO. They are integrated using octree "OR" operations or integration functions during pre-processing, rendering, or post-processing. The integrated LLO is then visualized using the standard algorithm, saving memory as no additional volume modalities are stored separately [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def llo_shear_warp_render(volume, view_direction, threshold_T):
    """
    Main entry point for LLO-based Shear-Warp Rendering.
    """
    # 1. Preprocessing
    llo = convert_volume_to_llo(volume, threshold_T)
    gradients = compute_vertex_gradients(llo)
    
    # 2. Pre-compute Display Orders
    # There are 4 distinctive sign patterns for (Vx, Vy, Vz)
    # We generate 4 arrays of leaf pointers
    display_arrays = precompute_display_arrays(llo)
    
    # 3. Determine which array to use
    signs = get_signs(view_direction)
    selected_array = get_array_for_signs(display_arrays, signs)
    
    # 4. Rendering
    intermediate_image = create_intermediate_image_buffer()
    
    for octant_ptr in selected_array:
        octant = llo.get_octant(octant_ptr)
        
        # Get transformation for this octant
        # If hierarchical, transform is derived from parent
        transform = get_octant_transform(octant, view_direction)
        
        # Project and Composite
        # The octant is sheared and projected onto the intermediate plane
        # Samples are interpolated using the 8 vertices and precomputed gradients
        # Incremental computation is used for efficiency
        project_and_composite(octant, transform, gradients, intermediate_image)
        
    # 5. Final Warp
    final_image = warp_intermediate_to_final(intermediate_image, view_direction)
    
    return final_image

def convert_volume_to_llo(volume, threshold_T):
    """
    Converts volume to LLO based on leaf criteria.
    """
    llo = LinearLevelOctree()
    traverse_and_build_llo(volume, llo, threshold_T)
    return llo

def traverse_and_build_llo(volume, llo, threshold_T, current_level=0):
    """
    Recursive traversal to build LLO.
    """
    # Check if current region is a leaf
    min_val, max_val = get_min_max_volume_region(volume, current_bounds)
    
    # Leaf criteria 1: Minimum size 2x2x2
    if is_min_size(current_bounds):
        llo.add_leaf(current_bounds, min_val, max_val)
        return

    # Leaf criteria 2: Intensity variation <= T
    if (max_val - min_val) <= threshold_T:
        llo.add_leaf(current_bounds, min_val, max_val)
        return

    # Leaf criteria 3: All transparent (handled by excluding from LLO)
    if is_all_transparent(volume, current_bounds):
        return

    # Not a leaf, recurse
    sub_bounds = split_bounds(current_bounds)
    for sub_bound in sub_bounds:
        traverse_and_build_llo(volume, llo, threshold_T, current_level + 1, sub_bound)

def precompute_display_arrays(llo):
    """
    Traverses LLO once for each of the 4 distinctive view directions.
    Stores pointers to leaf nodes in 4 different arrays.
    """
    # Define the 4 distinctive sign patterns (others are inverse)
    patterns = [
        (NEG, NEG, NEG),
        (POS, NEG, NEG),
        (NEG, POS, NEG),
        (POS, POS, NEG)
    ]
    
    arrays = [None] * 4
    for i, pattern in enumerate(patterns):
        arrays[i] = []
        traverse_llo_for_order(llo, pattern, arrays[i])
        
    return arrays

def traverse_llo_for_order(llo, view_signs, output_array):
    """
    Recursive traversal to collect leaf pointers in display order.
    """
    # Use Table 1 to determine child node order based on view_signs
    child_order = get_child_order_from_table(view_signs)
    
    for child_idx in child_order:
        child_node = llo.get_child(child_idx)
        if child_node.is_leaf():
            output_array.append(child_node.pointer)
        else:
            # Recurse into child
            # Note: The view signs for the child might need adjustment 
            # relative to the child's local coordinate system, 
            # but the source implies a global view direction determines the order.
            # The source states "display orders of nodes from different view directions are given in Table 1"
            # and "traverse the LLO once for each of the four distinctive view directions".
            # This implies the order is determined globally.
            traverse_llo_for_order(child_node, view_signs, output_array)

def project_and_composite(octant, transform, gradients, intermediate_image):
    """
    Projects the octant into the sheared space and composites it.
    """
    # Transform octant to sheared object space
    sheared_octant = apply_transform(octant, transform)
    
    # Project onto intermediate image plane
    # Use RLE to skip opaque pixels
    rle_data = project_to_rle(sheared_octant)
    
    # Interpolate samples using 8 vertices and gradients
    # Incremental computation for efficiency
    samples = interpolate_samples(octant, gradients)
    
    # Composite into intermediate image
    composite_rle_into_image(rle_data, samples, intermediate_image)
```

## Operating characteristics
*   **Time Complexity:** The preprocessing step involves a single traversal of the LLO to build the structure and four traversals to build the display order arrays. The rendering loop avoids hierarchical traversal, processing each octant only once. This significantly reduces computing cost compared to conventional hierarchical structures where traversal is the most time-consuming operation [S1].
*   **Memory Complexity:** Stores only the 'black' (non-transparent) octants of a single LLO, along with vertex voxels and gradients. This is more memory-efficient than pure shear-warp which requires storing three copies of RLE-encoded data [S1].
*   **Performance:** Designed for fast 3D volume visualization on standard PCs [S1]. The use of 3D coherency and incremental computation reduces the cost of shading and interpolation [S1].
*   **Strengths:** Avoids runtime tree traversal; leverages 3D spatial coherency; supports multi-modality integration with memory savings; allows efficient supersampling in x, y, and z directions [S1].
*   **Limitations:** Requires preprocessing to build LLO and display order arrays; limited to 4 distinctive view direction orders (though this covers all octants); relies on the assumption that intensity variation within an octant is low (controlled by $T$) [S1].
*   **When to use:** PC-based medical simulation and visualization where interactive performance is needed and the volume data can be classified and preprocessed [S1].
*   **When not to use:** Dynamic volumes where preprocessing overhead is prohibitive; scenes with high-frequency intensity variations that cannot be captured by the threshold $T$ without excessive leaf nodes [S1].

## Background
The LLO-Based Shear-Warp Rendering method was introduced in a paper titled "Shear-Warp Volume Rendering Algorithms Using Linear Level Octree for PC-Based Medical Simulation" [S1]. It was developed as part of an effort to create PC-based interactive medical simulators, specifically for visualizing cerebral vasculature and registering volume rendered images with CT images [S1]. The algorithm addresses the limitations of existing methods like VolumePro and texture mapping, which are restricted to single datasets and do not support extensive shading [S1]. The work builds on the Shear-warp factorization technique and the Linear Level Octree (LLO) data structure to achieve fast 3D volume visualization [S1].

## Sources
[S1] Shear-Warp Volume Rendering Algorithms Using Linear Level Octree for PC-Based Medical Simulation, https://doi.org/10.1007/3-540-45787-9_76, fetched 2026-10-26
