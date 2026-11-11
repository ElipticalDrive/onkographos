# Generalized Trilinear Interpolation

Also called: GTI Method, Geometric Trilinear Interpolation

## Overview
Generalized Trilinear Interpolation (GTI) is a reconstruction method for cell-centered Tree-Based Adaptive Mesh Refinement (TB-AMR) data that computes field values at arbitrary points, specifically the vertices of dual cells or octants, ensuring $C^0$ continuity across refinement level boundaries [S1]. It solves the problem where standard trilinear interpolation fails because neighbors at different resolution levels are missing or misaligned, by using virtual unstructured elements to initialize octant vertices with precomputed weights derived from a multivariate polynomial basis [S1]. It builds on [Octant Method](2648-octant-method.md) by providing the vertex initialization step required before trilinear interpolation within an octant can proceed [S1].

## How it works
**Inputs:**
- A query point $\mathbf{p} = (x, y, z)$ in 3D space [S1].
- The TB-AMR hierarchy data structure containing cell-centered scalar values [S1].

**Outputs:**
- A scalar field value $F(\mathbf{p})$ [S1].

**Data Structures:**
- **Precomputed Weight Tables:**
  - **2D Cases:** 5 distinct cases for side/edge vertices. Each case stores a weight vector $\mathbf{c} = [c_A, c_B, c_C, c_D]$ of size 4 [S1].
  - **3D Corner Cases:** 20 distinct cases for corner vertices. Each case stores a weight vector $\mathbf{c}$ of size 8 [S1].
  - The weights are derived from the geometry of the "virtual unstructured element" formed by the neighboring cells at the level boundary [S1].
- **Virtual Unstructured Element:** A conceptual polygon (2D) or polyhedron (3D) formed by the centers of the neighboring AMR cells that share a vertex on the level boundary. These elements are not stored; their geometry is fixed by the 2:1 refinement ratio assumption [S1].

**Numbered Steps:**

1.  **Locate Containing Octant:**
    Determine the octant containing the query point $\mathbf{p}$ using the [Octant Method](2648-octant-method.md) logic. This identifies the 8 vertices of the octant: cell center $O_0$, side vertices $O_x, O_y, O_z$, edge vertices $O_{xy}, O_{yz}, O_{xz}$, and corner vertex $O_{xyz}$ [S1].

2.  **Initialize Octant Vertices via GTI:**
    For each vertex $V \in \{O_x, O_y, O_z, O_{xy}, O_{yz}, O_{xz}, O_{xyz}\}$ that lies on or crosses a level boundary, compute its value $I(V)$ using GTI. The cell center $O_0$ value is known directly from the AMR data [S1].
    
    *   **Determine Vertex Type and Neighbors:**
        Identify the type of vertex ($O_x$, $O_{xy}$, or $O_{xyz}$) and the refinement configuration of its neighboring cells [S1].
        
    *   **Select Case and Retrieve Weights:**
        Based on the neighbor configuration, select the corresponding precomputed case from the weight tables [S1].
        - For $O_x, O_y, O_z$: The vertex is shared by 2 or 5 cells. It is enclosed in a pyramid. The interpolation is reduced to a 2D Case 1 by interpolating along the base of the pyramid [S1].
        - For $O_{xy}, O_{yz}, O_{xz}$: The vertex is shared by 4 cells. The interpolation simplifies to 2D Cases 2–5 depending on which neighbors are refined [S1].
        - For $O_{xyz}$: The vertex is shared by 8 cells. There are 20 distinct cases modulo symmetry. Select the precomputed weight vector $\mathbf{c}$ for that case [S1].

    *   **Compute Interpolated Value:**
        Let $\mathbf{I}_{neighbors}$ be the vector of known scalar values at the centers of the neighboring cells forming the virtual unstructured element. Compute the vertex value as the dot product:
        $$ I(V) = \mathbf{c} \cdot \mathbf{I}_{neighbors} = \sum_{i} c_i I_i $$
        where $c_i$ are the precomputed weights for the selected case and $I_i$ are the field values at the neighbor cell centers [S1].

    *   **Symmetric Side Vertex Handling:**
        For side vertices like $O_x$ on a boundary, the value is computed from the coarser side octant vertex (e.g., $O'_x$) using the appropriate 2D case. The finer side vertex value is the average of the finer corner vertex ($O_{xy}$) and the coarser side vertex ($O'_x$) [S1].

