# Silhouette Image Space Acceleration Structure (Mip Pyramid)

Also called: Hierarchical Voxelization Acceleration Structure

## Overview
This method accelerates voxelization queries by providing conservative bounds on silhouette coverage using a mip pyramid over silhouette images. It solves the problem of efficiently determining if a voxel intersects with the visual hull surface without sampling every point within the voxel volume. It is distinguished from sibling methods by operating in image space rather than voxel space, leveraging the hierarchical z-buffer concept to prune voxels based on projected 2D bounds [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ calibrated binary silhouette images $I_i$ (one per camera view), where each image has been smoothed and corrected for lens distortion [S1].
*   **Output:** A mip pyramid for each of the $n$ silhouette images, storing min/max aggregates.

**Data Structures:**
*   **Mip Pyramid:** For each silhouette image $i$, a hierarchy of textures (pyramid levels).
*   **Texel Layout:** Each texel at level $L > 0$ stores two values: the minimum and maximum of the four corresponding texels at level $L-1$. Level 0 contains the raw smoothed silhouette values.
*   **Value Domain:** The values represent silhouette coverage. A value of 1 indicates full coverage, 0 indicates no coverage. The min/max aggregation ensures conservative bounds [S1].

**Construction:**
1.  Generate the base level (Level 0) of the pyramid from the smoothed silhouette images. This is done by launching a compute kernel immediately after silhouette image smoothing [S1].
2.  For each subsequent level $L$, compute the texels by taking the minimum and maximum of the 2x2 block of texels from level $L-1$ that maps to the current texel's position. This is repeated until the pyramid reaches 1x1 resolution [S1].

**Query Algorithm (`sampleSilhouette`):**
To determine a conservative bound on the silhouette value $\mathcal{S}(\mathbf{p})$ for all $\mathbf{p}$ within a voxel $\mathbf{v}$:

1.  **Projection:** Project the 8 corners of voxel $\mathbf{v}$ from Normalized Device Coordinate (NDC) space into the 2D image space of silhouette image $i$.
2.  **Bounding Rectangle:** Compute the axis-aligned bounding rectangle (AABB) of these projected corners. Let the bounds be $minX, minY, maxX, maxY$.
3.  **Level Selection:** Determine the sampling level $L$ in the mip pyramid using the formula:
    $$ L = \lfloor \log_2(\max(maxX - minX, maxY - minY)) \rfloor $$
    At this level $L$, the bounding rectangle covers at most a 2x2 block of texels [S1].
4.  **Sampling:** Sample the texels at the corners of the AABB in the pyramid at level $L$.
5.  **Aggregation:** Return the minimum and maximum of these sampled values as the conservative bound for image $i$.
6.  **Final Bound:** To get the bound for the visual hull $\mathcal{S}(\mathbf{p})$, take the product of the minimums (and similarly maximums) over all $n$ images. This product represents the conservative bound on the silhouette value for the entire voxel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_silhouette(voxel_corners, pyramid_levels, num_images):
    """
    voxel_corners: List of 8 3D points (NDC space) for the voxel.
    pyramid_levels: List of lists, where pyramid_levels[i] is the list of mip levels for image i.
                    Each level is a 2D array of (min, max) tuples.
    num_images: Total number of silhouette images.
    
    Returns:
    global_min: Conservative lower bound on silhouette coverage for the voxel.
    global_max: Conservative upper bound on silhouette coverage for the voxel.
    """
    global_min = 1.0
    global_max = 0.0
    
    for i in range(num_images):
        # 1. Project corners to 2D image space for image i
        # Note: Actual projection matrix and distortion correction are assumed handled
        # by the caller or a helper function not detailed here.
        projected_2d_corners = project_to_image_space(voxel_corners, i)
        
        # 2. Find AABB
        min_x = min(p[0] for p in projected_2d_corners)
        min_y = min(p[1] for p in projected_2d_corners)
        max_x = max(p[0] for p in projected_2d_corners)
        max_y = max(p[1] for p in projected_2d_corners)
        
        # 3. Determine level
        width = max_x - min_x
        height = max_y - min_y
        level = math.floor(math.log2(max(width, height)))
        
        # 4. Sample corners at this level
        # The pyramid at 'level' has texels. We sample the 4 corners of the AABB.
        # Since the AABB covers at most 2x2 texels at this level, we sample the 
        # texels corresponding to the corners of the AABB.
        
        # Helper to get texel value at specific coordinate and level
        def get_texel_min_max(x, y, lvl):
            # Clamp coordinates to texture bounds
            x = clamp(x, 0, width_of_level(lvl) - 1)
            y = clamp(y, 0, height_of_level(lvl) - 1)
            return pyramid_levels[i][lvl][y][x]
            
        # Sample the 4 corners of the AABB
        # Note: The source says "sample these texels at the corners of the axis aligned bounding rectangle".
        # This implies sampling the texels that contain the corners of the AABB.
        
        # Get the texel indices for the corners
        # Assuming integer coordinates for texel centers or top-left corners.
        # The source does not specify interpolation, so we assume point sampling of the 
        # texels at the corners of the AABB.
        
        texel_min_max_1 = get_texel_min_max(min_x, min_y, level)
        texel_min_max_2 = get_texel_min_max(max_x, min_y, level)
        texel_min_max_3 = get_texel_min_max(min_x, max_y, level)
        texel_min_max_4 = get_texel_min_max(max_x, max_y, level)
        
        # 5. Aggregate min/max from these 4 texels
        local_min = min(texel_min_max_1[0], texel_min_max_2[0], texel_min_max_3[0], texel_min_max_4[0])
        local_max = max(texel_min_max_1[1], texel_min_max_2[1], texel_min_max_3[1], texel_min_max_4[1])
        
        # 6. Combine with global bounds
        global_min *= local_min
        global_max *= local_max
        
    return global_min, global_max

def project_to_image_space(voxel_corners, image_index):
    # Placeholder for actual projection logic involving camera intrinsics/extrinsics
    # and lens distortion correction.
    # Returns list of 2D (x, y) coordinates.
    pass

def clamp(val, min_val, max_val):
    return max(min_val, min(val, max_val))

def width_of_level(level):
    # Placeholder for pyramid dimension logic
    pass

def height_of_level(level):
    # Placeholder for pyramid dimension logic
    pass
```

**Parameters:**
*   **Pyramid Depth:** Determined by the resolution of the silhouette images. Not given in the sources.
*   **Smoothing:** The silhouette images are smoothed before pyramid construction. The specific filter is not detailed in the method section, but is part of the acquisition process [S1].

**Edge Cases and Pitfalls:**
*   **Conservative Bounds:** The min/max aggregation ensures that no small features are missed and no holes are created in the voxelization. The bounds are conservatively correct [S1].
*   **Precision:** The use of $\log_2$ and floor operations for level selection ensures that the bounding rectangle covers at most a 2x2 block of texels, preventing oversampling or undersampling artifacts [S1].

## Subtopics

### [Silhouette Image Acquisition and Filtering](3754-silhouette-image-acquisition-and-filtering.md)
Generates the calibrated binary silhouette images used as the base level for the mip pyramid. It integrates lens distortion correction, demosaicing, and Gaussian filtering into a single GPU compute kernel. Choose this when raw sensor data needs to be converted into smooth, geometrically accurate occupancy masks for the acceleration structure.

## Operating characteristics
*   **Time Complexity:** The query time is $O(1)$ per voxel per image, as it involves a constant number of projections and texture lookups (4 samples) regardless of voxel size or pyramid depth. The construction time is linear with respect to the number of texels in the silhouette images [S1].
*   **Memory Complexity:** $O(N \cdot H \cdot W)$, where $N$ is the number of images, and $H \times W$ is the resolution of the base silhouette image. The total memory is approximately $4/3$ of the base image size due to the mip pyramid structure (sum of geometric series $1 + 1/4 + 1/16 + \dots$) [S1].
*   **Measured Performance:** The structure greatly reduces the amount of computation and data accesses needed by the oracle kernel to make a culling decision for a given voxel [S1].
*   **Strengths:** Avoids missing small features or introducing holes in voxelization due to conservative bounds. Efficient for culling large empty regions.
*   **Limitations:** Requires pre-computation of silhouette images and pyramids. Performance may degrade if the voxel projects to very small screen space regions requiring lower pyramid levels with higher memory bandwidth.
*   **When to Use:** When voxelizing from multiple calibrated camera views and efficient culling of voxels outside the visual hull is required.
*   **When Not to Use:** When real-time updates to the silhouette images are required, as the pyramid must be rebuilt.

## Background
The method was introduced in the context of real-time high-resolution sparse voxelization for image-based modeling. It is inspired by the hierarchical z-buffer technique [Greene et al. 1993]. The specific implementation is described in the paper "Real-time high-resolution sparse voxelization with application to image-based modeling" [S1].

## Sources
[S1] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
