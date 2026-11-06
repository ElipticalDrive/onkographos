# Kdtree-based Iso-surface Extraction

Also called: Kdtree extraction, KdtreeISO

## Overview
Kdtree-based Iso-surface Extraction is a method for extracting iso-surfaces from scalar fields using a hierarchical K-d tree structure built upon an octree, utilizing Quadratic Error Function (QEF) for vertex optimization and clusterability checks for simplification [S1, unofficial]. It differs from its sibling [Adaptive Dual Contouring](3664-adaptive-dual-contouring.md) by explicitly constructing a K-d tree to guide the traversal and subdivision, whereas the sibling relies on adaptive octree refinement without the explicit K-d tree hierarchy for mesh generation. It builds on [Dual Contouring with Octree](1271-dual-contouring-with-octree.md) by replacing the standard octree extraction path with a K-d tree variant when the 'kd' mode is active.

## How it works
**Inputs and Outputs**
*   **Input:** A scalar field, a maximum octree depth (`octDepth`), a base octree size unit (`octSize`), an iso-surface threshold (`threshold`), and a mode flag (`mode`) which is either `'kd'` or `'oct'` [S1, unofficial].
*   **Output:** A triangular mesh representing the iso-surface [S1, unofficial].

**Data Structures**
The core data structure is the `Kdtree` node [S1, unofficial]:
*   `grid`: A `RectilinearGrid` object storing vertex positions, normals, and QEF coefficients.
*   `planeDir`: An integer indicating the splitting plane direction (0, 1, or 2 for x, y, z).
*   `depth`: The current depth in the K-d tree.
*   `clusterable`: A boolean flag (default `true`) indicating if the node's vertices can be merged.
*   `children`: An array of two `Kdtree` pointers (`children[0]` and `children[1]`), initialized to `nullptr`.
*   The `RectilinearGrid` contains:
    *   `vertices`: A list of vertex positions.
    *   `components`: Data structures for solving QEF components.
    *   `approximate`: The optimized vertex position.

**Parameters**
*   `octDepth`: Maximum depth of the underlying octree, typically 8 [S1, unofficial].
*   `octSize`: Base size unit for the octree, typically 16 [S1, unofficial].
*   `threshold`: The iso-surface value threshold, read from parameters as a float [S1, unofficial].

**Algorithm Steps**

1.  **Build Octree:** Construct an initial octree from the scalar field using `Octree::buildWithScalarField`. Pass the minimum code (`-sizeCode / 2`), `octDepth`, the scalar field pointer, and a boolean flag indicating if KD-tree mode is active (`parameters['s'] == 'kd'`) [S1, unofficial].
    *   *Pitfall:* If no sign change is found during octree build, the program exits with an error [S1, unofficial].

2.  **Build K-d Tree (if mode == 'kd'):** If the mode is `'kd'`, build the K-d tree from the octree using `Kdtree::buildFromOctree`. Pass the octree, minimum code (`-sizeCode / 2`), maximum code (`sizeCode / 2`), the scalar field, and a parameter (0) [S1, unofficial].
    *   The `buildFromOctree` function recursively splits the octree.
    *   It calculates the approximate error for the current grid. If the error is less than the threshold, it returns the current node as a leaf.
    *   Otherwise, it determines the splitting axis using `chooseAxisDir`.
    *   It creates a new `Kdtree` node with the QEF sum, min/max codes, direction, and depth.
    *   It recursively builds `children[0]` and `children[1]` by splitting the octree bounds.
    *   For non-leaf nodes, it assigns signs, calculates clusterability (`calClusterability`), and combines QEFs (`combineQef`).
    *   If `clusterable` is true, it solves the QEF for each component (`solveComponent`) [S1, unofficial].

3.  **Extract Mesh (if mode == 'kd'):** Call `Kdtree::extractMesh` with the K-d tree root, scalar field, and threshold [S1, unofficial].
    *   This function initializes a new `Mesh` object.
    *   It calls `generateVertexIndices` to add vertices to the mesh based on the grid's approximate positions.
    *   It calls `contourCell` to generate the triangles.

