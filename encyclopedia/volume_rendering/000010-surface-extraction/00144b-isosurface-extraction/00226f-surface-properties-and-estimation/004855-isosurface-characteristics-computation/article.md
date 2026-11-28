# Isosurface Characteristics Computation

Also called: Integral of Gradient, Mean Gradient, Total Area, Confined Volume, Curvature, Pekar et al. Isosurface Characteristics

## Overview
Isosurface Characteristics Computation quantifies the geometric and topological properties of an extracted isosurface by calculating the integral of the gradient along the surface, the mean gradient, the total area, the confined volume, and local curvature measures [S1, unofficial]. It sets itself apart from sibling methods like [Marching Cubes Surface Normal Estimation](1005-marching-cubes-surface-normal-estimation.md) and [Marching Cubes Surface Area Estimation](1315-marching-cubes-surface-area-estimation.md) by providing a unified framework for multiple scalar metrics derived from the underlying scalar field's gradient, rather than focusing on a single geometric primitive or post-processing step [G].

## How it works
The method operates on a 3D scalar field $V$ and an extracted isosurface mesh $S$ defined by $V(\mathbf{x}) = c$, where $c$ is the isovalue [G]. It computes five key characteristics:

1.  **Integral of the Gradient**: The surface integral of the magnitude of the gradient vector field over the isosurface [S1, unofficial].
2.  **Mean Gradient**: The average magnitude of the gradient over the surface [S1, unofficial].
3.  **Total Area**: The total surface area of the isosurface [S1, unofficial].
4.  **Confined Volume**: The volume enclosed by the isosurface [S1, unofficial].
5.  **Curvature**: Local curvature measures (mean and Gaussian) at each point on the surface [S1, unofficial].

### Data Structures
*   **Scalar Field $V$**: A 3D grid of values $V_{i,j,k}$ [G].
*   **Isosurface Mesh $S$**: A set of triangles $\{T_k\}$ with vertices $\mathbf{v}_i$ [G].
*   **Gradient Field $\nabla V$**: A 3D grid storing the partial derivatives $\frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z}$ [G].

### Steps

1.  **Compute Gradient Field**: Calculate $\nabla V(\mathbf{x})$ for all points in the domain using central differences [G]. For a regular grid:
    $$ \frac{\partial V}{\partial x} \approx \frac{V_{i+1,j,k} - V_{i-1,j,k}}{2\Delta x} $$
    [G] Handle boundaries by using one-sided differences or clamping [G].

2.  **Extract Isosurface**: Generate the mesh $S$ using an extraction algorithm (e.g., Marching Cubes) for isovalue $c$ [G].

3.  **Compute Total Area ($A$)**: Sum the areas of all triangles $T_k$ in $S$ [S1, unofficial].
    $$ A = \sum_{k} \text{Area}(T_k) $$
    [G] For a triangle with vertices $\mathbf{a}, \mathbf{b}, \mathbf{c}$:
    $$ \text{Area}(T_k) = \frac{1}{2} \| (\mathbf{b} - \mathbf{a}) \times (\mathbf{c} - \mathbf{a}) \| $$
    [G]

4.  **Compute Confined Volume ($Vol$)**: Calculate the volume enclosed by $S$ using the divergence theorem [S1, unofficial].
    $$ Vol = \frac{1}{6} \sum_{k} (\mathbf{a}_k \times \mathbf{b}_k + \mathbf{b}_k \times \mathbf{c}_k + \mathbf{c}_k \times \mathbf{a}_k) \cdot \mathbf{p}_0 $$
    [G] Use a fixed reference point $\mathbf{p}_0$ (e.g., the origin) and ensure consistent vertex winding order for signed volume [G].

5.  **Compute Integral of Gradient ($I_\nabla$)**: Integrate the magnitude of the gradient over the surface [S1, unofficial].
    $$ I_\nabla = \int_S \| \nabla V(\mathbf{x}) \| \, dA $$
    [G] Approximate by summing over triangles:
    $$ I_\nabla \approx \sum_{k} \text{Area}(T_k) \cdot \| \nabla V(\mathbf{p}_k) \| $$
    [G] where $\mathbf{p}_k$ is the centroid of triangle $T_k$. Interpolate $\nabla V$ at $\mathbf{p}_k$ from the grid using trilinear interpolation [G].

6.  **Compute Mean Gradient ($\bar{g}$)**: Average gradient magnitude over the surface [S1, unofficial].
    $$ \bar{g} = \frac{1}{A} \int_S \| \nabla V(\mathbf{x}) \| \, dA = \frac{I_\nabla}{A} $$
    [G]

