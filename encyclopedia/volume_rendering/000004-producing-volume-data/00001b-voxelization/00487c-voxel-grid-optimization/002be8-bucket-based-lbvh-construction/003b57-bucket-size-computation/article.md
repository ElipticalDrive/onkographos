# Bucket Size Computation

Also called: Bucket Texture Construction via Auxiliary Texture, Two-step bucket boundary detection

## Overview
Bucket Size Computation computes the **Bucket Texture**, which stores the position (index) of the last voxel of each bucket (bucket boundary voxel) [S1, S2, unofficial]. It solves the problem of efficiently identifying these boundaries on the GPU without incurring the high cost of texture fetches in vertex shaders, which are required to compare a voxel's bucket with the next voxel's bucket [S1, S2, unofficial]. It sets itself apart from direct approaches by splitting the computation into a fragment shader step (to identify boundaries) and a vertex shader step (to construct the texture), avoiding the inefficiency of vertex shader texture fetches [S1, S2, unofficial].

## How it works
**Inputs:**
*   **Z-Sorted Voxels Texture** (also referred to as Depth-Sorted Voxels Texture): A texture containing all selected voxels, sorted by depth. Each voxel has an index within this texture [S1, S2, unofficial].
*   **Bucket Parameters:** The number of buckets/slabs and the mapping from voxel coordinates to bucket coordinates [S1, S2, unofficial].

**Outputs:**
*   **Bucket Texture:** A small texture where each entry corresponds to a bucket and stores the index of the last voxel belonging to that bucket in the Z-Sorted Voxels Texture [S1, S2, unofficial].

**Data Structures:**
1.  **Z-Sorted Voxels Texture:** Contains all voxels. The algorithm assumes access to the current voxel and the next voxel to determine boundary status [S1, S2, unofficial].
2.  **Auxiliary Texture:** An intermediate texture used in Step 1. Layout per voxel:
    *   **R, G channels:** Store the bucket coordinates (x, y) of the corresponding bucket in the Bucket Texture [S1, S2, unofficial].
    *   **B channel:** Stores a z-value. If the voxel is a bucket boundary, this value is inside the viewing frustum (e.g., 0.5 if $z_{near}=0.0, z_{far}=1.0$). If not a boundary, it is outside the viewing frustum [S1, S2, unofficial].
    *   **A channel:** Stores the index of the voxel in the Z-Sorted Voxels Texture [S1, S2, unofficial].
3.  **Bucket Texture:** The final output. Size is equal to the number of buckets/slabs [S2, unofficial].

**Numbered Steps:**

1.  **Identify Bucket Boundaries (Fragment Shader Step):**
    Render the Z-Sorted Voxels Texture into the Auxiliary Texture using a fragment shader. For each voxel:
    *   Compute the bucket coordinates for the current voxel and the next voxel in the sorted list [S1, S2, unofficial].
    *   Determine if the current voxel is a bucket boundary. A voxel is a boundary if the center of the next voxel falls inside the next slab (bucket) [S1, Fig 5, unofficial].
    *   Write the bucket coordinates to the R and G channels of the Auxiliary Texture [S1, S2, unofficial].
    *   Write the voxel's index in the Z-Sorted Voxels Texture to the A channel of the Auxiliary Texture [S1, S2, unofficial].
    *   Write a z-value to the B channel:
        *   If boundary: Write a value inside the viewing frustum (e.g., 0.5) [S1, S2, unofficial].
        *   If not boundary: Write a value outside the viewing frustum [S1, S2, unofficial].

