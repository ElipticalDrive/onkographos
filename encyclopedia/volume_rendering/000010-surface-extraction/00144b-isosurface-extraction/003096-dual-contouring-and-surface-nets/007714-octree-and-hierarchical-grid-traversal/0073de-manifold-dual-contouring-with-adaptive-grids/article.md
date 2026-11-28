# Manifold Dual Contouring with Adaptive Grids

Also called: K-d Tree Based Dual Contouring, k-d tree-based manifold dual contouring, octree-based manifold dual contouring

## Overview
Manifold Dual Contouring with Adaptive Grids is an isosurface extraction method that uses adaptive data structures (k-d trees or octrees) to perform error-controlled simplification while preserving topology via manifold vertex clustering and Quadratic Error Function (QEF) minimization [S1]. It sets itself apart from standard Dual Contouring by integrating adaptive spatial partitioning directly into the contouring process, allowing for more compact implicit representations, particularly for thin object structures, and ensuring the generation of two-manifold meshes even when cells contain multiple surface components [S1].

## How it works
The method operates on a discretized domain using either a k-d tree or an octree hierarchy to represent the implicit object [S1]. It builds on the shared principles of Dual Contouring and Surface Nets [6319-dual-contouring-and-surface-nets.md], specifically placing a single vertex inside each grid cell intersecting the isosurface and determining its position by optimizing a QEF using Hermite data [G].

### Inputs and Outputs
*   **Input:** An implicit scalar field defined on a rectilinear grid, where edges are tagged with Hermite data (exact intersection points and normals) [S1].
*   **Output:** A two-manifold triangular mesh representing the isosurface [S1].

### Data Structures
The core data structure is an adaptive hierarchy, either a k-d tree or an octree [S1].
*   **k-d tree:** A binary tree where each node represents a region of space, split along one axis. It is noted to be superior to octrees in adapting to object surfaces and obtaining compact representations for thin structures [S1].
*   **Octree:** A tree where each node has up to eight children, representing an octant of space. It follows the standard process of building an octree hierarchy as described by Schaefer et al. [S1].
*   **Hermite Data:** Stored at grid vertices or edges, containing the intersection point and the surface normal [S1].

### Steps
The algorithm execution is divided into two main phases [S1]:

**Phase 1: Construction of the underlying rectilinear grid**
1.  **Formulation of QEFs:** Construct the Quadratic Error Function for each cell. The QEF formulation is adopted from Ju et al. [20], specifically Equations (1) and (3) from Subsection 2.1.2 of that source [S1]. These equations determine vertex locations and associated error based on Hermite data [S1].
2.  **Generation of the scalar field:** Generate the scalar field of the implicit surface representation [S1].
3.  **Composition of vertex data with Hermite data:** Associate Hermite data (intersection points and normals) with the grid vertices [S1].
4.  **Preparation of the manifold criterion topology preserving scheme:** Prepare the scheme that ensures the resulting mesh is a two-manifold, handling cases where cells might contain multiple surface components [S1].
5.  **Construction of the general mesh:** Construct the initial general mesh in accordance with the topology preserving scheme [S1].

**Phase 2: Isosurface extraction of implicit models through error-controlled simplification**
1.  **Generation of the scalar field:** Re-generate or access the scalar field of the implicit surface representation for the adaptive grid [S1].
2.  **Preparation of the computation of topology preserving vertex clustering using QEFs:** Prepare the clustering logic that uses QEFs to merge vertices while preserving topology [S1].
3.  **QEF controlled isosurface simplification:** Perform the simplification on the implicit k-d tree- or octree-based adaptive grids. This step uses the QEF to minimize error while applying the manifold criterion to simplify the surface [S1].

