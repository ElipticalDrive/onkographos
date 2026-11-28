# Clough–Tocher Bézier Patch Interpolation

Also called: C1 continuous Bézier patch, Clough-Tocher scheme, Clough–Tocher C1 Interpolation, Cubic Feature Interpolation

## Overview
Clough–Tocher Bézier Patch Interpolation replaces standard linear barycentric interpolation with a cubic Bézier patch construction to interpolate feature values and gradients over a triangle [S1]. It solves the problem of visible seam artifacts and high-frequency noise that arise from gradient discontinuities (C0 continuity) across triangle edges in fully opaque meshes [S1]. Unlike its sibling [Component-wise Tensor Interpolation](12137-component-wise-tensor-interpolation.md), which may handle tensor fields or different continuity requirements, this method specifically guarantees C1 continuity everywhere and C∞ within each triangle by utilizing vertex gradients to define 10 Bézier control points [S1].

## How it works
**Inputs:**
- Triangle vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$.
- Feature values at vertices $f_0, f_1, f_2$ (scalar or vector).
- Feature gradients at vertices $\nabla f_0, \nabla f_1, \nabla f_2$ (2D vectors in the tangent plane or 3D vectors projected appropriately).

**Outputs:**
- Interpolated feature value $f(\mathbf{u})$ at any point inside the triangle defined by barycentric coordinates $\mathbf{u} = (u, v, w)$ where $u+v+w=1$.

**Data Structures:**
The method computes 10 Bézier control points $\mathbf{c}_{ijk}$ for each triangle, indexed by non-negative integers $i,j,k$ such that $i+j+k=3$.
1.  **Vertex Points:** $\mathbf{c}_{300}, \mathbf{c}_{030}, \mathbf{c}_{003}$
2.  **Edge Points:** $\mathbf{c}_{210}, \mathbf{c}_{120}, \mathbf{c}_{021}, \mathbf{c}_{012}, \mathbf{c}_{102}, \mathbf{c}_{201}$
3.  **Center Point:** $\mathbf{c}_{111}$

**Step 1: Compute Edge Vectors**
Calculate the edge vectors for the triangle:
$$ \mathbf{e}_{01} = \mathbf{v}_1 - \mathbf{v}_0 $$
$$ \mathbf{e}_{12} = \mathbf{v}_2 - \mathbf{v}_1 $$
$$ \mathbf{e}_{20} = \mathbf{v}_0 - \mathbf{v}_2 $$

**Step 2: Compute Bézier Control Points**
Define the 10 control points using the vertex features and gradients:

$$ \mathbf{c}_{300} = f_0 $$
$$ \mathbf{c}_{030} = f_1 $$
$$ \mathbf{c}_{003} = f_2 $$

$$ \mathbf{c}_{210} = f_0 + \frac{1}{3} \nabla f_0 \cdot \mathbf{e}_{01} $$
$$ \mathbf{c}_{120} = f_1 - \frac{1}{3} \nabla f_1 \cdot \mathbf{e}_{01} $$

$$ \mathbf{c}_{021} = f_1 + \frac{1}{3} \nabla f_1 \cdot \mathbf{e}_{12} $$
$$ \mathbf{c}_{012} = f_2 - \frac{1}{3} \nabla f_2 \cdot \mathbf{e}_{12} $$

$$ \mathbf{c}_{102} = f_2 + \frac{1}{3} \nabla f_2 \cdot \mathbf{e}_{20} $$
$$ \mathbf{c}_{201} = f_0 - \frac{1}{3} \nabla f_0 \cdot \mathbf{e}_{20} $$

$$ \mathbf{c}_{111} = \frac{1}{6} (\mathbf{c}_{210} + \mathbf{c}_{120} + \mathbf{c}_{021} + \mathbf{c}_{012} + \mathbf{c}_{102} + \mathbf{c}_{201}) - \frac{1}{6} (\mathbf{c}_{300} + \mathbf{c}_{030} + \mathbf{c}_{003}) $$

**Step 3: Evaluate the Patch**
For a point with barycentric coordinates $(u, v, w)$, the interpolated value is given by the Bernstein polynomial basis of degree 3:

$$ f(u, v, w) = \sum_{i+j+k=3} \frac{3!}{i!j!k!} u^i v^j w^k \mathbf{c}_{ijk} $$

Explicitly:
$$
\begin{aligned}
f(u,v,w) = \;& u^3 \mathbf{c}_{300} + v^3 \mathbf{c}_{030} + w^3 \mathbf{c}_{003} \\
&+ 3u^2v \mathbf{c}_{210} + 3uv^2 \mathbf{c}_{120} \\
&+ 3v^2w \mathbf{c}_{021} + 3vw^2 \mathbf{c}_{012} \\
&+ 3w^2u \mathbf{c}_{102} + 3wu^2 \mathbf{c}_{201} \\
&+ 6uvw \mathbf{c}_{111}
\end{aligned}
$$

**Gradient Estimation (Preprocessing)**
The method requires per-vertex gradients $\nabla f_v$. These are estimated via a per-vertex least-squares fit over the one-ring neighborhood [S1]. For a vertex $v$ with neighbors $\{v_j\}_{j \in N(v)}$:

