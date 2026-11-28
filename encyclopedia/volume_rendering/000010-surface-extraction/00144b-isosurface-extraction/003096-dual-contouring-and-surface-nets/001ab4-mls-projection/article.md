# MLS Projection

Also called: MLS projection via Hermite Point Set Surfaces, Moving Least Squares Surface, Parabolic Case Triangulation

## Overview

MLS Projection is an iterative algorithm that projects a query point onto a surface defined by a set of input points and their associated normals, using Moving Least Squares (MLS) with Hermite data [S2, unofficial]. It solves the problem of robustly approximating a smooth surface from noisy or irregular point cloud data by fitting a local quadratic function to the neighborhood of the query point [S3]. It differs from [Dual Contouring](778-dual-contouring.md) and [Surface Nets](921-surface-nets.md) by explicitly computing a projected position and normal via weighted tangent plane projections and polynomial fitting, rather than relying solely on grid-based sign changes or linear interpolation of vertex positions [S2, unofficial].

## How it works

**Inputs:**
- `inputPositions`: Array of 3D vectors representing the input point cloud [S2, unofficial].
- `inputNormals`: Array of 3D vectors representing the normals at each input position [S2, unofficial].
- `inputPointsetKdTree`: A KD-tree data structure built from `inputPositions` for efficient nearest-neighbor search [S2, unofficial].
- `queryPoint`: The 3D vector to be projected onto the surface [S2, unofficial].
- `knn`: Integer, number of nearest neighbors to consider. Default: 20 [S2, unofficial].
- `radius`: Float, radius parameter for the kernel function [S2, unofficial].
- `kernel_type`: Integer selector for the weighting function (0=Gaussian, 1=Wendland, 2=Singular) [S2, unofficial].
- `numberOfIterations`: Integer, number of iterations for the optimization loop [S2, unofficial].

**Outputs:**
- `outputPoint`: The projected 3D position on the surface [S2, unofficial].
- `outputNormal`: The normalized normal vector at the projected position [S2, unofficial].

**Data Structures:**
- `Vec3`: A 3D vector structure with components `x`, `y`, `z` and operations for addition, subtraction, dot product, scalar multiplication, and normalization [G].
- `ANNidxArray`: Array of integers storing neighbor indices [S2, unofficial].
- `ANNdistArray`: Array of floats storing squared distances to neighbors [S2, unofficial].

**Step-by-Step Method:**

1. **Initialization:** Set `currentPoint = queryPoint` and `outputPoint = queryPoint` [S2, unofficial].
2. **Iteration Loop:** For `iter` from 0 to `numberOfIterations - 1`:
   a. **Nearest Neighbor Search:** Find the `knn` nearest neighbors of `currentPoint` in `inputPointsetKdTree` [S2, unofficial]. Retrieve indices `id_nearest_neighbors` and squared distances `square_distances_to_neighbors` [S2, unofficial].
   b. **Accumulator Initialization:** Initialize accumulators for the Hermite Point Set Surface (HPSS) formulation [S2, unofficial]: `sum_weight = 0.0`, `sum_1 = 0.0`, `sum_2 = Vec3(0, 0, 0)`, `sum_3 = Vec3(0, 0, 0)`, `sum_4 = 0.0`.
   c. **Weighted Accumulation:** For `i` from 0 to `knn - 1`: Get neighbor position `neighbor = inputPositions[id_nearest_neighbors[i]]` and neighbor normal `neighborNormal = inputNormals[id_nearest_neighbors[i]]` [S2, unofficial]. Compute weight `w = Get_kernel(kernel_type, sqrt(square_distances_to_neighbors[i]), radius)` [S2, unofficial]. Update accumulators: `sum_1 += w * dot(neighbor, neighborNormal)`, `sum_2 += w * neighbor`, `sum_3 += w * neighborNormal`, `sum_weight += w`, `sum_4 += w * dot(neighbor, neighbor)` [S2, unofficial].
   d. **Polynomial Coefficient Calculation:** Compute the coefficients of the local quadratic polynomial $p(x) = u_4 x^T x + u_{123}^T x + u_0$ [S2, unofficial]:
      $$u_4 = 0.5 \cdot \frac{\sum_1 - \frac{\text{dot}(\sum_2, \sum_3)}{\sum_{\text{weight}}}}{\sum_4 - \frac{\text{dot}(\sum_2, \sum_2)}{\sum_{\text{weight}}}}$$
      $$u_{123} = \frac{\sum_3 - 2 \cdot u_4 \cdot \sum_2}{\sum_{\text{weight}}}$$
      $$u_0 = -\frac{\text{dot}(\sum_2, u_{123}) + u_4 \cdot \sum_4}{\sum_{\text{weight}}}$$
   e. **Implicit Function Evaluation and Update:** Evaluate the implicit function value at `outputPoint`: $f(\text{outputPoint}) = u_0 + \text{dot}(u_{123}, \text{outputPoint}) + u_4 \cdot \text{dot}(\text{outputPoint}, \text{outputPoint})$ [S2, unofficial]. Compute the gradient (normal) at `outputPoint`: $\nabla f(\text{outputPoint}) = u_{123} + 2 \cdot u_4 \cdot \text{outputPoint}$ [S2, unofficial]. Normalize the gradient to get the new normal: $\text{outputNormal} = \frac{\nabla f(\text{outputPoint})}{\|\nabla f(\text{outputPoint})\|}$ [S2, unofficial]. Compute the step: $\text{diff} = f(\text{outputPoint}) \cdot \text{outputNormal}$ [S2, unofficial]. Update the point: $\text{outputPoint} = \text{outputPoint} - \text{diff}$ [S2, unofficial]. Set `currentPoint = outputPoint` for the next iteration [S2, unofficial].
