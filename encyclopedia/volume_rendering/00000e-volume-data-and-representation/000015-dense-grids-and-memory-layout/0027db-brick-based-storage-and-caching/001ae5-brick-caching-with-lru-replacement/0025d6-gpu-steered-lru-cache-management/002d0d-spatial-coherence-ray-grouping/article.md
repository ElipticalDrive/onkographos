# Spatial Coherence Ray Grouping

Also called: Package Ray Grouping, Pattern Attribution of Node Sets

## Overview
Spatial Coherence Ray Grouping is a technique for optimizing volume rendering by exploiting the spatial coherence of ray traversal paths. It groups neighboring rays into packages (e.g., 2x2 blocks) to collect node traversal information, thereby reducing the volume of data sent to the CPU for cache management [S1, unofficial]. Unlike standard per-ray processing, this method unifies frustum containment, visibility, and Level-of-Detail (LOD) selection within the rendering kernel, offloading workload from the CPU [S1, unofficial]. It is a specific implementation of the broader Ray Aggregation strategy, which traces groups of rays as single units to leverage register SIMD capabilities [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of rays defined by their origins and directions, grouped into packages based on image plane coordinates.
*   **Output:** Rendered color for each ray, plus supplementary traversal data (node indices) required for cache updates.

**Data Structures:**
The core data structure is the **Ray Package**. For a standard 2x2 package:
*   Contains 4 rays.
*   Each ray outputs a fixed-size list of traversed node indices.
*   Typical size: 12 node indices per ray [S1, unofficial].
*   Total capacity per package: $4 \times 12 = 48$ node indices [S1, unofficial].

**Algorithm Steps:**
1.  **Grouping:** Divide the image plane into packages of size $P \times P$ (commonly $2 \times 2$) [S1, unofficial].
2.  **Traversal:** Trace all rays within a package. Due to spatial coherence, neighboring rays traverse similar nodes in the underlying data structure (e.g., a brick-based grid) [S1, unofficial].
3.  **Attribution:** Assign specific slots in the package's output buffer to specific rays based on their position within the package. For a 2x2 package:
    *   Upper-left ray stores the first 12 node indices.
    *   Upper-right ray stores the second 12 node indices.
    *   Lower-left ray stores the third 12 node indices.
    *   Lower-right ray stores the fourth 12 node indices [S1, unofficial].
4.  **Supplementary Output:** Alongside the final color, each ray outputs its assigned node indices. This data serves as the "helpful node" indicator for the parent GPU-Steered LRU Cache Management system [S1, unofficial].
5.  **Aggregation:** The package outputs the combined traversal data (e.g., 48 indices) to be processed by the CPU for cache updates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_ray_package(rays, package_size=2):
    """
    Processes a package of rays, exploiting spatial coherence.
    rays: List of Ray objects in the package.
    package_size: Integer, e.g., 2 for a 2x2 block.
    Returns: Tuple of (colors, node_indices_list)
    """
    num_rays = len(rays)
    nodes_per_ray = 12  # Typical value from [S1, unofficial]
    total_nodes = num_rays * nodes_per_ray
    
    colors = []
    all_node_indices = []
    
    # Step 1 & 2: Traverse rays and collect nodes
    for i, ray in enumerate(rays):
        # Traverse the volume structure for this ray
        # Returns a list of visited node indices
        visited_nodes = traverse_volume(ray)
        
        # Step 3: Attribute nodes to the package buffer
        # In a real implementation, these would be written to a shared
        # package buffer at specific offsets based on ray position.
        # Here we collect them for the return value.
        all_node_indices.extend(visited_nodes[:nodes_per_ray])
        
        # Compute color
        color = compute_color(ray, visited_nodes)
        colors.append(color)
        
    return colors, all_node_indices
```

**Edge Cases and Pitfalls:**
*   **Transparency:** In transparent volumes, rays may traverse a large number of nodes. The fixed output size (e.g., 48 indices) may be insufficient to capture all relevant nodes for cache coherence [S1, unofficial]. Ad-hoc solutions or multi-render-target (MRT) extensions may be required if the limit is exceeded [S1, unofficial].
*   **Precision:** Node indices must be packed efficiently to fit within the framebuffer or intermediate buffer limits [S1, unofficial].

## Subtopics

### Ray Aggregation (Packet Traversal)
A general strategy where a group of rays is traced as a single unit (a packet) to exploit spatial coherence [S2]. It is the most used strategy in current interactive systems [S2]. It allows the use of register SIMD to process four rays in a bundle and can be combined with frustum- and interval-arithmetic techniques to avoid traversal steps by exploiting conservative bounds of the ray group [S2]. It is used with k-dtrees and for primary eye rays, hard/soft shadow rays, and secondary rays [S2]. Spatial Coherence Ray Grouping is a specific application of this strategy tailored for volume data streaming.

## Operating characteristics
*   **Time Complexity:** Reduces CPU overhead by batching traversal data. The GPU handles the traversal and data collection in parallel [S1, unofficial].
*   **Memory Complexity:** Requires additional buffer space for supplementary node indices per ray. For a 2x2 package with 12 nodes/ray, this is 48 indices per package [S1, unofficial].
*   **Strengths:** Unifies visibility, frustum culling, and LOD selection in the kernel [S1, unofficial]. Significantly reduces data transfer between GPU and CPU for cache management [S1, unofficial].
*   **Limitations:** Fixed buffer sizes may truncate traversal history in highly transparent scenes [S1, unofficial].
*   **When to use:** When using GPU-Steered LRU Cache Management for brick-based volume rendering [S1, unofficial].

## Background
Originated in the context of "Ray-Guided Streaming" for efficient voxel rendering [S1, unofficial]. The key insight is that neighboring rays visit mostly similar nodes because a brick in a node projects onto several pixels [S1, unofficial]. This method was proposed to combine the rendering and visibility update phases, taking workload off the CPU [S1, unofficial]. It builds on the general concept of Ray Aggregation, which has been used in interactive systems with k-dtrees [S2].

## Sources
[S1] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