7.  **Compute Curvature**: For each vertex or triangle, compute mean curvature $H$ and Gaussian curvature $K$ [S1, unofficial].
    *   **Mean Curvature**: $H = \frac{1}{2}(\kappa_1 + \kappa_2)$
    *   **Gaussian Curvature**: $K = \kappa_1 \kappa_2$
    [G] Use discrete differential geometry operators (e.g., cotangent weights) for mesh-based estimation [G].

### Parameters
*   **Isovalue $c$**: The threshold for extraction [G].
*   **Grid Resolution**: Affects gradient accuracy and mesh detail [G].

### Edge Cases and Pitfalls
*   **Noise**: High-frequency noise in $V$ leads to erratic gradients and curvature [G].
*   **Boundary Effects**: Gradient computation at domain boundaries requires careful handling [G].
*   **Degenerate Triangles**: Zero-area triangles in the mesh can skew area and curvature calculations [G].
*   **Non-Manifold Geometry**: Ensure the mesh is manifold for accurate volume and area integration [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_isosurface_characteristics(V, grid_spacing, mesh):
    """
    V: 3D list of scalar field values
    grid_spacing: tuple (dx, dy, dz)
    mesh: object with .triangles (list of vertex index triples) and .vertices (list of 3D coords)
    Returns: dict with 'area', 'volume', 'integral_gradient', 'mean_gradient', 'curvature_map'
    """
    dx, dy, dz = grid_spacing
    grad_V = compute_gradient(V, grid_spacing)
    triangles = mesh.triangles
    vertices = mesh.vertices

    area = 0.0
    volume = 0.0
    integral_gradient = 0.0
    curvature_map = {}

    for tri in triangles:
        a, b, c = vertices[tri]
        # Edge vectors
        ab = [b[0]-a[0], b[1]-a[1], b[2]-a[2]]
        ac = [c[0]-a[0], c[1]-a[1], c[2]-a[2]]
        # Cross product for area
        cross = [ab[1]*ac[2]-ab[2]*ac[1], ab[2]*ac[0]-ab[0]*ac[2], ab[0]*ac[1]-ab[1]*ac[0]]
        tri_area = 0.5 * math.sqrt(cross[0]**2 + cross[1]**2 + cross[2]**2)
        area += tri_area

        # Volume contribution (divergence theorem)
        ab_cross = [a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0]]
        bc_cross = [b[1]*c[2]-b[2]*c[1], b[2]*c[0]-b[0]*c[2], b[0]*c[1]-b[1]*c[0]]
        ca_cross = [c[1]*a[2]-c[2]*a[1], c[2]*a[0]-c[0]*a[2], c[0]*a[1]-c[1]*a[0]]
        volume += (ab_cross[0]+bc_cross[0]+ca_cross[0])*vertices[0][0] + \
                  (ab_cross[1]+bc_cross[1]+ca_cross[1])*vertices[0][1] + \
                  (ab_cross[2]+bc_cross[2]+ca_cross[2])*vertices[0][2]

        # Integral of gradient
        centroid = [(a[0]+b[0]+c[0])/3.0, (a[1]+b[1]+c[1])/3.0, (a[2]+b[2]+c[2])/3.0]
        grad_at_centroid = interpolate_gradient(grad_V, centroid, grid_spacing)
        grad_mag = math.sqrt(sum(g**2 for g in grad_at_centroid))
        integral_gradient += tri_area * grad_mag

    volume = abs(volume) / 6.0
    mean_gradient = integral_gradient / area if area > 0 else 0.0

    for i, vertex in enumerate(vertices):
        H, K = compute_vertex_curvature(vertex, i, mesh)
        curvature_map[i] = {'mean': H, 'gaussian': K}

    return {
        'area': area,
        'volume': volume,
        'integral_gradient': integral_gradient,
        'mean_gradient': mean_gradient,
        'curvature_map': curvature_map
    }