3. **Final Output:** Set `outputPoint` to the final `currentPoint` and normalize `outputNormal` one last time [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x, self.y, self.z = x, y, z
    def __add__(self, other): return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __sub__(self, other): return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __mul__(self, scalar): return Vec3(self.x * scalar, self.y * scalar, self.z * scalar)
    def __rmul__(self, scalar): return self.__mul__(scalar)
    def dot(self, other): return self.x*other.x + self.y*other.y + self.z*other.z
    def length(self): return math.sqrt(self.x**2 + self.y**2 + self.z**2)
    def normalize(self):
        l = self.length()
        if l > 1e-10: self.x, self.y, self.z = self.x/l, self.y/l, self.z/l
        return self

def get_kernel(kernel_type, r, h, w=2.0):
    if kernel_type == 0: return math.exp(-(r/h)**2)  # Gaussian
    if kernel_type == 1: return ((1 - r/h)**4 * (1 + 4*r/h)) if r < h else 0.0  # Wendland
    if kernel_type == 2: return (r/h)**-w  # Singular
    return 0.0

def mls_projection(query_point, input_positions, input_normals, kdtree, knn, radius, kernel_type, num_iterations):
    current_point = Vec3(*query_point)
    output_point = Vec3(*query_point)
    output_normal = Vec3(0, 0, 1)
    
    for _ in range(num_iterations):
        id_nearest_neighbors, square_distances_to_neighbors = kdtree.knearest(current_point, knn)
        
        sum_weight = 0.0
        sum_1 = 0.0
        sum_2 = Vec3(0, 0, 0)
        sum_3 = Vec3(0, 0, 0)
        sum_4 = 0.0
        
        for i in range(knn):
            neighbor = Vec3(*input_positions[id_nearest_neighbors[i]])
            neighbor_normal = Vec3(*input_normals[id_nearest_neighbors[i]])
            dist = math.sqrt(square_distances_to_neighbors[i])
            weight = get_kernel(kernel_type, dist, radius)
            
            sum_1 += weight * neighbor.dot(neighbor_normal)
            sum_2 += weight * neighbor
            sum_3 += weight * neighbor_normal
            sum_weight += weight
            sum_4 += weight * neighbor.dot(neighbor)
            
        if sum_weight == 0:
            output_normal = Vec3(0, 0, 1)
            break
            
        denom = sum_4 - sum_2.dot(sum_2) / sum_weight
        if abs(denom) < 1e-10:
            output_point = sum_2 / sum_weight
            output_normal = sum_3 / sum_weight
            output_normal.normalize()
            current_point = output_point
            continue
            
        u4 = 0.5 * (sum_1 - sum_2.dot(sum_3) / sum_weight) / denom
        u123 = (sum_3 - 2 * u4 * sum_2) / sum_weight
        u0 = -(sum_2.dot(u123) + u4 * sum_4) / sum_weight
        
        f_val = u0 + u123.dot(output_point) + u4 * output_point.dot(output_point)
        grad = u123 + 2 * u4 * output_point
        grad_len = grad.length()
        if grad_len < 1e-10:
            output_normal = Vec3(0, 0, 1)
        else:
            output_normal = grad * (1.0 / grad_len)
            
        diff = f_val * output_normal
        output_point = output_point - diff
        current_point = output_point
        
    output_normal.normalize()
    return (output_point.x, output_point.y, output_point.z), (output_normal.x, output_normal.y, output_normal.z)
```

**Edge Cases and Pitfalls:**
- **Degenerate Denominator:** If the denominator in the calculation of $u_4$ is near zero, the local point distribution is degenerate (e.g., all points are collinear or coplanar in a way that makes the quadratic fit ill-conditioned) [G]. A fallback to linear averaging or a default normal is required [G].
- **Zero Weight:** If `sum_weight` is zero (no neighbors within radius), the algorithm cannot proceed [G]. A guard clause should handle this, possibly returning the input point and a default normal [G].
- **Precision:** The iterative nature requires careful handling of floating-point precision, especially when `f_val` becomes very small [G]. Convergence criteria should be checked to avoid infinite loops if `numberOfIterations` is large [G].
- **Parabolic Case:** In the context of surface triangulation from MLS, if one eigenvalue of the quadratic form is zero ($\lambda_1 > 0, \lambda_2 = 0$), the surface is parabolic [S1, unofficial]. The transformation to the canonical form $p(\bar{x}, \bar{y}) = \bar{x}^2$ is used for triangulation, where the error profile is determined by the projection onto the first eigenvector $e_1$, with a maximum component length of $\sqrt{4\epsilon/\lambda_1}$ [S1, unofficial]. Vertices are placed below the surface by $\epsilon$ if $\lambda_1 > 0$ [S1, unofficial].

## Subtopics

### [APSS (Approximate Projected Point Set Surfaces)](843-apss-approximate-projected-point-set-surfaces.md)
APSS is an iterative method to project a query point onto a surface defined by a point set with normals, using Moving Least Squares (MLS). It builds on [MLS Projection](823-mls-projection.md) by computing a local coordinate system and higher-order approximations via weighted accumulators, distinguishing it from standard MLS by explicitly tracking cross-product and dot-product terms for refined surface fitting.

### [Moving Least Squares Implicit Surface Reconstruction](5328-moving-least-squares-implicit-surface-reconstruction.md)
Moving Least Squares (MLS) Implicit Surface Reconstruction is a technique for reconstructing a surface from an unordered 3D point cloud by creating an implicit function field using Moving Least Squares, from which isosurfaces are extracted. It differs from explicit reconstruction methods by defining the surface as a zero-level set of a scalar field, which allows robust handling of noisy data and topological changes without requiring explicit connectivity [G].

### Kernel Functions for MLS
A set of kernel functions used to weight neighbors in MLS surface reconstruction based on distance $r$ and scale $h$ [S2, unofficial].
- **Gaussian kernel** (`kernel_type=0`): $K(r) = e^{-r^2/h^2}$ [S2, unofficial].
- **Wendland kernel** (`kernel_type=1`): $K(r) = (1 - r/h)^4 (1 + 4r/h)$ [S2, unofficial].
- **Singular kernel** (`kernel_type=2`): $K(r) = (r/h)^{-w}$ [S2, unofficial].
Parameter `kernel_type` is an integer selector: 0=Gaussian, 1=Wendland, 2=Singular [S2, unofficial].

## Operating characteristics

**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Measured Performance:** Performance depends heavily on `knn`, `radius`, and `numberOfIterations` [S2, unofficial]. Typical values are `knn=20`, `numberOfIterations=10-20` [S2, unofficial]. The method is robust to noise due to the smoothing effect of the MLS weighting functions [S3].
**Strengths:** Handles noisy data well [S3]. Produces smooth surfaces [S3]. Can be used for implicit function evaluation, allowing for topological changes [G].
**Limitations:** Requires normals as input [S2, unofficial]. Degenerate cases (e.g., collinear points) can lead to numerical instability [G].
**When to use:** Not given in the sources.
**When not to use:** Not given in the sources.

## Background

MLS Projection was introduced as part of the Hermite Point Set Surfaces (HPSS) method for surface reconstruction from point sets with normals [S2, unofficial]. It is based on the Moving Least Squares method, which fits a local polynomial to the data weighted by distance [S3]. The parabolic case handling is described in the context of multiresolution modeling for scientific visualization [S1, unofficial].

## Sources

[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S2] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S3] High-Quality Extraction of Isosurfaces from Regular and Irregular Grids, https://doi.org/10.1109/tvcg.2006.149, fetched 2026-10-26