2.  **Construct Bucket Texture (Vertex Shader Step):**
    Use the Auxiliary Texture as a Vertex Buffer Object (VBO) and render it using `glPoints` of size one [S1, S2, unofficial].
    *   The vertex shader reads the data from the Auxiliary Texture.
    *   The fragment shader (or early-depth test) rejects points where the B channel z-value is outside the viewing frustum [S1, S2, unofficial].
    *   Only points corresponding to bucket boundary voxels (with z-value inside the frustum) pass the depth test and are rendered to the Bucket Texture [S1, S2, unofficial].
    *   The shader writes the voxel index (from the A channel of the Auxiliary Texture) to the appropriate location in the Bucket Texture, determined by the bucket coordinates (R, G channels) [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Fragment Shader for Auxiliary Texture
def compute_auxiliary_texture(z_sorted_voxels_texture, bucket_coords_func):
    auxiliary_texture = new Texture(width, height, channels=4) # R, G, B, A
    
    for i in range(num_voxels):
        current_voxel = z_sorted_voxels_texture[i]
        next_voxel = z_sorted_voxels_texture[i + 1] if i + 1 < num_voxels else None
        
        # Compute bucket coordinates for current voxel
        bucket_x, bucket_y = bucket_coords_func(current_voxel)
        
        # Determine if boundary: check if next voxel's center is in next bucket
        is_boundary = False
        if next_voxel is not None:
            next_bucket_x, next_bucket_y = bucket_coords_func(next_voxel)
            # Boundary if current and next voxels are in different buckets
            # Or specifically, if next voxel's center falls inside the next slab
            if next_bucket_x != bucket_x or next_bucket_y != bucket_y:
                is_boundary = True
        
        # Write to Auxiliary Texture
        auxiliary_texture[i].R = bucket_x
        auxiliary_texture[i].G = bucket_y
        auxiliary_texture[i].A = i # Index in Z-Sorted Voxels Texture
        
        # B channel: z-value for depth test
        if is_boundary:
            auxiliary_texture[i].B = 0.5 # Inside frustum (example)
        else:
            auxiliary_texture[i].B = -1.0 # Outside frustum (example)
            
    return auxiliary_texture

# Step 2: Vertex/Fragment Shader for Bucket Texture
def construct_bucket_texture(auxiliary_texture_vbo, bucket_texture):
    # Render auxiliary_texture_vbo as glPoints of size 1
    # Early depth test rejects points with B channel outside frustum
    
    for point in auxiliary_texture_vbo:
        if point.B is inside_viewing_frustum:
            bucket_x = point.R
            bucket_y = point.G
            voxel_index = point.A
            
            # Write voxel index to Bucket Texture at (bucket_x, bucket_y)
            bucket_texture[bucket_x, bucket_y] = voxel_index
            
    return bucket_texture
```

**Parameters:**
*   **Frustum Z-range:** The values used to determine "inside" and "outside" the frustum for the B channel. Typically, any value within $[z_{near}, z_{far}]$ is considered inside, and any value outside is considered outside [S2, unofficial]. Example: $z_{near}=0.0, z_{far}=1.0$, so 0.5 is inside, -1.0 is outside [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Vertex Shader Texture Fetches:** Determining bucket boundaries requires comparing the current voxel's bucket with the next voxel's bucket. This requires a texture fetch for each voxel, which is inefficient in vertex shaders [S1, S2, unofficial]. This is the primary reason for the two-step approach [S1, S2, unofficial].
*   **Bucket Boundary Definition:** A voxel is a boundary if the center of the next voxel falls inside the next slab (bucket) [S1, Fig 5, unofficial].
*   **Texture Size:** The Bucket Texture size is small (number of buckets/slabs), making transfer to CPU low cost [S2, unofficial].

## Subtopics

### Bucket-based LBVH construction
The parent method that uses the computed bucket sizes to build a Low-Bit-Vector Hash (LBVH) for efficient volume rendering. [Bucket-based LBVH construction](5163-bucket-based-lbvh-construction.md)

## Operating characteristics
*   **Time Complexity:** The method involves two rendering passes. The first pass processes all voxels, and the second pass processes only bucket boundary voxels. The complexity is linear with respect to the number of voxels for the first pass and linear with respect to the number of buckets for the second pass.
*   **Memory Complexity:** Requires an auxiliary texture of the same size as the Z-Sorted Voxels Texture and a small Bucket Texture. The auxiliary texture is temporary.
*   **Performance:** The method is designed to be efficient on the GPU by avoiding expensive vertex shader texture fetches. The use of early-depth test in the second pass efficiently filters out non-boundary voxels.
*   **Strengths:** Efficient GPU implementation, avoids vertex shader texture fetch inefficiencies.
*   **Limitations:** Requires two rendering passes. Relies on specific GPU capabilities (texture fetch in fragment shader, early-depth test).
*   **When to use:** When building a Bucket-based LBVH on the GPU and efficiency is critical.
*   **When not to use:** If CPU-side computation is preferred or if the overhead of two rendering passes is unacceptable.

## Background
The method was introduced in the context of Image-space Sheet-buffered Splatting on the GPU [S1, S2, unofficial]. It addresses the specific challenge of constructing bucket boundaries for volume rendering on the GPU, where vertex shader texture fetches are inefficient [S1, S2, unofficial]. The technique was developed to enable efficient bucket-based LBVH construction, which is crucial for optimizing volume rendering performance [S1, S2, unofficial].

## Sources
[S1] Image-space sheet-buffered splatting on the GPU, https://openalex.org/W2419486389, fetched 2026-10-27 (unofficial)
[S2] PDF Image-space Sheet-buffered Splatting on The Gpu, https://iadisportal.org/ijcsis/papers/2007110210.pdf, fetched 2026-10-29 (unofficial)