### Pseudocode
The following pseudocode outlines the high-level flow described in the sources. Note that specific QEF matrix assembly and solver details are referenced from Ju et al. [20] [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def manifold_dual_contouring_adaptive_grid(implicit_field, grid_resolution, tree_type='kdtree'):
    """
    Performs manifold dual contouring using an adaptive grid structure.
    
    Args:
        implicit_field: Function or data structure representing the scalar field f(x,y,z).
        grid_resolution: Initial resolution of the rectilinear grid.
        tree_type: 'kdtree' or 'octree'.
        
    Returns:
        mesh: A two-manifold triangular mesh.
    """
    
    # Phase 1: Construction of the underlying rectilinear grid
    
    # 1.1 Formulation of QEFs
    # Initialize QEF structures for each cell. 
    # QEF is based on Ju et al. [20] Equations (1) and (3) [S1].
    qef_structures = initialize_qef_structures(grid_resolution)
    
    # 1.2 Generation of the scalar field
    # Evaluate the implicit field on the grid vertices
    scalar_values = evaluate_scalar_field(implicit_field, grid_resolution)
    
    # 1.3 Composition of vertex data with Hermite data
    # Compute intersection points and normals for edges crossing the isosurface
    hermite_data = compute_hermite_data(scalar_values, implicit_field, grid_resolution)
    
    # 1.4 Preparation of the manifold criterion topology preserving scheme
    # Define rules for vertex clustering to ensure manifold output
    manifold_criterion = prepare_manifold_criterion()
    
    # 1.5 Construction of the general mesh
    # Create initial mesh structure based on the topology preserving scheme
    general_mesh = construct_general_mesh(grid_resolution, hermite_data, manifold_criterion)
    
    # Build the adaptive hierarchy (k-d tree or octree)
    if tree_type == 'kdtree':
        adaptive_grid = build_kdtree(general_mesh, implicit_field)
    elif tree_type == 'octree':
        adaptive_grid = build_octree(general_mesh, implicit_field)
    else:
        raise ValueError("Unsupported tree type")
        
    # Phase 2: Isosurface extraction of implicit models through error-controlled simplification
    
    # 2.1 Generation of the scalar field of the implicit surface representation
    # (Re-evaluate or access scalar field within the adaptive grid context)
    adaptive_scalar_field = generate_adaptive_scalar_field(adaptive_grid, implicit_field)
    
    # 2.2 Preparation of the computation of topology preserving vertex clustering using QEFs
    # Prepare clustering logic that respects the manifold criterion and uses QEFs
    clustering_logic = prepare_clustering_logic(adaptive_grid, qef_structures, manifold_criterion)
    
    # 2.3 QEF controlled isosurface simplification on implicit k-d tree- or octree-based adaptive grids
    mesh = perform_simplification(adaptive_grid, clustering_logic, adaptive_scalar_field)
    
    return mesh

def initialize_qef_structures(resolution):
    """
    Initializes QEF structures for each cell in the grid.
    Based on Ju et al. [20] Equations (1) and (3) [S1].
    """
    # Not given in the sources. 
    # Implementation would involve creating a QEF object for each grid cell 
    # that can accumulate Hermite constraints.
    pass

def evaluate_scalar_field(implicit_field, resolution):
    """
    Evaluates the scalar field at all grid vertices.
    """
    # Not given in the sources.
    pass

def compute_hermite_data(scalar_values, implicit_field, resolution):
    """
    Computes Hermite data (intersection points and normals) for edges 
    crossing the isosurface.
    """
    # Not given in the sources.
    pass

def prepare_manifold_criterion():
    """
    Prepares the topology preserving scheme to ensure manifold output.
    """
    # Not given in the sources.
    pass

def construct_general_mesh(resolution, hermite_data, manifold_criterion):
    """
    Constructs the initial general mesh.
    """
    # Not given in the sources.
    pass

def build_kdtree(mesh, implicit_field):
    """
    Builds a k-d tree hierarchy from the mesh.
    """
    # Not given in the sources.
    pass

def build_octree(mesh, implicit_field):
    """
    Builds an octree hierarchy from the mesh.
    """
    # Not given in the sources.
    pass

def generate_adaptive_scalar_field(adaptive_grid, implicit_field):
    """
    Generates the scalar field within the adaptive grid context.
    """
    # Not given in the sources.
    pass

def prepare_clustering_logic(adaptive_grid, qef_structures, manifold_criterion):
    """
    Prepares the vertex clustering logic using QEFs and manifold criteria.
    """
    # Not given in the sources.
    pass

def perform_simplification(adaptive_grid, clustering_logic, adaptive_scalar_field):
    """
    Performs QEF controlled simplification.
    """
    # Not given in the sources.
    pass
```

### Parameters
*   **Rectilinear Grids:** Used to represent the extraction of implicit models. They offer negligible memory footprint and support smooth data reconstruction but have reduced geometric flexibility [S1].
*   **Tree Type:** Can be 'k-d tree' or 'octree'. The k-d tree is preferred for thin object structures due to better adaptation to the object surface [S1].

### Edge Cases and Pitfalls
*   **Multiple Surface Components:** The method generates two-manifold meshes even for k-d trees with cells containing multiple surface components [S1].
*   **Flat Surfaces:** Using the implementation by Greß and Klein [17] (uniform grids with topology preserving scheme) can result in generating too many polygons, particularly on relatively flat surfaces, making rendering difficult [S1].
*   **Precision:** The method uses a numerically stable representation for QEFs [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Rectilinear grids offer a negligible memory footprint [S1].
*   **Measured Performance:** The k-d tree-based hierarchy is superior to the octree in terms of adapting to the object surface, obtaining more compact implicit representations especially in case of thin object structures [S1].
*   **Strengths:** Generates two-manifold meshes; handles cells with multiple surface components; compact representation for thin structures (k-d tree); no crack patching required (octree variant) [S1].
*   **Limitations:** Reduced geometric flexibility of rectilinear grids; potential for excessive polygons on flat surfaces if not simplified properly [S1].
*   **When to use:** When a compact implicit representation is needed, especially for thin structures, or when manifold output is required without crack patching.
*   **When not to use:** When high geometric flexibility of the underlying grid is required, or when rendering performance is critical on flat surfaces without further simplification.

## Background
The method was proposed by Greß and Klein [17], who asserted that the k-d tree-based hierarchy was superior to the octree in terms of adapting to the object surface [S1]. They described a new isosurface extraction algorithm based on k-d-trees instead of octrees in the original dual contouring algorithm by Ju et al. [20] [S1]. Ju et al. developed a method for contouring a signed grid whose edges were tagged by Hermite data, using a numerically stable representation for QEFs and developing an octree-based contouring method that imposed no constraints on the octree and did not require crack patching [S1]. The octree algorithm implementation follows the standard process of building an octree hierarchy and performing dual contouring with manifold vertex clustering as described by Schaefer et al. [40] [S1].

## Sources
[S1] Evaluation of Manifold Dual Contouring Algorithms Based on K-d Tree and Octree Data Structures, https://doi.org/10.31449/inf.v48i3.5420, fetched 2026-10-26
