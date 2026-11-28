# Voxelization of Delaunay Cover

Also called: CVT Voxelization, Centroidal Voronoi Tessellation (CVT) for 3D Modeling

## Overview
This method generates a volumetric representation by discretizing 3D space using a Centroidal Voronoi Tessellation (CVT) derived from a set of input points, often used for surface reconstruction and neural implicit field optimization [S2, unofficial]. Unlike uniform grid voxelization, it provides a regular, optimized, and manifold-preserving discretization that adapts density to the geometry [S2, unofficial]. It is particularly advantageous for surface extraction tasks, where CVT-based algorithms significantly outperform Marching Cubes and standard Delaunay Tessellation in accuracy and efficiency [S2, unofficial].

## How it works
**Inputs:**
- A set of $K$ sites (points) $P = \{p_1, p_2, \dots, p_K\}$ in $\mathbb{R}^3$ [S2, unofficial].
- A bounding domain for the tessellation [S2, unofficial].

**Outputs:**
- A set of Voronoi cells $\{V_1, V_2, \dots, V_K\}$ defining the voxel structure [S2, unofficial].
- Optimized site locations that correspond to the centroids of their respective cells [S2, unofficial].

**Data Structures:**
- **Sites:** A list of 3D coordinates.
- **Voronoi Cells:** Implicitly defined by the bisectors between site pairs. Explicit cell geometry is not required for the optimization step [S2, unofficial].
- **Sparse Hash Table:** Used to store voxel occupancy if a discrete grid representation is needed, mapping voxel indices to boolean or density values [S1, unofficial].

**Steps:**

1. **Initialize Voronoi Tessellation:**
   Given the initial set of sites $P$, partition the space into Voronoi cells. A Voronoi cell $V_i$ associated with site $x_i$ contains all points $p$ closer to $x_i$ than to any other site $x_j$:
   $$ V_i = \{p \in \mathbb{R}^3 \mid \|p - x_i\| < \|p - x_j\|, \forall j \in [1, K], j \neq i\} $$
   [S2, unofficial]

2. **Optimize Sites (CVT Iteration):**
   The site locations are optimized to form a Centroidal Voronoi Tessellation. This is achieved using an approximated CVT algorithm that does not explicitly compute the Voronoi cells but instead considers the neighboring bisectors [S2, unofficial]. The process iteratively moves each site to the centroid of its associated Voronoi cell, minimizing the quantization error (variance) [S2, unofficial].

3. **Discretization (Optional):**
   If a discrete voxel grid is required, map the optimized sites to a grid. For a site $p_i = (x_i, y_i, z_i)$, the corresponding voxel index in a matrix $V$ is calculated as:
   $$ V\left(\left\lfloor \frac{x_i - x_{min}}{h} \right\rfloor, \left\lfloor \frac{y_i - y_{min}}{h} \right\rfloor, \left\lfloor \frac{z_i - z_{min}}{h} \right\rfloor\right) = 1 $$
   where $h$ is the voxel size and $(x_{min}, y_{min}, z_{min})$ is the bounding box origin [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cvt_voxelization(sites, iterations=10):
    """
    Performs CVT optimization on a set of 3D sites.
    Returns optimized sites forming a Centroidal Voronoi Tessellation.
    """
    current_sites = list(sites)
    
    for _ in range(iterations):
        # Step 1: Compute Voronoi cells implicitly via bisectors
        # Note: Explicit cell computation is skipped for efficiency [S2, unofficial]
        # Instead, we estimate the centroid of each cell using 
        # sampling or bisector intersection logic.
        
        new_sites = []
        for i, site in enumerate(current_sites):
            # Determine the region associated with site i
            # This involves finding the intersection of half-spaces defined by
            # bisectors between site i and all other sites j
            cell_region = compute_voronoi_region(current_sites, i)
            
            # Compute the centroid of the cell region
            centroid = compute_centroid(cell_region)
            new_sites.append(centroid)
            
        current_sites = new_sites
        
    return current_sites

def compute_voronoi_region(sites, target_index):
    """
    Computes the geometric region (polyhedron) for a specific site.
    In practice, this is often approximated or handled via bisector logic [S2, unofficial].
    """
    # Implementation detail: Intersect half-spaces defined by bisectors
    # between target site and all other sites.
    # This returns a convex polyhedron.
    pass

def compute_centroid(polyhedron):
    """
    Computes the geometric centroid of a convex polyhedron.
    """
    # Standard geometric calculation for polyhedron centroid
    pass
```

**Edge Cases and Pitfalls:**
- **Memory Consumption:** A classical 3D matrix representation grows as the cube of the number of cells, causing memory issues [S1, unofficial]. Solution: Use a sparse hash table storing only nonzero entries [S1, unofficial].
- **Manifold Guarantee:** CVTs are guaranteed to be manifold, ensuring topological consistency [S2, unofficial].
- **Surface Extraction:** When extracting surfaces, clipping cells by edge intersections or refining by adding vertices at bisector intersections yields better results than Marching Cubes [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the CVT optimization algorithm. The approximated method avoids explicit Voronoi cell identification, reducing overhead [S2, unofficial].
- **Memory Complexity:** Sparse hash table representation is recommended to avoid cubic growth of memory usage [S1, unofficial].
- **Strengths:** Regular and optimized discretization; guaranteed manifold structure; superior surface extraction accuracy compared to Marching Cubes [S2, unofficial].
- **Limitations:** Requires iterative optimization; complex to implement explicit Voronoi cell geometry if needed [S2, unofficial].
- **When to use:** Neural adaptive 3D surface reconstruction, SDF field discretization, and scenarios requiring high-quality manifold meshes [S2, unofficial].

## Background
Centroidal Voronoi Tessellations (CVTs) are used in a wide range of applications in Visual Computing and beyond as an elegant tool for regular and optimized discretization of 3D space [S2, unofficial]. The method leverages the duality between Voronoi tessellations and Delaunay triangulations [S2, unofficial]. Recent work has demonstrated that CVT-based surface extraction algorithms significantly outperform traditional methods like Marching Cubes and Delaunay Tessellation [S2, unofficial]. Efficient ray-marching algorithms also exist for cell-based 3D structures, further enhancing the utility of CVT in volume rendering contexts [S2, unofficial].

## Sources
[S1] , https://www.scitepress.org/PublishedPapers/2007/20798/pdf/index.html, fetched 2026-10-27 (unofficial)
[S2] WACV 2025 Open Access Repository, https://openaccess.thecvf.com/content/WACV2025/html/Thomas_VortSDF_3D_Modeling_with_Centroidal_Voronoi_Tesselation_on_Signed_Distance_WACV_2025_paper.html, fetched 2026-10-30 (unofficial)
