# Parallel Octree Data Reduction

Also called: Adaptive Multiresolution Subsampling, Adaptive Subsampling, Adaptive multiresolution visualization using parallel octrees, Parallel Octree Subsampling

## Overview
Parallel Octree Data Reduction is a technique for the interactive visualization of large scientific datasets by combining hierarchical data reduction with parallel computing [S1, unofficial]. It distributes data across Massively Parallel Processing (MPP) processors to build a distributed-memory octree, allowing users to interactively adapt resolution (refine or prune) on a local graphics workstation while retaining access to full-resolution data [S1, unofficial]. Unlike uniform grid subsampling, which uses a fixed grid and allows more grid points but higher approximation error in complex regions, this method places more points where data changes rapidly to minimize approximation error [S2]. It builds on [Round-robin Leaf Assignment](12287-round-robin-leaf-assignment.md) by providing the data structure and reduction logic that precedes or accompanies the distribution strategy.

## How it works
**Inputs and Outputs**
*   **Input:** File-based scalar field data requiring only spatial coordinate information; no connectivity information is necessary [S2]. The input format supports unstructured tetrahedral and hexahedral, block adaptive, and uniform meshes [S2].
*   **Output:** A reduced multiresolution dataset represented by a distributed octree. For each leaf octant, the output includes:
    *   The agglomerated field data value (typically the mean) [S1, unofficial].
    *   Statistical error indicators: normalized standard deviation ($a_n$) and normalized maximum deviation from the mean ($e_n$) [S1, unofficial].
    *   Spatial information for interoctant links [S1, unofficial].

**Data Structures**
The core data structure is a distributed octree [S1, unofficial].
*   **Leaf Octant:** Contains the agglomerated scalar value, the standard deviation ($\sigma$), the maximum deviation from the mean ($\epsilon$), and the mean ($\mu$) used for normalization [S1, unofficial].
*   **Interoctant Links:** Links between parent and child nodes, and between sibling nodes. These links may be local (within the same processor) or off-processor [S1, unofficial].
*   **Local Root Structure:** Off-processor parent links are represented by a local root structure containing spatial information. This enables local point searches without communication [S1, unofficial].

**Algorithm Steps**
1.  **Insertion:** Loop through the input field data. For each data point, insert its spatial location and field data value into the appropriate leaf octant of the octree [S1, unofficial].
2.  **Refinement Criterion Evaluation:** Evaluate each leaf octant according to a specified criterion [S2].
    *   The criterion determines if the current leaf octant should divide to create eight new leaf octants [S1, unofficial].
    *   Default criterion: Divide an octant leaf when a user-defined maximum number of elements have been inserted into it [S1, unofficial]. This maintains approximately uniform fidelity for h-adaptive meshes [S1, unofficial].
    *   Alternative criteria: User-specified bounds on standard deviation or maximum deviation, or an automatically determined error bound given a performance constraint (maximum target number of leaf octants) [S2].
    *   For uniform meshes or p-adaptive methods, the max-element criterion is insufficient; error indicator techniques such as energy norms or gradient bounds should be used [S1, unofficial].
3.  **Splitting and Reassignment:** If the criterion indicates refinement is necessary, split the leaf octant and reassign its associated data points to the new child leaf octants [S2].
4.  **Aggregation and Statistics:** After insertion (and any subsequent reassignments), aggregate the field data values within each leaf octant [S1, unofficial]. Typically, this involves averaging the values [S1, unofficial].
5.  **Error Computation:** For each leaf octant, compute statistical values:
    *   Standard deviation ($\sigma$) [S1, unofficial].
    *   Maximum deviation from the mean ($\epsilon$) [S1, unofficial].
    *   Mean ($\mu$) [S1, unofficial].
6.  **Normalization:** Normalize the standard deviation and maximum deviation by the mean to yield normalized error indicators:
    $$ a_n = \sigma / \mu $$
    $$ e_n = \epsilon / \mu $$
    These values are included as additional scalar fields alongside the reduced data [S1, unofficial].