4.  **Simplify and Extract (if mode == 'oct'):** If the mode is `'oct'`, simplify the octree using `Octree::simplify` with the threshold [S1, unofficial].
    *   Then extract the mesh using `Octree::extractMesh`, passing the octree, scalar field, and references to `intersectionPreservingVerticesCount` and `intersectionFree` [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kdtree_iso_extraction(scalar_field, oct_depth=8, oct_size=16, threshold=0.0, mode='kd'):
    """
    Main entry point for Kdtree-based Iso-surface Extraction.
    """
    # Step 1: Build Octree
    min_code = PositionCode(-oct_size / 2)
    max_code = PositionCode(oct_size / 2)
    octree = Octree.build_with_scalar_field(scalar_field, min_code, oct_depth, threshold, mode == 'kd')
    
    if octree is None:
        raise Error("No sign change found during octree build")

    if mode == 'kd':
        # Step 2: Build K-d Tree
        kdtree_root = Kdtree.build_from_octree(octree, min_code, max_code, scalar_field, 0)
        
        # Step 3: Extract Mesh
        mesh = Kdtree.extract_mesh(kdtree_root, scalar_field, threshold)
        return mesh
    else:
        # Step 4: Oct Mode
        Octree.simplify(octree, threshold)
        intersection_preserving_vertices_count = 0
        intersection_free = False
        mesh = Octree.extract_mesh(octree, scalar_field, intersection_preserving_vertices_count, intersection_free)
        return mesh

class Kdtree:
    def __init__(self, qef_solver, min_code, max_code, plane_dir, depth):
        self.grid = RectilinearGrid()
        self.plane_dir = plane_dir
        self.depth = depth
        self.clusterable = True
        self.children = [None, None]
        self.qef_solver = qef_solver

    @staticmethod
    def build_from_octree(octree, min_code, max_code, scalar_field, depth):
        # Calculate approximate error
        qef = QefSolver()
        # ... (Assume qef is populated from octree nodes) ...
        
        # Check if leaf
        if qef.get_approximate_error() < threshold:
            node = Kdtree(qef, min_code, max_code, 0, depth)
            node.grid.assign_sign(threshold)
            node.grid.sample_qef(threshold, False)
            return node

        # Choose splitting axis
        plane_dir = Kdtree.choose_axis_dir(qef, min_code, max_code)
        
        # Split bounds
        mid_code = (min_code + max_code) / 2
        # Adjust mid_code based on plane_dir
        mid_code[plane_dir] = (min_code[plane_dir] + max_code[plane_dir]) / 2
        
        # Recursively build children
        # Note: Exact splitting logic for octree bounds is implementation-specific
        # Assuming helper functions to split octree bounds
        left_max_code = max_code
        left_max_code[plane_dir] = mid_code[plane_dir]
        right_min_code = min_code
        right_min_code[plane_dir] = mid_code[plane_dir]
        
        node = Kdtree(qef, min_code, max_code, plane_dir, depth)
        node.children[0] = Kdtree.build_from_octree(octree, min_code, left_max_code, scalar_field, depth + 1)
        node.children[1] = Kdtree.build_from_octree(octree, right_min_code, max_code, scalar_field, depth + 1)
        
        node.grid.assign_sign(threshold)
        node.cal_clusterability(threshold)
        node.combine_qef()
        
        if node.clusterable:
            for i in range(node.grid.components.size()):
                node.grid.solve_component(i)
                
        return node

    @staticmethod
    def choose_axis_dir(qef, min_code, max_code):
        size = max_code - min_code
        # Variance approach
        approximate, error = qef.solve()
        variance = qef.get_variance(approximate)
        variance[0] *= size[0]
        variance[1] *= size[1]
        variance[2] *= size[2]
        
        max_var_dir = 0
        min_var_dir = 1
        if variance[1] > variance[0]:
            max_var_dir = 1
            min_var_dir = 0
        if variance[2] > variance[max_var_dir]:
            max_var_dir = 2
        if variance[min_var_dir] > variance[2]:
            min_var_dir = 2
            
        dir = max_var_dir
        if size[max_var_dir] < 2:
            dir = 3 - max_var_dir - min_var_dir
            if size[3 - max_var_dir - min_var_dir] < 2:
                dir = min_var_dir
        return dir

    @staticmethod
    def extract_mesh(root, scalar_field, threshold):
        mesh = Mesh()
        Kdtree.generate_vertex_indices(root, mesh, scalar_field, threshold)
        Kdtree.contour_cell(root, mesh, scalar_field, threshold)
        return mesh

    @staticmethod
    def generate_vertex_indices(root, mesh, scalar_field, threshold):
        if root is None:
            return
        # Add vertices from grid
        for v in root.grid.vertices:
            mesh.add_vertex(v.position, scalar_field)
        # Recurse
        if root.children[0]:
            Kdtree.generate_vertex_indices(root.children[0], mesh, scalar_field, threshold)
        if root.children[1]:
            Kdtree.generate_vertex_indices(root.children[1], mesh, scalar_field, threshold)

    @staticmethod
    def contour_cell(root, mesh, scalar_field, threshold):
        if root is None:
            return
        if root.is_contouring_leaf(threshold):
            root.grid.draw(mesh)
            return
        Kdtree.contour_cell(root.children[0], mesh, scalar_field, threshold)
        Kdtree.contour_cell(root.children[1], mesh, scalar_field, threshold)
```

## Subtopics

### Octree Mode
Uses standard octree simplification and extraction. Choose when K-d tree overhead is not desired or when the scalar field structure aligns better with axis-aligned octree splits. See [Dual Contouring with Octree](1271-dual-contouring-with-octree.md).

### K-d Tree Mode
Uses the K-d tree for adaptive splitting based on QEF variance. Choose when higher fidelity in complex regions is needed and the variance-based splitting provides better adaptation than uniform octree depth.

## Operating characteristics
*   **Time Complexity:** Building the K-d tree is $O(N \log N)$ where $N$ is the number of octree nodes, due to recursive splitting and QEF solving. Extraction is linear in the number of leaves.
*   **Memory Complexity:** $O(N)$ for storing the K-d tree nodes and their associated grids.
*   **Strengths:** Adaptive resolution based on geometric complexity (QEF error) rather than just scalar field variation. The K-d tree allows for efficient traversal and splitting along the direction of maximum variance.
*   **Limitations:** Higher memory overhead due to K-d tree structure. More complex implementation than standard octree extraction.
*   **When to use:** When extracting iso-surfaces from scalar fields with high geometric complexity where uniform octree refinement is inefficient.
*   **When not to use:** When memory is constrained or when the scalar field is smooth and uniform octree refinement is sufficient.

## Background
The method is described in the GitHub repository [Danielmelody/KdtreeISO](https://github.com/Danielmelody/KdtreeISO) [S1, unofficial]. It extends the concept of Dual Contouring with Octree by introducing a K-d tree structure to guide the subdivision process, aiming to improve the quality of the extracted mesh in complex regions.

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