$$ \nabla f_v = \mathbf{A}_v^{-1} \mathbf{b}_v $$

Where:
$$ \mathbf{A}_v = \sum_{j} \Delta \mathbf{p}_j \Delta \mathbf{p}_j^\top $$
$$ \mathbf{b}_v = \sum_{j} \Delta f_j \Delta \mathbf{p}_j $$

Here $\Delta \mathbf{p}_j = \mathbf{v}_j - \mathbf{v}_v$ and $\Delta f_j = f_j - f_v$. The system $\mathbf{A}_v$ is a $2 \times 2$ matrix (assuming 2D tangent plane or planar projection) and is solved analytically per vertex [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_bezier_control_points(v0, v1, v2, f0, f1, f2, grad0, grad1, grad2):
    """
    Computes the 10 Bézier control points for a Clough-Tocher patch.
    
    Args:
        v0, v1, v2: numpy arrays of shape (3,) representing vertex positions.
        f0, f1, f2: numpy arrays of shape (D,) representing feature values at vertices.
        grad0, grad1, grad2: numpy arrays of shape (2,) or (3,) representing gradients.
                              Note: Dot product assumes gradients are compatible with edge vectors.
                              If gradients are 3D and edges are 3D, the dot product is standard.
                              If gradients are 2D (tangent), edges must be projected or 2D.
                              The source [S1] implies standard dot product with edge vectors.
    
    Returns:
        control_points: dict mapping index tuple (i,j,k) to numpy array of shape (D,)
    """
    # Edge vectors
    e01 = v1 - v0
    e12 = v2 - v1
    e20 = v0 - v2
    
    # Vertex control points
    c300 = f0
    c030 = f1
    c003 = f2
    
    # Edge control points
    # c210 = f0 + (1/3) * grad0 . e01
    c210 = f0 + (1.0/3.0) * np.dot(grad0, e01)
    # c120 = f1 - (1/3) * grad1 . e01
    c120 = f1 - (1.0/3.0) * np.dot(grad1, e01)
    
    # c021 = f1 + (1/3) * grad1 . e12
    c021 = f1 + (1.0/3.0) * np.dot(grad1, e12)
    # c012 = f2 - (1/3) * grad2 . e12
    c012 = f2 - (1.0/3.0) * np.dot(grad2, e12)
    
    # c102 = f2 + (1/3) * grad2 . e20
    c102 = f2 + (1.0/3.0) * np.dot(grad2, e20)
    # c201 = f0 - (1/3) * grad0 . e20
    c201 = f0 - (1.0/3.0) * np.dot(grad0, e20)
    
    # Center control point
    # c111 = (1/6) * (sum of edge points) - (1/6) * (sum of vertex points)
    sum_edges = c210 + c120 + c021 + c012 + c102 + c201
    sum_vertices = c300 + c030 + c003
    c111 = (1.0/6.0) * sum_edges - (1.0/6.0) * sum_vertices
    
    control_points = {
        (3,0,0): c300, (0,3,0): c030, (0,0,3): c003,
        (2,1,0): c210, (1,2,0): c120,
        (0,2,1): c021, (0,1,2): c012,
        (1,0,2): c102, (2,0,1): c201,
        (1,1,1): c111
    }
    
    return control_points

def evaluate_clough_tocher(u, v, w, control_points):
    """
    Evaluates the Clough-Tocher Bézier patch at barycentric coordinates (u, v, w).
    
    Args:
        u, v, w: floats such that u + v + w = 1.
        control_points: dict from compute_bezier_control_points.
        
    Returns:
        f: numpy array of shape (D,) interpolated feature value.
    """
    # Bernstein basis coefficients for degree 3
    # Term: (3! / (i!j!k!)) * u^i * v^j * w^k
    
    # Vertex terms
    term_300 = (u**3) * control_points[(3,0,0)]
    term_030 = (v**3) * control_points[(0,3,0)]
    term_003 = (w**3) * control_points[(0,0,3)]
    
    # Edge terms (coeff 3)
    term_210 = 3 * (u**2 * v) * control_points[(2,1,0)]
    term_120 = 3 * (u * v**2) * control_points[(1,2,0)]
    term_021 = 3 * (v**2 * w) * control_points[(0,2,1)]
    term_012 = 3 * (v * w**2) * control_points[(0,1,2)]
    term_102 = 3 * (w**2 * u) * control_points[(1,0,2)]
    term_201 = 3 * (w * u**2) * control_points[(2,0,1)]
    
    # Center term (coeff 6)
    term_111 = 6 * (u * v * w) * control_points[(1,1,1)]
    
    f = term_300 + term_030 + term_003 + \
        term_210 + term_120 + \
        term_021 + term_012 + \
        term_102 + term_201 + \
        term_111
        
    return f

def estimate_vertex_gradients(vertices, features, neighbors_map):
    """
    Estimates per-vertex gradients using least-squares fit over one-ring neighborhood.
    
    Args:
        vertices: dict mapping vertex_id to numpy array (3,) position.
        features: dict mapping vertex_id to numpy array (D,) feature value.
        neighbors_map: dict mapping vertex_id to list of neighbor vertex_ids.
        
    Returns:
        gradients: dict mapping vertex_id to numpy array (2,) or (3,) gradient.
                   Note: The source [S1] solves a 2x2 system, implying 2D gradients
                   or projection to tangent plane. This implementation assumes
                   2D gradient output for planar triangles or requires projection.
                   For simplicity, we return 2D gradients as per the 2x2 system note.
    """
    gradients = {}
    
    for v_id, v_pos in vertices.items():
        neighbors = neighbors_map.get(v_id, [])
        if not neighbors:
            gradients[v_id] = np.zeros(2) # Or 3, depending on context. Source says 2x2.
            continue
            
        # Collect local data
        # A_v = sum(pj * pj^T)
        # b_v = sum(fj * pj)
        # where pj = vj - v, fj = fj - f
        
        A = np.zeros((2, 2))
        b = np.zeros(2)
        
        fv = features[v_id]
        
        for n_id in neighbors:
            vn = vertices[n_id]
            fn = features[n_id]
            
            dp = vn - v_pos
            df = fn - fv
            
            # We need to project dp and df into a 2D basis if they are 3D,
            # or assume they are already 2D. The source [S1] mentions "2x2 system".
            # Assuming 2D coordinates for the local fit for this implementation.
            # If 3D, one would typically project to the triangle's tangent plane.
            # Here we assume dp and df are 2D vectors for the 2x2 solve.
            
            # Outer product dp * dp^T
            A += np.outer(dp, dp)
            
            # df * dp^T (result is vector)
            b += df * dp
            
        # Solve A * grad = b
        # Analytical solution for 2x2:
        # det = A00*A11 - A01*A10
        det = A[0,0]*A[1,1] - A[0,1]*A[1,0]
        
        if abs(det) < 1e-8:
            gradients[v_id] = np.zeros(2)
        else:
            grad = np.zeros(2)
            grad[0] = (A[1,1]*b[0] - A[0,1]*b[1]) / det
            grad[1] = (-A[1,0]*b[0] + A[0,0]*b[1]) / det
            gradients[v_id] = grad
            
    return gradients
```

**Edge Cases and Pitfalls:**
- **Degeneracy:** The mesh can become degenerate (e.g., vertices becoming collinear) during densification. This is handled by detecting degeneracy and re-meshing, although this rarely occurs in practice [S1].
- **Gradient Discontinuity:** Standard barycentric interpolation produces a C0 feature field where the gradient is discontinuous across edges, causing high-frequency artifacts. The Clough–Tocher scheme eliminates this by ensuring C1 continuity [S1].
- **Gradient Estimation:** The least-squares fit assumes a local planar approximation. If the neighborhood is sparse or noisy, the gradient estimate may be inaccurate, affecting the smoothness of the patch [S1].

## Subtopics

### Finite Element Interpolation
Finite Element Interpolation (FEM) approximates solutions over triangulated or tetrahedral meshes tailored to irregular 3D domains, incorporating basis functions for weak-form integration [S2]. Unlike Clough–Tocher which focuses on C1 continuity for rendering artifacts, FEM excels in handling complex geometries (e.g., terrain, biomedical structures) by adapting mesh resolution locally [S2]. It ensures conformity to boundaries but demands mesh generation, increasing setup complexity compared to direct interpolation methods [S2]. It is suitable for scenarios with scattered or irregular data where grid alignment is impractical [S2].

## Operating characteristics
- **Time Complexity:** The evaluation of the patch is O(1) per triangle, involving a fixed number of arithmetic operations (dot products and polynomial evaluations). Gradient estimation is O(N) where N is the number of vertices, as it involves a local neighborhood sum.
- **Memory Complexity:** Requires storing 10 control points per triangle (or equivalently, 3 vertex features and 3 vertex gradients).
- **Performance:** The method is implemented in custom CUDA kernels for real-time rendering [S1]. Training generally takes 1-3 minutes on an RTX A6000 Ada [S1].
- **Strengths:** Eliminates seam artifacts in fully opaque meshes; guarantees C1 continuity; differentiable (gradients flow through vertex topology) [S1].
- **Limitations:** Requires gradient information (estimated via least-squares); more complex than linear interpolation; sensitive to mesh degeneracy (though rare) [S1].
- **When to use:** When rendering fully opaque meshes with feature maps (normals, colors, UVs) where smoothness and continuity across triangle edges are critical, and linear interpolation causes visible seams [S1].
- **When not to use:** When computational resources are extremely limited and seam artifacts are acceptable; or when the mesh is highly irregular and gradient estimation is unstable.

## Background
The Clough–Tocher scheme is a classical method in computer-aided geometric design for constructing C1 continuous surfaces over triangular domains [S1]. In the context of volume rendering and neural reconstruction, it has been adapted to replace linear barycentric interpolation to improve the quality of feature reconstruction in primitive-based neural reconstruction methods [S1]. The specific application to neural harmonic textures and high-quality primitive-based reconstruction was detailed in "Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction" [S1].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] trilinear interpolation, https://grokipedia.com/page/Trilinear_interpolation, fetched 2026-10-26