3.  **Trilinear Interpolation within Octant:**
    With all 8 vertex values of the containing octant now initialized (either from direct data or GTI), perform standard trilinear interpolation to compute $F(\mathbf{p})$ for the query point $\mathbf{p}$ inside the octant [S1].

**Formulas:**

**2D Weight Derivation (Example Case 4):**
Assume a bilinear function $f(x,y) = ax + by + cxy + d$.
The interpolated value $I(x,y)$ is a weighted average of cell values $A, B, C, D$:
$$ I(x,y) = c_A A + c_B B + c_C C + c_D D = \sum_{i=A,B,C,D} c_i f(x_i, y_i) $$
Subject to the constraint that for any vertex $p \in \{A,B,C,D\}$:
$$ f(x_p, y_p) = I(x_p, y_p) $$
For Case 4, with vertices $A=(-2,-2), B=(2,-2), C=(-2,2), D=(1,1)$ and target origin at $O'_{xy}$, the system yields:
$$ c_A = 2/9, \quad c_B = 1/6, \quad c_C = 1/6, \quad c_D = 4/9 $$
[S1]

**3D General Formulation:**
The field value $f(\mathbf{x})$ is a linear combination of basis functions $\phi_i(\mathbf{x})$ with unknown coefficients $k_i$:
$$ f(\mathbf{x}) = \sum_{i=1}^N k_i \phi_i(\mathbf{x}) $$
In 3D, $N=8$, and the basis functions are:
$$ \phi_1(\mathbf{x}) = 1, \quad \phi_2(\mathbf{x}) = x, \quad \dots, \quad \phi_8(\mathbf{x}) = xyz $$
The coefficients $k_i$ are determined by solving the system $\mathbf{I} = V \mathbf{k}$, where $V$ is a multivariate Vandermonde matrix with entries $V_{ij} = \phi_j(\mathbf{x}_i)$, and $\mathbf{I}$ contains the known field values at the $N$ known neighbor points $\mathbf{x}_i$ [S1].
The final interpolated value at the target vertex $O_{xyz}$ (set to origin $\mathbf{0}$) is:
$$ \mathbf{I}(\mathbf{x}_{O_{xyz}}) = \mathbf{c} \cdot \mathbf{I} $$
where $\mathbf{c} = \mathbf{e}_1^T V^{-1}$ is the precomputed weight vector [S1].

**Parameters:**
- **Refinement Ratio:** Assumed to be 2:1. This fixes the relative positions of the virtual element vertices, allowing weights to be precomputed and independent of absolute level [S1].
- **Number of Cases:** 5 in 2D, 20 in 3D (for corner vertices) [S1].
- **Basis Functions:** 8 in 3D [S1].