def compute_gradient(V, spacing):
    """Compute gradient using finite differences."""
    dx, dy, dz = spacing
    dims = [len(V), len(V[0]), len(V[0][0])]
    dVdx = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    dVdy = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    dVdz = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    
    # Central differences
    for i in range(1, dims[0]-1):
        for j in range(dims[1]):
            for k in range(dims[2]):
                dVdx[i][j][k] = (V[i+1][j][k] - V[i-1][j][k]) / (2 * dx)
    for i in range(dims[0]):
        for j in range(1, dims[1]-1):
            for k in range(dims[2]):
                dVdy[i][j][k] = (V[i][j+1][k] - V[i][j-1][k]) / (2 * dy)
    for i in range(dims[0]):
        for j in range(dims[1]):
            for k in range(1, dims[2]-1):
                dVdz[i][j][k] = (V[i][j][k+1] - V[i][j][k-1]) / (2 * dz)
    
    # Boundary handling (one-sided)
    for j in range(dims[1]):
        for k in range(dims[2]):
            dVdx[0][j][k] = (V[1][j][k] - V[0][j][k]) / dx
            dVdx[dims[0]-1][j][k] = (V[dims[0]-1][j][k] - V[dims[0]-2][j][k]) / dx
    for i in range(dims[0]):
        for k in range(dims[2]):
            dVdy[i][0][k] = (V[i][1][k] - V[i][0][k]) / dy
            dVdy[i][dims[1]-1][k] = (V[i][dims[1]-1][k] - V[i][dims[1]-2][k]) / dy
    for i in range(dims[0]):
        for j in range(dims[1]):
            dVdz[i][j][0] = (V[i][j][1] - V[i][j][0]) / dz
            dVdz[i][j][dims[2]-1] = (V[i][j][dims[2]-1] - V[i][j][dims[2]-2]) / dz
            
    return [dVdx, dVdy, dVdz]

def interpolate_gradient(grad_V, point, spacing):
    """Trilinear interpolation of gradient at a point."""
    dx, dy, dz = spacing
    dims = [len(grad_V[0]), len(grad_V[0][0]), len(grad_V[0][0][0])]
    u = point[0] / dx
    v = point[1] / dy
    w = point[2] / dz
    u = max(0, min(u, dims[0]-1))
    v = max(0, min(v, dims[1]-1))
    w = max(0, min(w, dims[2]-1))
    i0, i1 = int(math.floor(u)), int(math.ceil(u))
    j0, j1 = int(math.floor(v)), int(math.ceil(v))
    k0, k1 = int(math.floor(w)), int(math.ceil(w))
    i1 = min(i1, dims[0]-1)
    j1 = min(j1, dims[1]-1)
    k1 = min(k1, dims[2]-1)
    fu, fv, fw = u-i0, v-j0, w-k0
    
    def trilinear(g):
        c00 = (1-fu)*g[i0][j0][k0] + fu*g[i1][j0][k0]
        c10 = (1-fu)*g[i0][j1][k0] + fu*g[i1][j1][k0]
        c01 = (1-fu)*g[i0][j0][k1] + fu*g[i1][j0][k1]
        c11 = (1-fu)*g[i0][j1][k1] + fu*g[i1][j1][k1]
        return (1-fw)*( (1-fv)*c00 + fv*c10 ) + fw*( (1-fv)*c01 + fv*c11 )
        
    return [trilinear(grad_V[0]), trilinear(grad_V[1]), trilinear(grad_V[2])]

def compute_vertex_curvature(vertex, idx, mesh):
    """Compute mean and Gaussian curvature for a vertex using cotangent weights [G]."""
    # Gather neighboring vertices and triangles
    neighbors = set()
    tri_indices = []
    for t_idx, tri in enumerate(mesh.triangles):
        if idx in tri:
            tri_indices.append(t_idx)
            for v_idx in tri:
                if v_idx != idx:
                    neighbors.add(v_idx)
    
    if not neighbors:
        return 0.0, 0.0
        
    # Compute Voronoi area and curvature sums
    A_voronoi = 0.0
    sum_cot = [0.0, 0.0, 0.0]
    sum_angle = 0.0
    
    for t_idx in tri_indices:
        tri = mesh.triangles[t_idx]
        verts = [mesh.vertices[v] for v in tri]
        # Find angle opposite to vertex idx
        angle = 0.0
        cot_sum = 0.0
        for i in range(3):
            if tri[i] == idx:
                # Angle at vertex i
                p0 = verts[i]
                p1 = verts[(i+1)%3]
                p2 = verts[(i+2)%3]
                v1 = [p1[j]-p0[j] for j in range(3)]
                v2 = [p2[j]-p0[j] for j in range(3)]
                dot = sum(a*b for a,b in zip(v1,v2))
                cross = [v1[1]*v2[2]-v1[2]*v2[1], v1[2]*v2[0]-v1[0]*v2[2], v1[0]*v2[1]-v1[1]*v2[0]]
                angle = math.atan2(math.sqrt(sum(c**2 for c in cross)), dot)
                norm1 = math.sqrt(sum(a**2 for a in v1))
                norm2 = math.sqrt(sum(a**2 for a in v2))
                cot_sum = (norm1*norm2) / dot if dot != 0 else 0.0
                break
        sum_angle += angle
        sum_cot[0] += cot_sum
        # Voronoi area contribution (simplified as 1/3 of triangle area for each vertex)
        tri_area = 0.5 * math.sqrt(cross[0]**2 + cross[1]**2 + cross[2]**2)
        A_voronoi += tri_area / 3.0
        
    # Mean curvature vector approximation
    H_vec = [0.0, 0.0, 0.0]
    for n_idx in neighbors:
        n_vec = [mesh.vertices[n_idx][j] - vertex[j] for j in range(3)]
        H_vec = [H_vec[j] + n_vec[j] for j in range(3)]
    H = math.sqrt(sum(h**2 for h in H_vec)) / (2.0 * A_voronoi) if A_voronoi > 0 else 0.0
    
    # Gaussian curvature
    K = (2.0 * math.pi - sum_angle) / A_voronoi if A_voronoi > 0 else 0.0
    
    return H, K
