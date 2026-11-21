# Octree Subdivision

Also called: LiNode::subdivide

## Overview
Octree Subdivision is the fundamental operation for refining an adaptive octree node into eight children by splitting its bounding box in half along each axis [S1, unofficial]. It solves the problem of resolving high-frequency geometry or irradiance gradients that cannot be captured by a coarse node, enabling variable granularity in large, complex scenes [S1, unofficial]. Unlike static subdivision, it dynamically determines refinement based on voxel fragments or implicit surface constraints [S1, unofficial][S3].

## How it works
The method transforms a single parent node into eight child nodes. The specific algorithm depends on the application context: standard geometric subdivision for general volume data or isotopic subdivision for implicit surfaces.

### Inputs and Outputs
*   **Input**: A parent node defined by its bounding box $[\mathbf{min}, \mathbf{max}]$ and potentially a scalar field function $F$ (for isotopic cases).
*   **Output**: Eight child nodes, each with a sub-region of the parent's bounding box.

### Data Structures
*   **Node**: Contains bounding box coordinates $\mathbf{min}$ and $\mathbf{max}$ (vectors of size 3).
*   **Child Index**: Integer $i \in [0, 7]$.

### Standard Geometric Subdivision
This variant redistributes existing surfels or simply creates the spatial hierarchy [S1, unofficial].

1.  **Calculate Half-Size**: Compute the vector $\mathbf{s} = (\mathbf{max} - \mathbf{min}) \times 0.5$.
2.  **Validation**: If the length of $\mathbf{s}$ is less than a small epsilon (e.g., $10^{-8}$), abort with an error to prevent degenerate nodes [S1, unofficial].
3.  **Iterate Children**: For each child index $i$ from 0 to 7:
    *   Determine bit components: $x = i \pmod 2$, $y = \lfloor i/2 \rfloor \pmod 2$, $z = \lfloor i/4 \rfloor$.
    *   Compute child minimum: $\mathbf{min}_i = \mathbf{min} + \mathbf{s} \times (x, y, z)$.
    *   Compute child maximum: $\mathbf{max}_i = \mathbf{min}_i + \mathbf{s}$.
    *   Create child node $C_i$ with bounds $[\mathbf{min}_i, \mathbf{max}_i]$ [S1, unofficial].

### Isotopic Subdivision
Used to create an isotopic mesh of an implicit surface, ensuring the topological equivalence between the surface and the octree approximation [S3].

1.  **Subdivision Loop**: Starting with a root bounding box $B$, iteratively subdivide leaves until the stopping condition is met for every leaf node $C$.
2.  **Stopping Condition**: A leaf $C$ is not subdivided if:
    $$0 \notin \Box F(C) \quad \vee \quad \langle \Box \nabla F(C), \Box \nabla F(C) \rangle > 0$$
    where $\Box F(C)$ is the interval evaluation of the function over the cell, and $\Box \nabla F(C)$ is the interval evaluation of the gradient [S3].
