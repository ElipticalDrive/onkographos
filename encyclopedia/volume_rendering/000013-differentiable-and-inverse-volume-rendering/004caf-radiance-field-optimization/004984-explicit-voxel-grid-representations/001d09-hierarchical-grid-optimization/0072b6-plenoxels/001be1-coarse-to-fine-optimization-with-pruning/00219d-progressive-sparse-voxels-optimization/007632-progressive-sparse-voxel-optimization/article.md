# Progressive Sparse Voxel Optimization

Also called: Coarse-to-fine reconstruction, Hierarchical Optimization

## Overview
Progressive Sparse Voxel Optimization is a coarse-to-fine reconstruction strategy for explicit radiance fields that gradually refines a 3D scene representation by alternating between field optimization and structural modification of a Sparse Voxel Octree (SVO). It distinguishes itself from static grid methods by dynamically pruning free space and subdividing occupied regions to balance memory efficiency and geometric detail, ensuring no undersampling occurs during refinement.

## How it works
The method reconstructs scene models in a coarse-to-fine manner, starting with a dense, coarse scene grid (a full, shallow SVO) and gradually sparsifying or refining the structure based on optimized field values.

**Inputs and Outputs:**
*   **Input:** A set of input images with known camera parameters; an initial full, shallow SVO structure.
*   **Output:** An optimized SVO with pruned free-space nodes and subdivided detail nodes, along with optimized field values (opacity, color).

**Data Structures:**
*   **Sparse Voxel Octree (SVO):** A hierarchical grid where each node contains field data (e.g., opacity $\alpha$).
*   **Field Data:** Per-node opacity values used to determine visibility and sampling requirements.

**Algorithm Steps:**
1.  **Initialization:** Start with a dense, but coarse scene grid, i.e., a full, shallow SVO [S1, unofficial].
2.  **Field Optimization:** Optimize the SVO fields (e.g., opacity, color) for the current fixed SVO structure. The optimizer used is Adam with recommended settings [S1, unofficial].
3.  **Structure Update (Outer Loop):** After optimizing fields, the SVO structure changes infrequently. This involves two phases:
    *   **Merging Nodes (Pruning):**
        1.  Determine the set of nodes $\{N_{keep}\}$ required for rendering using Dijkstra searches on each tree level's 27-neighborhood graph [S1, unofficial].
        2.  The search is hysteresis-based for robustness [S1, unofficial].
        3.  Sample each tree node using a regular pattern of $8^3$ sampling points to determine its maximum opacity $\alpha_{max}(n)$ [S1, unofficial].
        4.  Start a Dijkstra search on each unvisited node [S1, unofficial].
        5.  The resulting set $\{N_{keep}\}$ defines nodes that must not be discarded. If any child of an SVO node is within $\{N_{keep}\}$, keep all 8 children; otherwise, discard all of them [S1, unofficial]. This merges nodes in free space to save memory and reduce rendering costs, while increasing view ray sampling point distribution quality [S1, unofficial].
    *   **Subdividing Nodes (Refinement):**
        1.  Find the set of required nodes $\{N_{sub}\}$ using the same Dijkstra search method as merging [S1, unofficial].
        2.  A leaf node within $\{N_{sub}\}$ is eligible for subdivision only if it does not induce undersampling [S1, unofficial].
        3.  Undersampling check: There must be an input pixel with a small enough 3D footprint such that the Nyquist sampling theorem holds [S1, unofficial].
        4.  The footprint $f(p)$ is defined by back-projecting the pixel to the depth $d$ of the node. The sampling rate is defined by this footprint, and the node edge length defines the signal rate [S1, unofficial].
        5.  If a leaf is within $\{N_{sub}\}$ and does not induce aliasing, allocate all of its 8 children [S1, unofficial].
        6.  Fill the new leaf nodes with 3D-interpolated data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_sparse_voxel_optimization(input_images, initial_svo, optimizer_params):
    # Initialize Adam optimizer with recommended settings [S1, unofficial]
    optimizer = Adam(initial_svo.fields, **optimizer_params)
    
    svo = initial_svo
    
    while not converged:
        # Step 1: Optimize fields for fixed structure
        for _ in optimization_steps:
            loss = compute_rendering_loss(svo, input_images)
            optimizer.step(loss)
            
        # Step 2: Update structure
        # Determine nodes to keep using hysteresis-based Dijkstra search
        nodes_to_keep = set()
        for level in svo.levels:
            # Sample each node with 8^3 points to find max opacity
            for node in level.nodes:
                alpha_max = sample_max_opacity(node, samples=8**3)
                # Hysteresis-based Dijkstra search logic [S1, unofficial]
                # ... (implementation details of Dijkstra on 27-neighborhood)
                nodes_to_keep.add(node)
                
        # Merge nodes: Keep children if any child is in nodes_to_keep
        new_svo = SVO()
        for node in svo.nodes:
            if node in nodes_to_keep or any(child in nodes_to_keep for child in node.children):
                new_svo.add_node(node)
            else:
                # Discard all children (merge node)
                pass
                
        # Subdivide nodes
        nodes_to_subdivide = set()
        for level in new_svo.levels:
            # Re-run Dijkstra to find nodes requiring detail
            # ... (same logic as above)
            nodes_to_subdivide.update(nodes_in_detail_region)
            
        for node in new_svo.leaves:
            if node in nodes_to_subdivide:
                # Check Nyquist sampling theorem
                footprint = back_project_pixel_footprint(node.depth)
                signal_rate = node.edge_length
                if not induces_undersampling(footprint, signal_rate):
                    # Allocate 8 children and fill with 3D-interpolated data
                    children = node.subdivide()
                    fill_with_interpolated_data(children)
                    new_svo.add_nodes(children)
                    
        svo = new_svo
        
    return svo
```

**Parameters:**
*   **Optimizer:** Adam [S1, unofficial].
*   **Sampling:** $8^3$ regular pattern for opacity estimation [S1, unofficial].
*   **Neighborhood:** 27-neighborhood graph for Dijkstra search [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Undersampling:** Subdivision must strictly adhere to the Nyquist sampling theorem to avoid aliasing [S1, unofficial].
*   **Precision:** Hysteresis-based search provides robustness against noise in opacity estimates [S1, unofficial].

## Subtopics

### [ERF Optimization with SVO Priors](23432-erf-optimization-with-svo-priors.md)
Iterative scene reconstruction from scratch using SGD and importance pixel sampling in a coarse-to-fine manner. It compares input images against differentiable renderings and updates the model according to model loss and gradients.

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization steps and the cost of Dijkstra searches on the 27-neighborhood graph at each level.
*   **Memory Complexity:** Reduced compared to dense grids due to merging nodes in free space [S1, unofficial].
*   **Performance:** Merging nodes increases the quality of view ray sampling point distributions [S1, unofficial].
*   **Strengths:** Memory efficiency, reduced rendering costs, high-quality sampling distribution.
*   **Limitations:** Requires careful tuning of hysteresis thresholds and subdivision criteria to avoid undersampling.

## Background
The method is described in "ERF: Explicit Radiance Field Reconstruction From Scratch" [S1, unofficial]. It builds on the concept of Sparse Voxel Octrees and uses Adam optimization [S1, unofficial].

## Sources
[S1] ERF: Explicit Radiance Field Reconstruction From Scratch, https://arxiv.org/abs/2203.00051v1, fetched 2026-10-26 (unofficial)
