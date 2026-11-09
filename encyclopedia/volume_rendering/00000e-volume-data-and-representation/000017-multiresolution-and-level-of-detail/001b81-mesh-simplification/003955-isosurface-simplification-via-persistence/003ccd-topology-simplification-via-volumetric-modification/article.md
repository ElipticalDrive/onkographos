# Topology Simplification via Volumetric Modification

Also called: Genus Reduction, Handle Removal, Handle collapsing, Handle pinching, Loop Closure for Handle Removal, Topological simplification of isosurfaces, Topology Simplification via Handle Removal

## Overview
This method removes extraneous topological handles from isosurfaces extracted from volumetric data (e.g., CT, MRI) by modifying the scalar volume values rather than performing mesh surgery, ensuring the resulting isosurface remains a manifold [S1]. It addresses the problem of identifying and removing insignificant topological features based on their persistence, building on [Isosurface Simplification via Persistence](8470-isosurface-simplification-via-persistence.md). Unlike mesh-based approaches that perform surgery on extracted meshes, this method operates directly on the volume representation, allowing for out-of-core execution and preserving the manifold property of the final isosurface [S1].

## How it works
The algorithm performs an axis-aligned sweep through the volume grid to locate handles, compute their sizes, and selectively remove them [S1]. It uses a greedy method to isolate geometrically succinct handles, processing a single local handle at a time before moving to the next, avoiding global access to the entire surface topology [S1].

### Inputs and Outputs
- **Input**: A scalar volume grid $V$ of dimensions $N_x \times N_y \times N_z$. A threshold value $T$ for handle size. A buffer size $b$ for out-of-core processing.
- **Output**: A modified scalar volume grid $V'$ with reduced genus.

### Data Structures
- **Volume Grid**: A 3D array of scalar values. Access is managed via a sliding window of $b$ slices to support out-of-core execution [S1].
- **Augmented Reeb Graph**: Incrementally constructed during the sweep. Contains:
    - **Contour Nodes**: Represent connected components of the pre-image of the function (e.g., intersections of the surface with $z$-planes).
    - **Ribbon Nodes**: Represent connected components of the surface between contours [S1].
- **Loop Data**: For each handle, the algorithm stores the vertices of the Reeb loop and the Cross loop [S1].

### Parameters
- **Threshold ($T$)**: The maximum size (length of the shortest nonseparating cut) for a handle to be removed. Typical values range from 10–20 units of cube edge size, determined by examining a histogram of handle sizes where extraneous handles typically have loop lengths of 4–8 [S1].
- **Buffer Size ($b$)**: The number of volume slices kept in memory at any time. Typical value $b=50$ [S1].

### Algorithm Steps

1.  **Initial Pass**: Make an initial pass over the volume to gather statistics on handle sizes. Examine these using a scatterplot or histogram to select an appropriate threshold $T$ [S1].

2.  **Sweep and Reeb Graph Construction**: Perform an axis-aligned sweep (e.g., along the $z$-axis) through the volume grid. Incrementally construct and analyze the augmented Reeb graph to represent the topology of the isosurface [S1].

3.  **Handle Identification**: For each handle discovered in the Reeb graph:
    - Compute the length of the Reeb loop and the Cross loop using breadth-first searches (BFS) on the surface mesh derived from the current volume slice and adjacent slices [S1].
    - The handle size is defined as the smaller of the Reeb loop length and the Cross loop length [S1].
    - Note: If the loop length exceeds the buffer size $b$, previously flushed slices may need to be reloaded from disk. This is rare in practice [S1].

4.  **Handle Removal Decision**: If the handle size is less than the threshold $T$:
    - Proceed to removal.
    - If the handle size is $\ge T$, skip to the next handle.

5.  **Loop Closure (Volumetric Modification)**:
    - Identify the nonseparating cycle (loop) to be collapsed. The algorithm finds nearly shortest nonseparating cycles rather than globally minimal-length ones to support out-of-core processing [S1].
    - Construct an approximation to the minimal spanning surface for the nonplanar loop. For efficiency, use a triangle fan about the centroid of the loop [S1].
    - **Scan-convert (rasterize)** the triangle fan into the volume grid. Write positive or negative scalar values in the grid depending on the orientation of the loop. This operation collapses or pinches the handle by inserting a thin wall [S1].
    - *Note*: The triangle fan may be self-intersecting or intersect other regions of the surface if the handle contains a nested handle. In practice, this has not occurred for small handles from measurement imperfections [S1].

6.  **Reeb Graph Reconstruction**: If loop closure introduces new handles or components, locally rebuild the Reeb graph. Any new components or handles will appear in the reconstructed graph and be subsequently processed [S1].

