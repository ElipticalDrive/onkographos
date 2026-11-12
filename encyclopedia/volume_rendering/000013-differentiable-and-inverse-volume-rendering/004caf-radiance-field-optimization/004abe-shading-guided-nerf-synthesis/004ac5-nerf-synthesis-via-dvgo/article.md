# NeRF Synthesis via DVGO

Also called: 2D texture synthesis for 3D NeRF

## Overview
NeRF Synthesis via DVGO is a method for generating large-scale radiance fields by applying 2D texture synthesis algorithms to the flattened density and feature grids of a Differentiable Voxel Grid Object (DVGO) representation [S1]. It solves the problem of efficiently synthesizing repetitive 3D structures (e.g., vegetation) by exploiting the observation that only two spatial dimensions typically exhibit clear repeatability, thereby avoiding the inefficiency of full 3D synthesis [S1]. It sets itself apart from direct 3D synthesis by flattening the 3D grid along the Z-axis into 2D grids, allowing the use of established 2D texture synthesis techniques while treating each unit as storing density and feature values rather than RGB pixels [S1].

## How it works
**Inputs:**
*   Source DVGO: Density grid $V^{(d)}$ and feature grid $V^{(f)}$, both of size $W \times H \times D$.
*   Target dimensions: Width $W_{target}$, Height $H_{target}$.
*   Synthesis parameters: Patch size $S \times S$, number of nearest neighbors $k_g$, blending radius.

**Outputs:**
*   Synthesized Density grid $I^{(d)}$ of size $W_{target} \times H_{target}$.
*   Synthesized Feature grid $I^{(f)}$ of size $W_{target} \times H_{target}$.

**Data Structures:**
The 3D grids are flattened into 2D grids along the Z-axis. For a voxel at $(x, y, z)$, the corresponding 2D coordinate $(x', y')$ is calculated as:
$$ x' = x $$
$$ y' = y \cdot D + z $$
The resulting 2D grid $I^{(d)}$ has dimensions $W \times (H \cdot D)$. Each "pixel" in this 2D grid stores a scalar density value $d$ and a feature vector $\mathbf{f}$.

**Algorithm (Two-Phase Method):**
The method prioritizes geometric continuity over appearance to avoid artifacts caused by occluded interior parts [S1].

1.  **Flattening:** Convert $V^{(d)}$ and $V^{(f)}$ into 2D grids $I^{(d)}$ and $I^{(f)}$ by flattening along the Z-axis [S1].
2.  **Initialization:** Initialize the target grids $I^{(d)}_{target}$ and $I^{(f)}_{target}$ with zeros or random values.
3.  **Synthesis Loop:** Iterate through each voxel patch $P_n$ in the target grid (typically in a raster order or random order).
    *   **Phase 1 (Geometry Selection):**
        *   Extract the boundary of the current density patch $P_n^{(d)}$, denoted $\partial(P_n^{(d)})$.
        *   Search the source grid $I^{(d)}$ for candidate patches $E_i^{(d)}$.
        *   Calculate the similarity between the boundary of the target patch and the boundary of each candidate using a distance metric $d(\cdot, \cdot)$ (e.g., L2 distance on flattened boundary vectors) [S1].
        *   Select the $k_g$ best matching density patches $E_n^{(d)} = \{E_1^{(d)}, \dots, E_{k_g}^{(d)}\}$ based on minimum distance [S1].
    *   **Phase 2 (Appearance Selection):**
        *   For each of the $k_g$ selected density patches, retrieve the corresponding feature patch $E_i^{(f)}$ from the source feature grid $I^{(f)}$.
        *   Calculate the similarity between the boundary of the target feature patch $P_n^{(f)}$ and the boundary of each candidate feature patch $\partial(E_i^{(f)})$ using $d(\partial(P_n^{(f)}), \partial(E_i^{(f)}))$ [S1].
        *   Convert these similarities into likelihoods for random selection. Patches with higher similarity (lower distance) have higher probability of being selected [S1].
        *   Randomly select one final patch $E_n^*$ from the $k_g$ candidates based on these likelihoods [S1].
    *   **Blending:**
        *   Copy the non-overlapping parts of $E_n^*$ into $P_n$.
        *   For overlapping regions between $E_n^*$ and the existing content in $P_n$, blend the values using inverse distance weighting [S1].