```

## Subtopics

### [Volumetric Divergence Metric](12575-volumetric-divergence-metric.md)
The Volumetric Divergence Metric (VDC) estimates the accuracy of isosurfaces extracted by marching algorithms by calculating the total volume between the produced mesh isosurface and the ideal trilinear interpolation isosurface. It sets itself apart from sibling metrics like the Tetrahedron Volumetric Divergence (TVD) Predictor by providing a highly accurate ground truth through recursive subdivision, whereas TVD uses a faster, closed-form geometric approximation.

### [Curvature-Based Anisotropy Estimation](16471-curvature-based-anisotropy-estimation.md)
This method estimates anisotropy for surface shading by computing the principal curvatures of an isosurface. It serves as a baseline comparison against covariance-based methods, relying on differential geometry properties where principal curvature directions form an orthonormal basis in the tangent plane. Unlike covariance-based approaches that evaluate feature extent in both spatial and data domains, this method focuses strictly on the local geometry of the isosurface.

### [Precomputed Geometric Quantities for Contour Trees](17208-precomputed-geometric-quantities-for-contour-trees.md)
This method precomputes the geometric properties (area and volume) of all possible contours for a 3D scalar field, enabling real-time evaluation for any selected isovalue without expensive surface extraction. It differs from post-extraction methods by using a contour tree to hierarchically aggregate B-spline functions, allowing immediate quantification of topological features.

### Hierarchical Surface Segmentation
Segmentation of a surface into four types by combining Boolean operators for mean curvature (convex/concave) and Gaussian curvature (elliptic/hyperbolic) [S2]. Compute the mean curvature operator to determine if a region is convex or concave, and the Gaussian curvature sign operator to determine if a region is elliptic or hyperbolic [S2]. Classify each vertex based on the combination of mean and Gaussian curvature results [S2]. Used to explicitly localize minima of sulci and maxima of gyri, which serve as seed points for topology graph generation [S2].

## Operating characteristics
*   **Time Complexity**: Dominated by isosurface extraction ($O(N)$ for $N$ voxels) and gradient computation ($O(N)$) [G]. Curvature computation is $O(M)$ for $M$ vertices [G].
*   **Memory Complexity**: $O(N)$ for the scalar field and gradient, $O(M)$ for the mesh [G].
*   **Strengths**: Provides comprehensive geometric and topological insights; efficient for large datasets if optimized [G].
*   **Limitations**: Sensitive to noise; requires a well-formed mesh for accurate volume/area [G].
*   **When to Use**: When detailed geometric analysis of the isosurface is required, such as in medical imaging for tissue characterization or in scientific visualization for feature extraction [G].
*   **When Not to Use**: For real-time applications where only basic visualization is needed, or when the scalar field is extremely noisy without pre-smoothing [G].

## Background
The method was proposed by Pekar et al. in 2001 as an efficient approach for computing isosurface characteristics including the integral of the gradient, mean gradient, total area, confined volume, and curvature [S1, unofficial]. It builds on earlier strategies by Bajaj et al. [S1, unofficial]. The method is particularly useful in volumetric data visualization for understanding the structure and properties of isosurfaces [S1, unofficial].

## Sources
[S1] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
[S2] Hierarchical Isosurface Segmentation Based on Discrete Curvature, https://inria.hal.science/inria-00537470v1, fetched 2026-10-26
