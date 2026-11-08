# Block-based Quadtree Index with GPU Frustum Culling

Also called: Compute shader culling, GPU-side quadtree traversal

## Overview
This method organizes large-scale terrain data into a block-based quadtree index where node metadata (bounding boxes, hierarchy levels, offsets) is packed into a compact structure array residing entirely in GPU video memory [S1]. It solves the CPU traversal and assembly overhead bottleneck by executing frustum culling and occlusion determination in parallel on the GPU via compute shaders, with each thread processing one node to output a list of visible nodes and their Level of Detail (LOD) levels [S1]. It differs from CPU-side hierarchical traversal by offloading the culling logic to the GPU, leveraging parallel processing for cache-friendly Z-order storage [S1].

## How it works
**Inputs:**
*   Global terrain size: $2^L \times 2^L$ samples [S1].
*   Leaf node size: $2^K \times 2^K$ samples [S1].
*   Camera frustum parameters [S1].

**Outputs:**
*   List of visible nodes and their corresponding LOD levels [S1].

**Data Structures:**
1.  **Quadtree Node Index Structure:** A compact structure array in GPU memory. Fields include:
    *   Bounding boxes [S1].
    *   Center coordinates [S1].
    *   Hierarchy levels [S1].
    *   Offsets within the data buffer [S1].
    *   Stored in Z-order curve (space-filling curve) order for cache friendliness [S1].
2.  **Tile Data Buffer:** Elevation data written in Z-order curve order. Within each tile, elevation values are stored contiguously in row-major order [S1].
3.  **Texture Atlas:** Aligned one-to-one with elevation data blocks. Elevation and texture for the same tile have a deterministic mapping relationship in buffer offsets, avoiding secondary lookup tables [S1].

**Formulas:**
*   Quadtree depth: $D = L - K$ [S1].
*   Total number of nodes: $\sum_{i=0}^{D} 4^i = \frac{4^{D+1}-1}{3}$ [S1].

**Parameters:**
*   $K$ (Leaf node size exponent): Determines tile resolution. Typical value $K=9$ (512×512 samples per tile) [S1].
    *   Example: For global SRTM30m data ($2^L \times 2^L$ large), $K=9$ yields $\approx 6.3 \times 10^5$ leaf nodes and $\approx 8.4 \times 10^5$ total nodes, fitting in GPU memory [S1].
    *   Example: For a $2^\circ \times 2^\circ$ Rocky Mountains region at 1-arc-second resolution ($\approx 7200 \times 7200$ samples), $K=9$ yields $\approx 225$ leaf nodes and $\approx 341$ total nodes [S1].

**Steps:**
1.  **Data Organization:** Divide the terrain region into fixed-size tile blocks of $N \times N$ elevation samples ($N=2^K$) [S1].
2.  **Index Construction:** Pack bounding boxes, center coordinates, hierarchy levels, and buffer offsets for all quadtree nodes into a compact structure array [S1].
3.  **Storage Layout:** Write tile data into the buffer in Z-order curve order to ensure spatially adjacent tiles remain adjacent in storage addresses [S1]. Store elevation values within tiles in row-major order [S1].
4.  **Texture Mapping:** Align texture atlases one-to-one with elevation data blocks, ensuring deterministic buffer offset mapping between elevation and texture for each tile [S1].
5.  **Parallel Culling:** Execute frustum culling and occlusion determination in parallel on the GPU via compute shaders [S1].
6.  **Thread Processing:** Each compute shader thread processes one node from the index structure [S1].
7.  **Output Generation:** Threads output a list of visible nodes and their calculated LOD levels [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Entry Point
# Each thread handles one node index 'node_idx' from the packed index array

struct NodeIndex {
    float3 bbox_min;
    float3 bbox_max;
    float3 center;
    int level;
    int data_offset;
};

// Global buffers
NodeIndex* node_indices; // Packed in Z-order
float* elevation_data;   // Z-order, row-major within tiles
float* texture_data;     // Aligned with elevation_data

void main() {
    int node_idx = gl_GlobalInvocationID.x;
    
    // 1. Load node metadata
    NodeIndex node = node_indices[node_idx];
    
    // 2. Frustum Culling
    // Check if node's bounding box intersects the camera frustum
    if (!is_visible_in_frustum(node.bbox_min, node.bbox_max)) {
        return; // Node is culled
    }
    
    // 3. LOD Calculation
    // Determine appropriate LOD level based on distance or screen-space error
    // Note: Specific LOD formula not provided in sources [S1]
    int lod_level = compute_lod(node.center, node.level); 
    
    // 4. Output Visible Node
    // Append to output list of visible nodes
    append_visible_node(node_idx, lod_level);
}

// Helper: Check intersection with frustum planes
bool is_visible_in_frustum(float3 min, float3 max) {
    // Standard AABB vs Frustum plane test
    // Returns true if any part of the box is inside the frustum
    // Implementation details assumed standard graphics primitive
    return true; // Placeholder for actual frustum test logic [G]
}

// Helper: Compute LOD
int compute_lod(float3 center, int max_level) {
    // Distance-based or error-based LOD selection
    // Specific algorithm not detailed in sources [S1]
    return max_level; // Placeholder [G]
}
```

**Edge Cases and Pitfalls:**
*   **Memory Capacity:** For global data scales, the total number of nodes must fit in GPU memory. With $K=9$, global SRTM30m data fits ($\approx 8.4 \times 10^5$ nodes) [S1].
*   **Data Scale Distinction:** Global data scale differs from experimental region scale. A $2^\circ \times 2^\circ$ region has far fewer nodes ($\approx 341$) than global data [S1].
*   **Storage Order:** Deviation from Z-order storage may reduce cache efficiency [S1].

## Subtopics

### Dynamic LOD Generation
Uses compute shaders to calculate subdivision levels based on viewpoint distance and screen-space error, replacing CPU-based calculations [S1].

### WebGPU Integration
Tailored for WebGPU environments, addressing raw terrain parsing, GPU-friendly storage, and copy-free rendering pipelines [S1].

## Operating characteristics
*   **Performance:** Parallel processing on the GPU reduces traversal and assembly overhead at the CPU when data scales grow large [S1].
*   **Memory:** Index data for global SRTM30m data ($\approx 8.4 \times 10^5$ nodes) fully resides in GPU memory with $K=9$ [S1].
*   **Cache Efficiency:** Z-order curve storage provides good cache friendliness for large-area terrain data [S1].
*   **Limitations:** Relies on preprocessed formats or efficient parsing; raw terrain parsing and organization can consume significant loading time, especially over networks [S1].

## Background
This method was proposed in the context of WebGPU terrain visualization, addressing issues with raw terrain parsing, GPU-friendly storage organization, and compute-shader-based dynamic LOD [S1]. It builds on previous works exploring WebGPU for terrain, such as occlusion culling and multi-threaded scheduling [S1]. The approach is tailored for large-scale data, such as global SRTM30m or high-resolution regional data like the Rocky Mountains [S1].

## Sources
[S1] Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU, https://doi.org/10.1145/3844212.3844228, fetched 2026-11-02