4.  **Unflattening (Optional):** If the final output requires a 3D grid, reshape $I^{(d)}_{target}$ and $I^{(f)}_{target}$ back to $W \times H \times D_{target}$ by reversing the flattening mapping.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synthesize_dvgo(source_density_3d, source_feature_3d, target_w, target_h, target_d, patch_size, k_g):
    # Flatten 3D to 2D along Z-axis
    # Source: (W, H, D) -> (W, H*D)
    source_density_2d = flatten_z(source_density_3d)
    source_feature_2d = flatten_z(source_feature_3d)
    
    target_w_2d = target_w
    target_h_2d = target_h * target_d
    
    target_density_2d = zeros((target_w_2d, target_h_2d))
    target_feature_2d = zeros((target_w_2d, target_h_2d))
    
    # Iterate over target patches
    for y in range(0, target_h_2d, patch_size):
        for x in range(0, target_w_2d, patch_size):
            # Define current patch coordinates
            patch_coords = get_patch_coords(x, y, patch_size, target_w_2d, target_h_2d)
            
            # Phase 1: Geometry
            # Extract boundary of current target density patch
            current_boundary_d = get_boundary(target_density_2d, patch_coords)
            
            # Find k_g best matches in source
            candidates_d = find_k_nearest_neighbors(
                source_density_2d, current_boundary_d, k_g, patch_size
            )
            
            # Phase 2: Appearance
            # Calculate feature similarity for candidates
            current_boundary_f = get_boundary(target_feature_2d, patch_coords)
            similarities = []
            for cand_d in candidates_d:
                # Get corresponding feature patch from source
                cand_f = get_feature_patch(source_feature_2d, cand_d.coords, patch_size)
                cand_boundary_f = get_boundary(cand_f, cand_d.coords)
                dist = l2_distance(current_boundary_f, cand_boundary_f)
                similarities.append((dist, cand_d))
            
            # Sort by distance (lower is better)
            similarities.sort(key=lambda x: x[0])
            
            # Convert distances to likelihoods (e.g., softmax or inverse)
            # Note: Exact likelihood function not specified, assume inverse distance weighting
            likelihoods = [1.0 / (dist + 1e-6) for dist, _ in similarities]
            total = sum(likelihoods)
            likelihoods = [l / total for l in likelihoods]
            
            # Random selection based on likelihoods
            selected_candidate = weighted_random_choice(similarities, likelihoods)
            
            # Blend selected patch into target
            blend_patch(target_density_2d, target_feature_2d, selected_candidate, patch_coords)
            
    # Unflatten back to 3D
    target_density_3d = unflatten_z(target_density_2d, target_w, target_h, target_d)
    target_feature_3d = unflatten_z(target_feature_2d, target_w, target_h, target_d)
    
    return target_density_3d, target_feature_3d

def flatten_z(grid_3d):
    W, H, D = grid_3d.shape
    grid_2d = np.zeros((W, H * D))
    for x in range(W):
        for y in range(H):
            for z in range(D):
                idx = y * D + z
                grid_2d[x, idx] = grid_3d[x, y, z]
    return grid_2d

def unflatten_z(grid_2d, W, H, D):
    grid_3d = np.zeros((W, H, D))
    for x in range(W):
        for y in range(H):
            for z in range(D):
                idx = y * D + z
                grid_3d[x, y, z] = grid_2d[x, idx]
    return grid_3d

def get_boundary(grid_2d, coords, patch_size):
    # Extract boundary pixels of the patch defined by coords
    # Returns flattened array of boundary values
    pass

def find_k_nearest_neighbors(source_grid, target_boundary, k, patch_size):
    # Search source grid for patches with minimum boundary distance
    # Returns list of best k patches
    pass

def blend_patch(target_d, target_f, source_patch, coords):
    # Blend source_patch into target_d and target_f at coords
    # Using inverse distance weighting for overlaps
    pass
```

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** The two-phase method struggles with boundary synthesis because occluded interior parts are synthesized onto the boundary, lacking pixel value supervision during DVGO training, leading to artifacts [S1].
*   **Dimensionality:** High dimensionality of the combined grid increases search time [S1].
*   **Weighting:** In the baseline combined method, high weight for density causes color inconsistencies, while high weight for color causes geometric inconsistencies [S1].

## Subtopics

### Two-Phase NeRF Synthesis
A baseline method that concatenates density and feature grids $V^{(d)}$ and $V^{(f)}$ into a combined 2D image $I^{(d \oplus f)}$ and measures distance via L2 [S1]. It suffers from weighting challenges between geometry and appearance [S1]. It prioritizes density over color because the synthesis operates at the voxel column level where color features inherently incorporate geometric information; ensuring geometric continuity first prevents density from affecting subsequent appearance considerations [S1]. The process involves Phase 1 (Geometry): selecting $k_g$ best density voxel patches $E_n^{(d)}$ based on similarity of $d(\partial(P_n^{(d)}), \partial(E_n^{(d)}))$; and Phase 2 (Appearance): calculating similarity $d(\partial(P_n^{(f)}), \partial(E_n^{(f)}))$ for the selected patches, using these as likelihoods for random selection, and blending overlapping regions using inverse distance weighting [S1]. Pitfalls include high weight for density causing color inconsistencies, high weight for color causing geometric inconsistencies, and high dimensionality of $I^{(d \oplus f)}$ increasing search time [S1].

## Operating characteristics
*   **Time Complexity:** The search for nearest neighbors in the source grid dominates the complexity. With high dimensionality of the feature space, search time increases significantly [S1].
*   **Memory Complexity:** Requires storing the flattened 2D grids $I^{(d)}$ and $I^{(f)}$, which have size $W \times (H \cdot D)$.
*   **Strengths:** Efficiently synthesizes repetitive 3D structures by leveraging 2D texture synthesis methods; avoids the inefficiency of full 3D synthesis [S1].
*   **Limitations:** Boundary artifacts due to lack of supervision for occluded parts; sensitivity to weighting in combined methods; high search time for high-dimensional features [S1].
*   **When to use:** Suitable for scenes with clear 2D repeatability (e.g., grass, trees) where 3D synthesis is inefficient [S1].
*   **When not to use:** Scenes with complex 3D structures lacking 2D repeatability or requiring precise boundary continuity without artifacts [S1].

## Background
The method originates from the paper "NeRF Synthesis with Shading Guidance" [S1]. It builds on the DVGO representation, which uses a hybrid implicit-explicit structure [S1]. The approach was developed to address the inefficiency of 3D synthesis for objects like grass, where only two dimensions exhibit clear repeatability [S1].

## Sources
[S1] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