3.  **Balancing**: After subdivision, balance the octree to ensure regularity [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subdivide_node(parent_min, parent_max, method="standard", F=None, grad_F=None):
    """
    Subdivides an octree node.
    
    Args:
        parent_min: Vector3, minimum corner of parent bounding box.
        parent_max: Vector3, maximum corner of parent bounding box.
        method: String, "standard" or "isotopic".
        F: Function, scalar field evaluator (required for isotopic).
        grad_F: Function, gradient evaluator (required for isotopic).
        
    Returns:
        List of 8 child nodes, each containing min and max bounds.
    """
    # Step 1: Calculate half-size
    half_size = (parent_max - parent_min) * 0.5
    
    # Step 2: Validation (Standard only)
    if method == "standard":
        if half_size.length() < 1e-8:
            raise ValueError("Node too small to subdivide")
            
    children = []
    
    # Step 3: Iterate over 8 octants
    for i in range(8):
        # Determine bit components
        x = i % 2
        y = (i // 2) % 2
        z = i // 4
        
        # Compute child bounds
        offset = half_size * Vector3(x, y, z)
        child_min = parent_min + offset
        child_max = child_min + half_size
        
        # Isotopic Check
        if method == "isotopic":
            # Evaluate interval bounds of F and gradient over child cell
            F_interval = F.interval(child_min, child_max)
            Grad_interval = grad_F.interval(child_min, child_max)
            
            # Check stopping condition:
            # 0 not in F_interval OR gradient norm squared > 0
            # Note: <Grad, Grad> > 0 implies non-zero gradient interval
            zero_in_F = F_interval.contains(0)
            grad_norm_sq = Grad_interval.dot(Grad_interval)
            
            if zero_in_F and grad_norm_sq <= 0:
                # Condition not met, this node needs further subdivision
                # In a recursive top-down pass, we would call subdivide_node recursively here
                # For this atomic function, we return the node, but the caller
                # must decide whether to keep it or subdivide further based on this result.
                pass 
            else:
                # Leaf node reached
                pass
        
        children.append({
            "min": child_min,
            "max": child_max,
            "index": i
        })
        
    return children
```

### Parameters and Edge Cases
*   **Epsilon**: $10^{-8}$ is used to prevent division by zero or infinite recursion in degenerate cases [S1, unofficial].
*   **Isotopy**: The condition $\langle \Box \nabla F(C), \Box \nabla F(C) \rangle > 0$ ensures the gradient does not vanish within the interval, preventing topological errors [S3].
*   **Pitfall**: In standard subdivision, failing to check node size leads to infinite loops or memory exhaustion on degenerate inputs [S1, unofficial].

## Subtopics

### [Node Subdivision](9141-node-subdivision.md)
Node Subdivision is the core mechanism for refining an adaptive octree during top-down construction, transforming coarse nodes into finer 2×2×2 tiles to resolve high-frequency geometry or irradiance gradients. It differs from static subdivision by dynamically determining which nodes require refinement based on a pre-computed list of voxel fragments, and from uniform grid approaches by allowing variable granularity. This method enables efficient memory usage by only allocating child nodes where necessary, addressing the unwieldiness of uniform grids in large, complex scenes.

### Dimensional Splitting
Dimensional Splitting is a variant where subdivision is not uniform across all axes. At each step, each dimension is either halved or not subdivided depending on the other two dimensions. For example, if a node's extent is $(4, 2, 2)$, only the dimension with extent 4 is subdivided. This can cause the octree to resemble a quadtree or binary tree in some levels depending on volume dimensions and leaf-node size [S2, unofficial].

### Octree Subdivision for Isotopic Meshing
This variant uses a balanced octree to create an isotopic mesh of an implicit surface, ensuring isotopy between the surface and the octree approximation. The process involves subdividing leaves until specific interval constraints on the function and its gradient are met, followed by balancing the octree [S3].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per node subdivided, but total time depends on the number of nodes refined. For isotopic subdivision, it depends on the complexity of the implicit surface and the cost of interval arithmetic.
*   **Memory Complexity**: Increases by a factor of 8 for each subdivided node (plus overhead for child pointers).
*   **Strengths**: Allows high resolution only where needed; isotopic subdivision guarantees topological correctness.
*   **Limitations**: Standard subdivision can create unbalanced trees if not managed; isotopic subdivision requires expensive interval evaluations.
*   **When to use**: Use for adaptive volume rendering where data density varies; use isotopic variant for accurate implicit surface extraction.

## Background
The concept of octree subdivision is foundational in spatial data structures. The specific implementation details cited here originate from various sources: the standard geometric subdivision logic is associated with the Mr-Voxel project [S1, unofficial], dimensional splitting is described in the context of scalable web-based volume rendering [S2, unofficial], and isotopic subdivision for implicit surfaces is detailed in research on topological approximation [S3].

## Sources
[S1] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29