7.  **Distribution:** Distribute the octree across the processors of the MPP using an octree partitioning algorithm and arbitrary octant migration for load balancing [S1, unofficial].
8.  **Traversal and Output Generation:** Form the reduced data set by traversing each local octree and computing the values on each leaf. This is done on each processor in linear time with respect to the local number of data points [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_octree_reduction(field_data, max_elements_per_octant=None, target_leaf_count=None):
    """
    field_data: List of tuples (x, y, z, value)
    max_elements_per_octant: Threshold for subdivision based on element count
    target_leaf_count: Threshold for subdivision based on total leaf count
    """
    octree = DistributedOctree()
    
    # Step 1 & 2: Insertion and Refinement
    for x, y, z, value in field_data:
        leaf = octree.find_leaf(x, y, z)
        leaf.insert(value)
        
        # Evaluate criterion
        should_refine = False
        if max_elements_per_octant is not None:
            if leaf.element_count > max_elements_per_octant:
                should_refine = True
        elif target_leaf_count is not None:
            # Check if total leaves exceed target (requires global state or communication)
            if octree.total_leaves > target_leaf_count:
                should_refine = True
        else:
            # Default or custom criterion
            should_refine = leaf.evaluate_criterion()
            
        if should_refine:
            octree.split_leaf(leaf)
            # Reassign points from the old leaf to new children
            octree.reassign_points(leaf)
            
    # Step 4, 5, 6: Aggregation, Statistics, Normalization
    for leaf in octree.all_leaves():
        leaf.compute_statistics()
        mu = leaf.mean
        sigma = leaf.std_dev
        epsilon = leaf.max_deviation
        
        # Avoid division by zero
        if mu == 0:
            leaf.normalized_std_dev = 0
            leaf.normalized_max_dev = 0
        else:
            leaf.normalized_std_dev = sigma / mu
            leaf.normalized_max_dev = epsilon / mu
            
    # Step 7: Distribution is handled by the DistributedOctree infrastructure
    # Step 8: Traversal
    reduced_data = []
    for leaf in octree.local_leaves():
        reduced_data.append({
            'center': leaf.center,
            'value': leaf.mean,
            'error_std': leaf.normalized_std_dev,
            'error_max': leaf.normalized_max_dev
        })
        
    return reduced_data
```

**Parameters**
*   **User-defined maximum number of elements per octant:** The threshold for subdividing an octant leaf. Typical value: Not specified, but used to maintain uniform fidelity for h-adaptive meshes [S1, unofficial].
*   **Criterion:** A threshold controlling refinement. Can be a user-specified bound on standard deviation, maximum deviation, or an automatically determined error bound given a performance constraint (maximum target number of leaf octants) [S2].
*   **Input format:** File-based scalar field data requiring only spatial coordinate information; no connectivity information is necessary [S2].

**Edge Cases and Pitfalls**
*   **Uniform/P-adaptive Meshes:** The max-element criterion is insufficient for uniform meshes or p-adaptive methods. Error indicator techniques such as energy norms or gradient bounds are required to determine subdivision [S1, unofficial].
*   **Approximation Error Scaling:** Approximation errors increase as the original problem size increases, because a smaller percentage of points can be used in the reduced dataset due to local workstation memory limits [S2]. This is mitigated by adaptive refinement at the cost of increased communication [S2].
*   **Complex Algorithms:** Complex algorithms operating on the reduced data set (such as streamlines or isosurface generation) may require fat-neighbor links or a guarantee of maximum level-difference between adjacent octants. Providing this functionality complicates octree insertion [S1, unofficial].

## Subtopics

### [AMR Structure-Preserving Parallel Volume Rendering](17552-amr-structure-preserving-parallel-volume-rendering.md)
A parallel volume rendering method that preserves the Adaptive Mesh Refinement (AMR) block structure, distributing leaves of the octree among processors and using raycasting with adaptive sampling. It differs from uniform resampling methods by avoiding the overhead of converting hierarchical data to a single uniform grid, instead maintaining the native AMR hierarchy to optimize memory and computation for regions of varying resolution.

## Operating characteristics
*   **Time Complexity:** The reduced data set is formed by traversing each local octree and computing the values on each leaf. This is done on each processor in linear time with respect to the local number of data points [S1, unofficial].
*   **Memory Complexity:** The maximum resolution of the reduced data set is limited by the memory size and speed of the local graphics workstation [S2].
*   **Performance:** Used for interactive remote exploration of large terabyte/petabyte scalar field datasets on desktop workstations connected via high-speed networks [S2].
*   **Strengths:** Minimizes approximation error by placing more points where data changes rapidly [S2]. Allows interactive adaptation of resolution (refine or prune) on a local workstation [S1, unofficial].
*   **Limitations:** Approximation errors increase with problem size due to workstation memory limits [S2]. Complex algorithms may require additional links (fat-neighbor) which complicate insertion [S1, unofficial].
*   **When to use:** Interactive remote exploration of large scalar field datasets where approximation error must be minimized in complex regions [S2].
*   **When not to use:** When the data is uniform and a fixed grid subsampling would suffice, or when the workstation memory is too small to hold even the reduced dataset [S2].

## Background
Parallel Octree Data Reduction was developed to address the challenge of visualizing large scientific datasets that exceed the performance and memory capacity of powerful graphics workstations [S1, unofficial]. The technique combines hierarchical data reduction with parallel computing to allow interactive exploration while retaining full-resolution capability [S1, unofficial]. It was developed by researchers at Argonne National Laboratory, including Lori A. Freitag and Raymond M. Loy [S2]. The system uses a client/server model based on TCP/IP and Unix sockets, implemented via the ALICE Memory Snooper (AMS) from Argonne National Laboratory, for communication between the octree code and the desktop graphics application [S2]. The graphics application uses JAVA Swing components to provide a GUI to VTK classes [S2].

## Sources
[S1] Adaptive, multiresolution visualization of large data sets using parallel octrees., https://digital.library.unt.edu/ark:/67531/metadc624851/?q=multiresolution volume rendering, fetched 2026-10-28 (unofficial)
[S2] Comparison of remote visualization strategies for interactive exploration of large data sets, https://digital.library.unt.edu/ark:/67531/metadc710831/?q=multiresolution volume rendering, fetched 2026-10-28