**Edge Cases and Pitfalls:**
- **Standard Trilinear Failure:** Standard trilinear interpolation fails at level boundaries because not all 8 neighbors of a dual cell vertex exist or are at the same resolution. GTI handles this by using only the available neighbors to solve for coefficients [S1].
- **Precision:** Weights are precomputed numerically. Ensure consistent floating-point precision when applying weights. [G]
- **Boundary Conditions:** The method assumes the query point is within the domain of the AMR hierarchy. Points outside the coarsest level boundary require separate handling (not defined in GTI). [G]
- **Symmetry:** The 20 3D cases are modulo symmetry. Implementations must correctly map the neighbor configuration to the canonical case index. [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generalized_trilinear_interpolation(query_point, amr_hierarchy, weights_2d, weights_3d):
    """
    Reconstructs the field value at query_point using GTI.
    
    Args:
        query_point: tuple (x, y, z)
        amr_hierarchy: Object providing access to cell-centered values and tree structure.
        weights_2d: Dict mapping 2D case ID to weight vector [cA, cB, cC, cD].
        weights_3d: Dict mapping 3D case ID to weight vector [c1, ..., c8].
        
    Returns:
        float: Interpolated field value.
    """
    # Step 1: Locate the containing octant using the Octant Method logic
    octant = locate_octant(query_point, amr_hierarchy)
    
    # Step 2: Initialize the 8 vertices of the octant
    vertex_values = {}
    
    # The cell center value is known directly
    vertex_values['O0'] = amr_hierarchy.get_cell_value(octant.cell_id)
    
    # Define the 7 boundary vertices to compute
    boundary_vertices = ['Ox', 'Oy', 'Oz', 'Oxy', 'Oyz', 'Oxz', 'Oxyz']
    
    for v_name in boundary_vertices:
        # Determine the geometric location of the vertex relative to the octant
        vertex_coords = get_vertex_coords(v_name, octant)
        
        # Check if this vertex lies on a level boundary
        if is_on_level_boundary(vertex_coords, amr_hierarchy):
            # Find the neighboring cells that form the virtual unstructured element
            neighbor_cells = find_neighbor_cells_for_vertex(v_name, vertex_coords, amr_hierarchy)
            
            # Determine the refinement configuration of these neighbors
            # This returns a case ID based on which neighbors are refined
            case_id = determine_refinement_case(v_name, neighbor_cells)
            
            # Retrieve precomputed weights for this case
            if v_name in ['Ox', 'Oy', 'Oz', 'Oxy', 'Oyz', 'Oxz']:
                # These map to 2D cases (1-5)
                weights = weights_2d[case_id]
            else:
                # Oxyz maps to 3D cases (1-20)
                weights = weights_3d[case_id]
            
            # Get the cell values for the neighbors
            neighbor_values = [amr_hierarchy.get_cell_value(cell.id) for cell in neighbor_cells]
            
            # Compute vertex value as dot product
            vertex_values[v_name] = sum(w * v for w, v in zip(weights, neighbor_values))
        else:
            # Vertex is not on a boundary; value might be known or computed differently
            # [G: In a full implementation, this branch handles vertices inside a single level]
            vertex_values[v_name] = amr_hierarchy.get_vertex_value(v_name, octant)
            
    # Step 3: Perform trilinear interpolation within the octant
    x, y, z = get_normalized_coords(query_point, octant)
    return trilinear_interpolate(x, y, z, vertex_values)

def locate_octant(point, hierarchy):
    # Standard tree traversal to find the leaf cell containing point
    current = hierarchy.root
    while not current.is_leaf():
        child_idx = get_child_index(point, current)
        current = current.children[child_idx]
    return current

def get_vertex_coords(v_name, octant):
    # Returns absolute coordinates of the octant vertex
    if v_name == 'O0':
        return octant.center
    elif v_name == 'Ox':
        return (octant.max_corner[0], octant.min_corner[1], octant.min_corner[2])
    elif v_name == 'Oy':
        return (octant.min_corner[0], octant.max_corner[1], octant.min_corner[2])
    elif v_name == 'Oz':
        return (octant.min_corner[0], octant.min_corner[1], octant.max_corner[2])
    elif v_name == 'Oxy':
        return (octant.max_corner[0], octant.max_corner[1], octant.min_corner[2])
    elif v_name == 'Oyz':
        return (octant.min_corner[0], octant.max_corner[1], octant.max_corner[2])
    elif v_name == 'Oxz':
        return (octant.max_corner[0], octant.min_corner[1], octant.max_corner[2])
    elif v_name == 'Oxyz':
        return octant.max_corner
    return None

def is_on_level_boundary(coords, hierarchy):
    # Checks if the coordinate lies on a refinement boundary
    return hierarchy.check_level_boundary(coords)

def find_neighbor_cells_for_vertex(v_name, coords, hierarchy):
    # Traverses the tree to find leaf cells sharing the vertex
    return hierarchy.find_leaf_cells_sharing_vertex(coords)

def determine_refinement_case(v_name, neighbor_cells):
    # Maps the refinement levels of neighbor cells to a canonical case ID
    levels = [cell.level for cell in neighbor_cells]
    return map_to_canonical_case(v_name, levels)

def get_normalized_coords(point, octant):
    # Returns coordinates normalized to [0, 1] within the octant
    dx = octant.max_corner[0] - octant.min_corner[0]
    dy = octant.max_corner[1] - octant.min_corner[1]
    dz = octant.max_corner[2] - octant.min_corner[2]
    x = (point[0] - octant.min_corner[0]) / dx
    y = (point[1] - octant.min_corner[1]) / dy
    z = (point[2] - octant.min_corner[2]) / dz
    return x, y, z

def trilinear_interpolate(x, y, z, vertex_values):
    # Standard trilinear interpolation formula
    v000 = vertex_values['O0']
    v100 = vertex_values['Ox']
    v010 = vertex_values['Oy']
    v001 = vertex_values['Oz']
    v110 = vertex_values['Oxy']
    v101 = vertex_values['Oxz']
    v011 = vertex_values['Oyz']
    v111 = vertex_values['Oxyz']
    
    t0 = v000 * (1-x) + v100 * x
    t1 = v010 * (1-x) + v110 * x
    t2 = v001 * (1-x) + v101 * x
    t3 = v011 * (1-x) + v111 * x
    
    b0 = t0 * (1-y) + t1 * y
    b1 = t2 * (1-y) + t3 * y
    
    return b0 * (1-z) + b1 * z
```

## Subtopics

### Optimized Octant Generation
A sibling method that focuses on the efficient generation of the octant structure itself, rather than the interpolation weights at the boundaries. Choose this when the bottleneck is octant construction rather than sampling. [Optimized Octant Generation](3412-optimized-octant-generation.md)

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample point after octant location. The octant location is $O(\log N)$ where $N$ is the number of cells, but typically fast due to tree traversal. The GTI step involves a fixed number of neighbor lookups and a dot product with 4 or 8 weights.
- **Memory Complexity:** $O(1)$ additional memory for precomputed weights (5 vectors of size 4, 20 vectors of size 8). The main memory cost is the AMR data structure itself.
- **Measured Performance:** In the ExaJet dataset (656M cells), the builder constructed the octree representation in 160s, indicating that the GTI weight computation (precomputed) is not a bottleneck relative to file I/O [S1].
- **Strengths:** Ensures $C^0$ continuity across level boundaries; avoids artifacts and cracks; adapts to AMR mesh resolution; enables artifact-free visualization [S1].
- **Limitations:** Requires precomputation of weights for all 20 3D cases; assumes 2:1 refinement ratio; more complex to implement than standard trilinear interpolation.
- **When to use:** When visualizing TB-AMR data with level boundaries and continuity is critical.
- **When not to use:** When the refinement ratio is not 2:1 (weights would need recomputation); when $C^0$ continuity is not required and simpler methods suffice.

## Background
GTI was introduced in 2020 by Wan et al. in the paper "CPU Ray Tracing of Tree-Based Adaptive Mesh Refinement Data" published in the Computer Graphics Forum. It builds on prior work such as [ME11, WCM12, WBUK17, WWW*19] which proposed various approaches for stitching AMR data, but GTI specifically ensures $C^0$ continuity and adapts to the AMR mesh resolution to avoid artifacts. It is used in the OSPRay volume rendering framework for both Direct Volume Rendering (DVR) and Implicit Isosurface Ray Tracing [S1].

## Sources
[S1] CPU Ray Tracing of Tree‐Based Adaptive Mesh Refinement Data, https://publications.sci.utah.edu/publications/Wan2020a/cgf.13958.pdf, fetched 2026-10-27
