# Topology Simplification via Volumetric Modification

Also called: Genus Reduction, Handle Removal, Handle collapsing, Handle pinching, Loop Closure for Handle Removal, Topological simplification of isosurfaces, Topology Simplification via Handle Removal

## Overview
This method removes extraneous topological handles from isosurfaces extracted from volumetric data (e.g., CT, MRI) by modifying the scalar volume values rather than performing mesh surgery, ensuring the resulting isosurface remains a manifold [S1]. It addresses the problem of identifying and removing insignificant topological features based on their persistence, specifically targeting handles with small geometric sizes [S1]. It builds on [Isosurface Simplification via Persistence](8470-isosurface-simplification-via-persistence.md) by providing a specific algorithmic strategy for the volumetric modification step using loop closure and Reeb graph analysis [S1]. Unlike mesh-based approaches that perform surgery on extracted meshes, this method operates directly on the volume representation, allowing for out-of-core execution and preserving the manifold property of the final isosurface [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume grid $V$ of dimensions $N_x \times N_y \times N_z$, a threshold $T$ for handle size, and a buffer size $b$ for out-of-core processing.
*   **Output:** A modified scalar volume grid $V'$ where handles with size $< T$ have been removed, and the corresponding isosurface has reduced genus.

**Data Structures:**
*   **Volume Grid:** A 3D array of scalar values. Access is managed via a sliding window of $b$ slices to support out-of-core processing.
*   **Augmented Reeb Graph:** Incrementally constructed during the sweep.
    *   **Contour Nodes:** Represent connected components of the pre-image of the function (intersections of the surface with $z$-planes).
    *   **Ribbon Nodes:** Represent connected components of the surface between contours.
    *   **Edges:** Connect contour nodes to ribbon nodes, representing the topology of the isosurface.
*   **Loop Data:** Stores the sequence of vertices/edges forming the Reeb loop and the Cross loop for each identified handle.

**Algorithm Steps:**

1.  **Initial Pass (Statistics Gathering):**
    *   Perform a sweep through the volume to gather statistics on handle sizes.
    *   Compute handle sizes for discovered handles (see Step 3).
    *   Generate a histogram or scatterplot of handle sizes to determine an appropriate threshold $T$ if not provided by the user. Typical extraneous handles have loop lengths of 4–8 units; $T$ is typically set to 10–20 [S1].

2.  **Main Sweep and Handle Identification:**
    *   Perform an axis-aligned sweep (e.g., along the $z$-axis) through the volume grid.
    *   Incrementally construct and analyze the augmented Reeb graph to represent the topology of the isosurface.
    *   Use a greedy method to isolate geometrically succinct handles, processing one local handle at a time without global access to the entire surface topology [S1].

3.  **Handle Measurement:**
    *   For each handle discovered in the Reeb graph, compute the length of the **Reeb loop** (the cycle in the Reeb graph corresponding to the handle) and the **Cross loop** (a cycle on the surface that intersects the Reeb loop).
    *   Compute these lengths using breadth-first searches (BFS) on the surface mesh or volume grid connectivity.
    *   The **Handle Size** $S$ is defined as the smaller of the Reeb loop length and the Cross loop length:
        $$ S = \min(\text{length}(\text{Reeb Loop}), \text{length}(\text{Cross Loop})) $$
    *   Note: The Reeb loop is a closed curve on the surface that does not separate the surface. The Cross loop is a nonseparating cut [S1].

4.  **Handle Removal (Loop Closure):**
    *   If $S < T$, remove the handle.
    *   Identify the vertices/edges of the Reeb loop on the isosurface.
    *   Construct an approximation to the minimal spanning surface for the nonplanar loop. For efficiency, use a **triangle fan** centered at the centroid of the loop vertices [S1].
    *   **Scan-convert (rasterize)** the triangle fan into the volume grid.
    *   Write positive or negative scalar values in the grid cells intersected by the triangle fan, depending on the orientation of the loop. This operation inserts a thin wall, collapsing and pinching off the handle [S1].
    *   *Note:* The algorithm finds nearly shortest nonseparating cycles rather than globally minimal ones to support out-of-core processing [S1].

5.  **Reeb Graph Reconstruction:**
    *   After loop closure, the topology of the volume has changed.
    *   Locally rebuild the Reeb graph to account for the resulting changes.
    *   If new components or handles were introduced (e.g., due to self-intersection of the fan or nested handles), they will appear in the reconstructed graph and be processed in subsequent iterations [S1].
    *   Repeat Steps 2-5 until no handles with size $< T$ remain.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topology_simplification(volume, buffer_size, threshold):
    """
    Simplifies topology of isosurface in volume by removing small handles.
    
    Args:
        volume: 3D array of scalar values (N_x, N_y, N_z)
        buffer_size: Number of slices to keep in memory (b)
        threshold: Maximum handle size to remove (T)
        
    Returns:
        modified_volume: 3D array with handles removed
    """
    # Initialize Reeb Graph
    reeb_graph = AugmentedReebGraph()
    
    # Initial pass to determine threshold if needed
    # (Assuming threshold is provided or determined via histogram)
    
    # Main sweep loop
    # The sweep direction is typically Z. We process slices incrementally.
    # To support out-of-core, we maintain a buffer of 'buffer_size' slices.
    
    current_slice_index = 0
    max_slices = volume.shape[2]
    
    # Load initial buffer
    buffer = load_slices(volume, 0, buffer_size)
    
    while current_slice_index < max_slices:
        # Update Reeb Graph with current slice data
        reeb_graph.update(buffer)
        
        # Identify handles in the current context of the Reeb Graph
        handles = reeb_graph.get_handles()
        
        for handle in handles:
            # Measure handle size
            reeb_loop = handle.reeb_loop
            cross_loop = handle.cross_loop
            
            reeb_len = compute_loop_length(reeb_loop, volume)
            cross_len = compute_loop_length(cross_loop, volume)
            
            handle_size = min(reeb_len, cross_len)
            
            # Check threshold
            if handle_size < threshold:
                # Remove handle via loop closure
                remove_handle(volume, reeb_loop, buffer_size)
                
                # Rebuild Reeb Graph locally to reflect changes
                reeb_graph.rebuild_local(handle)
                
                # Note: New handles may have been created. 
                # The loop continues to process them in subsequent iterations.
        
        # Advance buffer
        current_slice_index += 1
        if current_slice_index + buffer_size <= max_slices:
            buffer = load_slices(volume, current_slice_index, buffer_size)
        else:
            buffer = load_slices(volume, current_slice_index, max_slices - current_slice_index)
            
    return volume

def compute_loop_length(loop, volume):
    """
    Computes the length of a loop (sequence of vertices/edges) 
    using BFS on the surface connectivity derived from the volume.
    """
    # Implementation detail: Convert loop vertices to surface mesh indices
    # Then perform BFS to count steps or sum edge lengths
    # For discrete grid, length is often number of edges
    return len(loop.edges)

def remove_handle(volume, reeb_loop, buffer_size):
    """
    Removes a handle by scan-converting a triangle fan over the Reeb loop.
    """
    # 1. Get vertices of the Reeb loop
    vertices = reeb_loop.vertices
    
    # 2. Compute centroid
    centroid = sum(vertices) / len(vertices)
    
    # 3. Create triangle fan
    triangles = []
    for i in range(len(vertices)):
        v1 = vertices[i]
        v2 = vertices[(i + 1) % len(vertices)]
        triangles.append((centroid, v1, v2))
        
    # 4. Scan-convert triangles into volume
    # Determine orientation (normal direction)
    normal = compute_normal(triangles[0])
    
    for tri in triangles:
        rasterize_triangle(volume, tri, normal, buffer_size)

def rasterize_triangle(volume, triangle, normal, buffer_size):
    """
    Rasterizes a triangle into the volume grid, modifying scalar values.
    Writes positive or negative values to collapse the handle.
    """
    # Bounding box of triangle
    min_z = min(triangle[0].z, triangle[1].z, triangle[2].z)
    max_z = max(triangle[0].z, triangle[1].z, triangle[2].z)
    
    # Iterate over relevant slices
    for z in range(int(min_z), int(max_z) + 1):
        # Get slice intersection
        intersection = get_slice_intersection(triangle, z)
        
        # Rasterize 2D polygon on slice
        if intersection:
            # Modify volume values
            # Sign depends on normal orientation relative to sweep direction
            sign = 1 if normal.z > 0 else -1
            modify_volume_slice(volume, intersection, sign)
```

**Parameters:**
*   **Threshold ($T$):** User-provided scalar value representing the maximum size (length of nonseparating cut) for a handle to be removed. Typical values range from 10–20 units of cube edge size [S1].
*   **Buffer Size ($b$):** Number of volume slices kept in memory at any time. Flexible based on volume size and memory resources. Used $b=50$ for all example volumes in the source [S1].

**Edge Cases and Pitfalls:**
*   **Self-Intersection:** The triangle fan closing the nonplanar loop may be self-intersecting or intersect other regions of the surface (e.g., if the handle contains a nested handle). In practice, this has not occurred for small handles from measurement imperfections [S1].
*   **Cascading Handle Creation:** Loop closure could introduce new handles, potentially causing termination issues. In a discrete volume grid, this is limited by the size of features representable by grid samples. If termination issues persist, modify loop closure to only add or only remove material, guaranteeing termination [S1].
*   **Large Handle Loop Computation:** Reeb and cross loop computation for handles with length $> b$ may require reloading previously flushed slices from memory. This reloading is rare (e.g., only 5 of 106 handles in the Buddha model required it) [S1].

## Subtopics

### [Improved Topological Simplification via Gradient Reversal](11670-improved-topological-simplification-via-gradient-reversal.md)
This method modifies a scalar volume field to correspond to a simplified contour tree by reversing the gradient flow along a path from a saddle to an extremum, allowing for smoother evolution of isosurfaces compared to simple pruning. It solves the problem of visual artifacts (such as "chopping off" peaks) that occur when topological features are removed by simply deleting data or flattening regions, ensuring the resulting isosurface remains a manifold and the scalar field approximation is closer to the original. It builds on [Topology Simplification via Volumetric Modification](9327-topology-simplification-via-volumetric-modification.md) by providing a specific algorithmic strategy for the volumetric modification step using tube-based gradient reversal.

## Operating characteristics
**Time Complexity:**
*   Dominated by volume traversal $O(n^3)$ where $n$ is the grid extent [S1].
*   Handle processing is input-sensitive: for each loop, complexity is $O(d^2)$ where $d$ is the minimal loop length [S1].
*   Reeb graph reconstruction cost is $O(s \cdot n)$ where $s$ is the number of slices with $n$ polygons needing reconstruction [S1].

**Space Complexity:**
*   Typically $O(n)$ for Reeb graph and polygon storage using a buffer of $b$ slices [S1].
*   Worst-case space complexity is $O(n^3)$ for pathological volumes with space-filling handles [S1].
*   In practice, polygon storage is within $O(n)$ and the algorithm operates out-of-core [S1].

**Measured Performance:**
*   **David Model:** 885x736x709 grid, 15,244,302 faces, genus reduced from 1063 to 0, 1063 loops collapsed, 76 intra-ribbon handles, 87.5 CPU minutes on Pentium Xeon 2 Ghz [S1].
*   **Buddha Model:** 400x400x950 grid, 4,736,292 faces, genus reduced from 106 to 6, 100 loops collapsed, 26 intra-ribbon handles, 6.5 CPU minutes [S1].
*   **Brain Models:** 125x255x255 grid. Brain1: 688,248 faces, genus 366->0, 2.8 min. Brain2: 452,050 faces, genus 21->0, 0.7 min. Brain3: 529,012 faces, genus 41->0, 0.6 min. Brain4: 699,566 faces, genus 50->0, 1.7 min [S1].

**Strengths and Limitations:**
*   **Strengths:** Operates directly on volume, ensuring manifold output; supports out-of-core processing; preserves geometric detail by targeting small handles; facilitates subsequent surface operations like mesh simplification and parametrization [S1].
*   **Limitations:** Input-sensitive; may require reloading slices for large handles; potential for cascading handle creation (though limited in practice); triangle fan approximation may self-intersect for complex nested handles [S1].

**When to Use:**
*   When the input is volumetric data (CT, MRI, scanned models) and the goal is to remove topological noise (handles) before further processing (simplification, remeshing, parametrization) [S1].
*   When out-of-core processing is required for large volumes [S1].

**When Not to Use:**
*   When the input is already a mesh and volumetric data is not available or desirable to modify [S1].
*   When global optimality of handle removal is required (this method uses a greedy, local approach) [S1].

## Background
The method was introduced by [S1] (David et al., 2004) as a practical approach for removing handles in isosurfaces. It distinguishes itself from mesh-based approaches (e.g., Guskov and Wood [2001]) by operating directly on the volume representation. The method is designed to facilitate out-of-core execution and uses an augmented Reeb graph to represent the topology of the isosurface. It has been demonstrated on complex models including the David, Buddha, Dragon, and Brain models, showing significant genus reduction and improved geometrical quality for subsequent surface processing [S1].

## Sources
[S1] Removing excess topology from isosurfaces, https://doi.org/10.1145/990002.990007, fetched 2026-11-03
