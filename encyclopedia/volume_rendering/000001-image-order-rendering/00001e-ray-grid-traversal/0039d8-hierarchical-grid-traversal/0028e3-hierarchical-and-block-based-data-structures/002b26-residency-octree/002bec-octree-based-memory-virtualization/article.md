# Octree-based memory virtualization

Also called: Crassin et al. Octree-based memory virtualization, Interactive Visualization via Memory Mapping

## Overview
Octree-based memory virtualization is an out-of-core rendering technique for large-scale volume datasets that decouples the physical memory footprint from the logical data size by mapping data files directly to virtual memory [S2, unofficial]. It solves the problem of limited RAM when rendering massive multi-channel volume data by using an octree structure to progressively load only the necessary bricks and metadata into physical memory during traversal [S1, unofficial][S2, unofficial]. Unlike its sibling [Residency Octree](4950-residency-octree.md), which manages cache residency explicitly, this method relies on the operating system's memory manager to handle page faults, resulting in a simpler implementation but potentially higher memory overhead due to the requirement that entire subtrees be resident for high-resolution rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A multi-channel volume dataset stored as binary files, an octree structure defining the spatial subdivision, and a camera/view state.
- Output: A rendered image with alpha-blended multi-channel compositing.

**Data Structures:**
The octree nodes and bricks store information for up to $n$ channels [S1, unofficial].
- **Octree Node:** Contains pointers to children, culling metadata, and potentially average values for homogeneous bricks. For multi-channel data, a node is marked homogeneous only if its corresponding bricks in *all* channels are homogeneous [S1, unofficial].
- **Brick:** A block of voxels. In the brick cache, each entry stores data for $n$ channels [S1, unofficial].
- **Channel Limit:** The number of channels $n$ is implementation-defined and cannot be changed at run-time [S1, unofficial]. Due to GPU texture format limitations (typically four components), $n$ is practically limited to 4 [S1, unofficial].

**Steps:**
1. **Memory Mapping:** Use the OS-provided `mmap` function to map the binary data files to virtual memory [S2, unofficial]. This avoids loading the entire structure into physical memory upfront.
2. **Tree Traversal:** Perform a breadth-first traversal of the octree [S2, unofficial].
3. **Progressive Loading:** As the traversal encounters nodes, the OS loads the corresponding pages (bricks and metadata) into physical memory on demand [S2, unofficial]. This ensures only parts of the data structure actually rendered are loaded, making efficient use of physical memory [S2, unofficial].
4. **Culling and Refinement:**
   - Check culling tests for each child node. If a child fails, ignore it [S2, unofficial].
   - Calculate the screen size $S$ of the bounding sphere of the child [S2, unofficial].
   - If the child is a leaf node, render it (e.g., as a splat) [S2, unofficial].
   - If the frame rate is too low, smoothly increase the minimum regular splat size $\delta_r$ and boundary splat size $\delta_b$ to reduce rendering cost [S2, unofficial].
5. **Homogeneous Optimization:** If a brick is homogeneous (all voxels contain the same value up to a user-defined threshold), store the average value in the node instead of a pointer to the brick’s location in the brick cache [S1, unofficial]. This optimization applies only if all channels in the brick are homogeneous [S1, unofficial].
6. **Compositing:** Mix the channels using alpha blending [S1, unofficial]. Note that image-level intermixing may not handle occlusions between different channels correctly due to missing depth information in accumulated intermediate results [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_octree(node, delta_r, delta_b):
    """
    Recursively traverse the octree, relying on mmap for memory access.
    node: The current octree node.
    delta_r: Minimum regular splat size in pixels.
    delta_b: Minimum boundary splat size in pixels.
    """
    # Check culling tests for the current node's children
    for child in node.children:
        if not passes_culling_tests(child):
            continue
        
        # Calculate screen size of the child's bounding sphere
        S = screen_size_of_bounding_sphere(child)
        
        if child.is_leaf:
            # Render leaf node (e.g., as a splat)
            render_leaf(child, S, delta_r, delta_b)
        else:
            # Recursively traverse non-leaf nodes
            # The OS handles loading the child's data via mmap page faults
            traverse_octree(child, delta_r, delta_b)

def update_adaptive_resolution(frame_rate, delta_r, delta_b):
    """
    Adjust splat sizes if frame rate is too low.
    """
    if frame_rate < threshold:
        delta_r = smooth_increase(delta_r)
        delta_b = smooth_increase(delta_b)
    return delta_r, delta_b
```

**Parameters:**
- $n$: Number of channels. Implementation-defined, limited to 4 in practice [S1, unofficial].
- $\delta_r$: Minimum regular splat size in pixels [S2, unofficial].
- $\delta_b$: Minimum boundary splat size in pixels [S2, unofficial].
- Threshold: User-defined threshold for homogeneous brick detection [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Multi-channel Occlusion:** Image-level intermixing of multiple channels may fail to handle occlusions correctly because depth information is lost in accumulated intermediate results [S1, unofficial].
- **Homogeneity Check:** A node can only be marked homogeneous if *all* corresponding bricks in *all* channels are homogeneous [S1, unofficial].
- **Run-time Channel Limit:** The number of channels cannot be changed at run-time [S1, unofficial].

## Subtopics

### [Octree-based out-of-core volume rendering](5171-octree-based-out-of-core-volume-rendering.md)
An extension supporting multiple visible channels by storing pointers and culling metadata in each node. It enforces whole-subtree residency for high-resolution rendering, leading to a higher memory footprint than the [Residency Octree](4950-residency-octree.md) but comparable performance for single-channel data.

## Operating characteristics
- **Time Complexity:** Depends on the tree traversal and the number of loaded bricks. Breadth-first storage enables fast recursive refinements [S2, unofficial].
- **Memory Complexity:** Physical memory usage is proportional to the number of bricks actually rendered, not the total dataset size [S2, unofficial]. However, it requires the whole subtree to be resident for high-resolution rendering, leading to a higher memory footprint compared to more compact representations like the Residency Octree [S1, unofficial].
- **Strengths:** Efficient use of physical memory by only loading rendered parts; progressive loading; fast recursive refinements due to breadth-first storage [S2, unofficial].
- **Limitations:** Limited to 4 channels due to GPU texture formats; potential occlusion artifacts in multi-channel rendering; higher memory footprint than explicit cache management methods [S1, unofficial].
- **When to Use:** Suitable for interactive visualization of large multi-channel volume datasets where RAM is limited and the OS memory manager can effectively handle page faults [S2, unofficial].

## Background
The method was introduced by Crassin et al. [CNLE09] as a scheme for out-of-core volume rendering. It has been implemented in the Voreen framework [DLJL22] and extended by Brix et al. [BPH14] to support multi-channel data [S1, unofficial]. The approach leverages OS-level memory mapping (`mmap`) to virtualize memory access to large binary data files [S2, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] PDF TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