7.  **Termination**: Repeat steps 3-6 until no handles with size $< T$ remain. In a discrete volume grid, cascading handle creation is limited by the size of features representable by grid samples. If termination issues persist, modify loop closure to only add or only remove material, guaranteeing termination [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topology_simplification_via_volumetric_modification(volume, threshold, buffer_size):
    """
    Simplifies topology of isosurfaces in a volume by removing handles.
    
    Args:
        volume: 3D array of scalar values (Nx x Ny x Nz).
        threshold: Maximum handle size (loop length) to remove.
        buffer_size: Number of slices to keep in memory.
        
    Returns:
        modified_volume: 3D array with simplified topology.
    """
    # Step 1: Initial Pass for Threshold Selection
    # Gather statistics on handle sizes using a histogram or scatterplot.
    # Set threshold if not provided.
    # [G] Assume threshold is provided or computed here.
    
    # Step 2: Initialize Out-of-Core Buffer
    # Load first 'buffer_size' slices into memory.
    current_buffer = load_slices(volume, 0, buffer_size)
    
    # Step 3: Sweep and Reeb Graph Construction
    # Incrementally construct Reeb graph as we sweep through the volume.
    reeb_graph = AugmentedReebGraph()
    
    for z in range(0, volume.shape[2], slice_step):
        # Update buffer: load next slice, discard old if necessary
        current_buffer = update_buffer(current_buffer, volume, z, buffer_size)
        
        # Update Reeb graph based on changes in the current slice
        reeb_graph.update(current_buffer)
        
        # Step 4: Handle Identification
        handles = reeb_graph.get_handles()
        
        for handle in handles:
            # Compute Reeb and Cross loop lengths
            reeb_loop = handle.reeb_loop
            cross_loop = handle.cross_loop
            
            # If loops are larger than buffer, reload necessary slices
            if len(reeb_loop) > buffer_size or len(cross_loop) > buffer_size:
                reload_slices_for_loop(volume, reeb_loop, cross_loop, buffer_size)
            
            reeb_length = compute_loop_length(reeb_loop)
            cross_length = compute_loop_length(cross_loop)
            
            handle_size = min(reeb_length, cross_length)
            
            # Step 5: Handle Removal Decision
            if handle_size < threshold:
                # Perform Loop Closure
                remove_handle(volume, handle, current_buffer)
                
                # Step 6: Reeb Graph Reconstruction
                # Locally rebuild Reeb graph to account for topological changes
                reeb_graph.rebuild_local(handle)
                
                # Note: New handles may appear and will be processed in subsequent iterations
```

### Edge Cases and Pitfalls
- **Self-Intersecting Fans**: The triangle fan closing the nonplanar loop may be self-intersecting or intersect other regions of the surface (e.g., if the handle contains a nested handle). In practice, this has not occurred for small handles from measurement imperfections [S1].
- **Cascading Handle Creation**: Loop closure could introduce new handles, potentially causing termination issues. However, in a discrete volume grid, this is limited by the size of features representable by grid samples. If termination issues persist, modify loop closure to only add or only remove material, guaranteeing termination [S1].
- **Large Handle Loop Computation**: Reeb and cross loop computation for handles with length $> b$ may require reloading previously flushed slices from memory. This reloading is rare; e.g., only 5 of 106 handles in the Buddha model required it [S1].

## Operating characteristics
- **Time Complexity**: Dominated by volume traversal $O(n^3)$ where $n$ is the grid extent. Handle processing is input-sensitive: for each loop, complexity is $O(d^2)$ where $d$ is the minimal loop length. Reeb graph reconstruction cost is $O(s \cdot n)$ where $s$ is the number of slices with $n$ polygons needing reconstruction [S1].
- **Space Complexity**: Typically $O(n)$ for Reeb graph and polygon storage using a buffer of $b$ slices. Worst-case space complexity is $O(n^3)$ for pathological volumes with space-filling handles. In practice, polygon storage is within $O(n)$ and the algorithm operates out-of-core [S1].
- **Measured Performance**:
    - **David Model**: 885x736x709 grid, 15,244,302 faces, genus reduced from 1063 to 0, 1063 loops collapsed, 76 intra-ribbon handles, 87.5 CPU minutes on Pentium Xeon 2 Ghz [S1].
    - **Buddha Model**: 400x400x950 grid, 4,736,292 faces, genus reduced from 106 to 6, 100 loops collapsed, 26 intra-ribbon handles, 6.5 CPU minutes [S1].
    - **Brain Models**: 125x255x255 grid. Brain1: 688,248 faces, genus 366->0, 2.8 min. Brain2: 452,050 faces, genus 21->0, 0.7 min. Brain3: 529,012 faces, genus 41->0, 0.6 min. Brain4: 699,566 faces, genus 50->0, 1.7 min [S1].
- **Strengths**: Operates directly on volume data, ensuring the final isosurface remains a manifold. Facilitates out-of-core execution. Reduces genus, facilitating easier parametrization and fewer charts for subsequent processing [S1].
- **Limitations**: Input-sensitive. May require reloading slices for large loops. Approximate loop finding (nearly shortest) rather than globally minimal [S1].

## Background
The method was introduced by [S1] to address the problem of extraneous topological handles in isosurfaces reconstructed from scanned data (e.g., laser range data, CT, MRI). These artifacts hinder subsequent operations like mesh simplification, remeshing, and parametrization. The approach distinguishes itself from mesh-based approaches (e.g., Guskov and Wood [2001]) by operating on the volume representation directly. It is used in medical applications (e.g., brain surface analysis) and computer graphics for preparing models for remeshing and parametrization [S1].

## Sources
[S1] Removing excess topology from isosurfaces, https://doi.org/10.1145/990002.990007, fetched 2026-11-03
