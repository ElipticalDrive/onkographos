# Multiresolution LAO Implementation

Also called: Multiresolution Local Ambient Occlusion, Adaptive LAO Sampling

## Overview
This method implements Local Ambient Occlusion (LAO) within a multiresolution volume rendering framework to optimize performance and memory usage. It solves the problem of high computational cost in volumetric occlusion estimation by adapting the sampling density and data resolution to the local complexity of the transfer function (TF) domain. Unlike fixed-resolution approaches, it prioritizes blocks with high TF variation for higher detail and lower sampling density for coarser blocks, reducing the number of ray integrals required without significantly degrading image quality [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A regular linear volume dataset.
*   A Transfer Function (TF) mapping scalar values to color and opacity.
*   Rendering parameters: target memory limit, minimum/maximum LOD constraints, number of ray directions $K$.

**Outputs:**
*   A final ambient light intensity volume texture used in the rendering pass.
*   A multiresolution packed volume texture containing the selected LODs.

### Data Structures

**1. Packed 3D Texture**
*   **Layout:** Contains volume data for all selected LODs ($16^3, 8^3, 4^3, 2^3, 1^3$) organized into blocks.
*   **Boundary Handling:** No boundary voxel replication. Uses interblock interpolation when required [S1, unofficial].
*   **Interpolation Note:** Interblock interpolation is **not** used during LAO computations to avoid excessive texture lookups. Minor artifacts may result from this limitation [S1, unofficial].

**2. Forward Mapping Index Texture ($V \rightarrow P$)**
*   **Purpose:** Maps volume coordinates $V$ to packed texture coordinates $P$.
*   **Size:** Same size as the number of blocks in 3D.
*   **Content:** For each block, stores the 3D offset and the size (LOD) of the block in the packed texture.

**3. Reverse Mapping Index Texture ($P \rightarrow V$)**
*   **Purpose:** Maps packed texture coordinates $P$ to volume coordinates $V$.
*   **Size:** Restricted to be 64 times smaller than the packed volume.
*   **Constraint:** Achieved by limiting the minimum LOD for nonempty blocks to $4^3$ voxels. If a voxel covers an entire block, the reverse map would otherwise need to be as large as the packed volume [S1, unofficial].

**4. Ambient Light Texture**
*   **Purpose:** Holds the accumulated emissive color and ambient light for each voxel.
*   **Layout:** Same dimensions and multiresolution layout as the packed volume texture.
*   **Initialization:** $A^0 = 0$ [S1, unofficial].

### Algorithm Steps

1.  **Preprocessing (Block Division & LOD Selection):**
    *   Divide the regular linear volume into blocks of typically $16^3$ voxels.
    *   Encode each block with all possible LODs ($8^3, 4^3, 2^3, 1^3$).
    *   Select the resolution for each block based on TF domain variation:
        *   **High variation:** Prioritize high LOD.
        *   **Medium/Low variation:** Use lower LOD.
        *   **Empty blocks:** Skip completely.
    *   Update this selection when the TF changes or the volume is cropped.

2.  **Packing:**
    *   Pack the selected LOD data into the single Packed 3D Texture.
    *   Populate the Forward Mapping Index Texture with offsets and sizes.
    *   Populate the Reverse Mapping Index Texture (limited to min LOD $4^3$) [S1, unofficial].

3.  **Incremental LAO Pipeline:**
    *   Create the Ambient Light Texture (initialized to zero).
    *   Map volume slices one by one as rendering targets of a framebuffer object.
    *   For each pass, render a quad over the entire framebuffer where each pixel maps to one voxel.
    *   **Ray Direction Selection:** Choose one or more ray directions per pass. Typically, one direction per frame is used for interactivity. Directions are distributed uniformly (e.g., using a cube, octahedron, or icosahedron) [S1, unofficial].
    *   **Coordinate Mapping:**
        *   Use the fragment's framebuffer location and current slice $z$-offset.
        *   Apply Reverse Mapping ($P \rightarrow V$) to determine the voxel location $V$ in volume coordinates.
    *   **Ray Integral Computation:**
        *   Compute the LAO integral along the chosen ray direction.
        *   **Adaptive Sampling:** Adjust sampling density along the ray based on the block's resolution. Lower resolution blocks use larger step lengths to optimize the number of steps. Fixed sampling density relative to the LOD is avoided as it may miss important structures [S1, unofficial].
        *   For each sample point, use Forward Mapping ($V \rightarrow P$) to perform texture indirection into the Packed 3D Texture.
        *   Do **not** use interblock interpolation during this step [S1, unofficial].
    *   **Accumulation:**
        *   Update the Ambient Light Texture using the incremental blend operation. Let $A_k$ be the intensity contribution from the current ray direction $k$, and $A^{k-1}$ be the accumulated intensity from previous directions. The new accumulated intensity $A^k$ is:
            $$A^k = \frac{1}{k} \cdot A_k + \left(1 - \frac{1}{k}\right) \cdot A^{k-1}$$
        *   This is implemented using the OpenGL blend operation [S1, unofficial].

4.  **Completion:**
    *   Repeat the pipeline for all $K$ ray directions.
    *   The final ambient light intensity volume is used in the subsequent rendering pass.
    *   Interblock interpolation can be applied after interaction stops if discontinuity artifacts are visible [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_lao(volume_data, transfer_function, num_ray_directions):
    """
    Implements Multiresolution LAO.
    volume_data: 3D array of scalar values.
    transfer_function: Function mapping scalar -> (color, opacity).
    num_ray_directions: Total number of K ray directions to sample.
    """
    
    # 1. Preprocessing
    block_size = 16
    blocks = divide_into_blocks(volume_data, block_size)
    
    # LOD Selection based on TF variation
    selected_lods = {}
    for block in blocks:
        tf_variation = compute_tf_variation(block, transfer_function)
        if is_empty(block):
            selected_lods[block.id] = None # Skip empty
        elif tf_variation > high_threshold:
            selected_lods[block.id] = 16 # High LOD
        elif tf_variation > low_threshold:
            selected_lods[block.id] = 8  # Medium LOD
        else:
            selected_lods[block.id] = 4  # Low LOD (min for non-empty)
            
    # 2. Packing and Indexing
    packed_texture = create_packed_3d_texture()
    forward_map = create_forward_mapping_index_texture(blocks, selected_lods)
    reverse_map = create_reverse_mapping_index_texture(selected_lods, min_lod=4)
    
    # 3. Initialize Ambient Light Texture
    ambient_texture = create_3d_texture_same_layout_as(packed_texture)
    ambient_texture.fill(0.0)
    
    # 4. Ray Direction Generation
    ray_directions = generate_uniform_directions(num_ray_directions)
    
    # 5. Incremental Rendering Loop
    for k, direction in enumerate(ray_directions, start=1):
        # Render one slice at a time
        for z_slice in range(volume_depth):
            # Render quad covering framebuffer
            # Each pixel corresponds to a voxel in the current slice
            
            # Get fragment coordinates (pixel -> packed texture coord P)
            P = get_fragment_packed_coords()
            
            # Reverse map P -> V (volume coordinates)
            V = reverse_map.lookup(P)
            
            # Compute LAO integral for this ray direction
            # Adaptive sampling based on block resolution at V
            ldo = selected_lods.get_block_at(V)
            step_size = adaptive_step_size(ldo)
            
            lao_contribution = 0.0
            current_pos = V
            while current_pos is within volume bounds:
                # Forward map V_sample -> P_sample
                P_sample = forward_map.lookup(current_pos)
                
                # Sample packed texture (no interblock interpolation)
                sample_value = packed_texture.sample(P_sample)
                
                # Compute occlusion weight based on sample value and distance
                weight = compute_lao_weight(sample_value, current_pos, V)
                lao_contribution += weight
                
                current_pos += direction * step_size
            
            # Accumulate into ambient texture
            # A^k = (1/k) * A_k + (1 - 1/k) * A^{k-1}
            # A_k is the result of the current ray direction integration
            # Note: In practice, A_k might be normalized or scaled differently
            # depending on the specific LAO definition, but the blend formula
            # is as stated in the source.
            
            current_ambient = ambient_texture.sample(P)
            new_ambient = (1.0 / k) * lao_contribution + (1.0 - 1.0 / k) * current_ambient
            ambient_texture.write(P, new_ambient)
            
    return ambient_texture
```

### Parameters and Edge Cases
*   **Typical Block Size:** $16^3$ voxels.
*   **LODs:** $16^3, 8^3, 4^3, 2^3, 1^3$.
*   **Minimum LOD for Non-empty Blocks:** $4^3$ (to allow 64x smaller reverse map).
*   **Ray Directions:** Uniformly distributed (cube, octahedron, icosahedron).
*   **Edge Case - Empty Blocks:** Skipped entirely in LOD selection and packing.
*   **Edge Case - Interpolation:** Interblock interpolation is disabled during LAO computation to save texture lookups, potentially causing minor artifacts at block boundaries [S1, unofficial].
*   **Pitfall - Fixed Sampling:** Using a sampling density fixed to the LOD can miss important structures. Adaptive step lengths are required [S1, unofficial].

## Operating characteristics

**Time Complexity:**
*   Depends on the number of ray directions $K$ and the number of non-empty blocks.
*   Adaptive sampling reduces the number of steps for lower-resolution blocks, improving performance.
*   Incremental update allows for interactive feedback, as results improve over frames.

**Memory Complexity:**
*   Dominated by the Packed 3D Texture and the two index textures.
*   Memory usage is optimized by skipping empty blocks and using lower LODs for low-variation blocks.
*   The reverse map is limited to 1/64th the size of the packed volume due to the $4^3$ minimum LOD constraint.

**Strengths:**
*   Significant performance gain due to data reduction from LOD selection.
*   Adaptive sampling optimizes the trade-off between accuracy and speed.
*   Incremental update supports user interaction.

**Limitations:**
*   Discontinuities may appear at block boundaries if interblock interpolation is not used in LAO computation.
*   Requires preprocessing to build the multiresolution structure.
*   LOD selection depends on the TF, requiring updates if the TF changes.

## Background

This method is described in the paper "Local Ambient Occlusion in Direct Volume Rendering" [S1, unofficial]. It builds upon the Local Ambient Occlusion (LAO) technique and extends it with a multiresolution framework inspired by Kraus and Ertl [35] and Ljung et al. [33]. The approach aims to balance image quality and performance by adapting the resolution and sampling density to the visual importance of volume regions.

## Sources

[S1] Local Ambient Occlusion in Direct Volume Rendering, https://www.diva-portal.org/smash/get/diva2:321233/FULLTEXT01.pdf, fetched 2026-10-27 (unofficial)
