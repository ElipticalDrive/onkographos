# Z-axis RLE Voxel Raycasting

Also called: Column-based RLE traversal, RLE-Encoded Voxel Data Traversal, RLE-encoded Z-axis voxel raycasting, Z-column RLE iteration, Z-column RLE raytracing

## Overview
Z-axis RLE Voxel Raycasting is a traversal method for voxel volumes that compresses voxel attributes (such as normals and colors) per X,Y column using run-length encoding (RLE) along the Z-axis to reduce memory bandwidth and accelerate ray intersection [S1, unofficial]. It builds on [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) by separating the structural voxel grid from its metadata, storing the latter in a compressed format that allows linear iteration after the initial geometric hit is found [S1, unofficial]. Unlike raw voxel textures which allow fast runtime editing but lack compression, this method optimizes rendering speed at the cost of dynamic volume modification difficulty [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A ray defined by origin and direction; a pre-computed voxel grid (for spatial traversal); an RLE index texture; an RLE data texture.
*   **Output:** Voxel attributes (color, normal) for each hit along the ray.

**Data Structures**
1.  **Voxel Grid:** The standard sparse or dense voxel structure used for initial intersection [S1, unofficial].
2.  **RLE Index Texture:** A 2D texture mapping 2D voxel coordinates $(X, Y)$ to a pointer (texture coordinate) [S1, unofficial]. This pointer indicates the start of the RLE element list in the RLE data texture for that specific column [S1, unofficial].
3.  **RLE Data Texture:** A 1D or 2D texture storing RLE elements sequentially per column, encoded along the Z-axis [S1, unofficial]. Each element contains voxel metadata (colors, normals, etc.) [S1, unofficial].

**Algorithm Steps**
1.  **Initial Hit:** Perform standard voxel grid traversal to find the first voxel hit for the ray [S1, unofficial].
2.  **Index Lookup:** Use the X,Y coordinates of the hit voxel to index into the RLE index texture [S1, unofficial]. Retrieve the pointer (texture coordinate) to the start of the RLE element list for that column [S1, unofficial].
3.  **Linear Iteration:** Iterate sequentially through the RLE elements starting from the retrieved pointer [S1, unofficial].
4.  **Z-Comparison:** Compare the Z value of each RLE element against the ray's current Z position [S1, unofficial].
5.  **Hit Resolution:** Stop when the Z value matches or exceeds the ray's Z to identify the hit voxel and retrieve its attributes [S1, unofficial].
6.  **Repeat:** Continue the ray traversal loop, repeating steps 2-5 for subsequent hits [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cast_ray_with_z_rle(ray_origin, ray_direction, voxel_grid, rle_index_texture, rle_data_texture):
    # Step 1: Perform voxel grid traversal to find initial hit
    hit = voxel_grid.traverse(ray_origin, ray_direction)
    
    if hit is None:
        return []

    results = []
    
    while hit is not None:
        # Step 2: Retrieve RLE-encoded data for the corresponding z-column
        # Use X,Y coordinates to get pointer to start of RLE list
        rle_start_ptr = rle_index_texture.sample(hit.x, hit.y)
        
        # Step 3: Linearly iterate through RLE entries
        current_ptr = rle_start_ptr
        current_z = hit.z
        
        # Step 4 & 5: Compare Z value and resolve attributes
        # Iterate until we find the element matching the current hit's Z
        # Note: The source implies iterating through the list to find the specific Z
        # associated with the hit found by the grid traversal.
        
        while True:
            element = rle_data_texture.read(current_ptr)
            
            # Check if this RLE element corresponds to the current hit's Z
            # Assuming RLE elements store Z-coordinate and attributes
            if element.z == current_z:
                # Hit found in RLE data
                results.append({
                    'position': hit.position,
                    'normal': element.normal,
                    'color': element.color
                })
                break
            
            # Move to next element in the column
            current_ptr += element.stride
            current_ptr = current_ptr % rle_data_texture.size # Handle wrap-around if 1D texture
            
            # Safety break: if we looped through the whole column without match
            # This handles cases where the grid hit might be slightly off or 
            # the RLE list is exhausted.
            if current_ptr == rle_start_ptr:
                break

        # Continue ray traversal for next hit
        hit = voxel_grid.traverse_next(hit.position, ray_direction)

    return results
```

**Parameters**
*   **Base Scene Resolution:** 1024x1024x256 voxels [S1, unofficial].
*   **Voxel Data Size:** 16MB (static, does not change with scene complexity) [S1, unofficial].
*   **RLE Data Size:** 6.4MB (varies based on color and normal variety) [S1, unofficial].
*   **RLE Indexing Data Size:** 4MB [S1, unofficial].
*   **Total Memory Footprint:** 26.4MB [S1, unofficial].
*   **Maximum Iterations per Ray:** $Z$ (height of the voxel dataset) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Transformations:** Applying geometric transformations to RLE-encoded data is expensive because it requires shifting large blocks of data (hurting performance) or pre-padding the data (hurting memory usage) [S1, unofficial].
*   **Dynamic Updates:** RLE encoding makes dynamic volume modification difficult compared to raw voxel textures which support fast runtime editing via render-to-texture [S1, unofficial].
*   **Iteration Cost:** The maximum iteration of $Z$ per ray is considered to have a negligible performance impact compared to the voxel grid traversal algorithm itself [S1, unofficial].

## Operating characteristics
*   **Complexity:** The method is bandwidth-bound; performance is limited by memory bus throughput rather than arithmetic compute [S1, unofficial].
*   **Scaling:** Ray count increases (e.g., from 1000 to 4000 rays) do not increase render time if memory access patterns remain constant [S1, unofficial].
*   **Strengths:** Reduces memory bandwidth requirements by compressing attribute data per column; suitable for real-time rendering where memory bus is the bottleneck [S1, unofficial].
*   **Limitations:** Poor support for dynamic scene modifications; expensive to transform due to data shifting requirements [S1, unofficial].
*   **When to Use:** Static or semi-static voxel volumes where rendering performance is critical and memory bandwidth is the primary bottleneck [S1, unofficial].
*   **When Not to Use:** Scenes requiring frequent dynamic updates or geometric transformations of the voxel attributes [S1, unofficial].

## Background
This method was proposed by IJs (Ken Silverman forum, March 2008) as the core traversal method for a real-time voxel raytracing engine [S1, unofficial]. It was developed in the context of exploring GPU-based voxel raytracing, aiming to overcome memory bandwidth limitations inherent in high-resolution voxel datasets [S1, unofficial]. The approach was part of an experimental engine design intended to leverage parallel processing power for volumetric effects [S1, unofficial].

## Sources
[S1] Forum archive / Ken Silverman / Real-time voxel raytracing, in development.., https://www.jonof.id.au/forum/topic-1433.html, fetched 2026-10-26 (unofficial)
