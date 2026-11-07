# Implicit Geometry Sampling

Also called: GenusTwoSurface, PorousSurface, WineGlassSurface

## Overview
Implicit Geometry Sampling is a technique for generating volume data from mathematical implicit functions, providing scalar value and gradient at any point in space without relying on discrete voxel grids for the definition [S1, unofficial]. It serves as the evaluation engine for procedural and implicit volumes, allowing for analytic or procedural definition, local or global support, and composition via set operations or blending [Implicit Surface Definition](2798-implicit-surface-definition.md). Unlike discrete volume methods, it computes values on-the-fly, optimizing for GPU parallelism or ray marching step sizes [Implicit Surface Definition](2798-implicit-surface-definition.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{p} \in \mathbb{R}^3$, the implicit function definition (e.g., GenusTwoSurface, WineGlassSurface, PorousSurface), and optionally a bounding box.
*   **Output:** A scalar value $f(\mathbf{p})$ and a gradient vector $\nabla f(\mathbf{p}) = (dx, dy, dz)$.

**Data Structures:**
The implementation relies on an abstract interface `ImplicitGeometry` and a concrete `Volume` class [S1, unofficial].

1.  **`ImplicitGeometry` Base Class:**
    *   Protected member: `AABB bounding_box` (Axis-Aligned Bounding Box defining the domain).
2.  **`Volume` Class (for discrete storage if needed):**
    *   Protected member: `grid_data` (pointer to `VolumeSampleData`).
    *   Protected member: `grid_dimension` (`Eigen::Vector3i`, dimensions of the voxel grid).
    *   Protected member: `cell_size` (`Eigen::Vector3f`, size of each voxel).

**Numbered Steps:**
1.  **Domain Check:** If the query point $\mathbf{p}$ is outside the `bounding_box` or in a restricted domain (e.g., $z \le -3.2$ for WineGlassSurface), return $f(\mathbf{p}) = \infty$ and $\nabla f(\mathbf{p}) = (0, 0, 0)$ [S1, unofficial].
2.  **Coordinate Transformation:** Translate the point $\mathbf{p}$ to local coordinates relative to the bounding box center $\mathbf{c}$:
    $$ \mathbf{p}_{local} = \mathbf{p} - \mathbf{c} $$
    [S1, unofficial]
3.  **Function Evaluation:** Evaluate the specific implicit function $f$ and its partial derivatives at $\mathbf{p}_{local}$.
4.  **Gradient Handling:**
    *   If the implicit function provides an analytic gradient, use it directly.
    *   If the volume is stored discretely and gradients are not precomputed, compute them using central differences [S1, unofficial].
    *   Retrieve the voxel containing the position if accessing discrete data: `Voxel getVoxel(const Eigen::Vector3f& position)` [S1, unofficial].

**Formulas:**
The following specific implicit functions are defined in the sources [S1, unofficial]:

1.  **GenusTwoSurface:**
    $$ f(x,y,z) = 2y(y^2 - 3x^2)(1 - z^2) + (x^2 + y^2)^2 - (9z^2 - 1)(1 - z^2) $$
    Gradient components:
    $$ dx = -12(1-z^2)yx + 4x^3 + 4y^2x $$
    $$ dy = 6(1-z^2)(y^2-x^2) + 4y^3 + 4x^2y $$
    $$ dz = -4(y^2-3x^2)yz - (20z - 36z^3) $$

2.  **WineGlassSurface:**
    Defined for $z > -3.2$:
    $$ f(x,y,z) = x^2 + y^2 - \log(z + 3.2)^2 - 0.09 $$
    *Note: The sources provide the value formula but do not explicitly list the gradient components for WineGlassSurface. Implementers must derive them analytically or use numerical differentiation.*

3.  **PorousSurface:**
    $$ f(x,y,z) = -0.02 + (y^2-0.88)^2(2.92(x^3-x)x + 1.7y^2)^2 + (z^2-0.88)^2(2.92(y^3-y)y + 1.7z^2)^2 + (x^2-0.88)^2(1.7x^2 + 2.92(z^3-z)z)^2 $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class AABB:
    def __init__(self, center, half_size):
        self.center = center  # np.array([x, y, z])
        self.half_size = half_size  # np.array([x, y, z])

    def contains(self, point):
        return np.all(np.abs(point - self.center) <= self.half_size)

def evaluate_genus_two(p_local):
    x, y, z = p_local
    val = 2*y*(y**2 - 3*x**2)*(1 - z**2) + (x**2 + y**2)**2 - (9*z**2 - 1)*(1 - z**2)
    dx = -12*(1-z**2)*y*x + 4*x**3 + 4*y**2*x
    dy = 6*(1-z**2)*(y**2 - x**2) + 4*y**3 + 4*x**2*y
    dz = -4*(y**2 - 3*x**2)*y*z - (20*z - 36*z**3)
    return val, np.array([dx, dy, dz])

def evaluate_wine_glass(p_local):
    x, y, z = p_local
    if z + 3.2 <= 0:
        return float('inf'), np.array([0.0, 0.0, 0.0])
    val = x**2 + y**2 - math.log(z + 3.2)**2 - 0.09
    # Gradient not provided in sources; derived as:
    # dx = 2x, dy = 2y, dz = -2*log(z+3.2)/(z+3.2)
    dx = 2*x
    dy = 2*y
    dz = -2 * math.log(z + 3.2) / (z + 3.2)
    return val, np.array([dx, dy, dz])

def evaluate_porous(p_local):
    x, y, z = p_local
    term1 = (y**2 - 0.88)**2 * (2.92*(x**3 - x)*x + 1.7*y**2)**2
    term2 = (z**2 - 0.88)**2 * (2.92*(y**3 - y)*y + 1.7*z**2)**2
    term3 = (x**2 - 0.88)**2 * (1.7*x**2 + 2.92*(z**3 - z)*z)**2
    val = -0.02 + term1 + term2 + term3
    # Gradient not provided in sources; implementers must derive analytically or use central differences.
    # [G] Central difference approximation for gradient if analytic is unavailable:
    eps = 1e-5
    dx = (evaluate_porous(np.array([x+eps, y, z]))[0] - evaluate_porous(np.array([x-eps, y, z]))[0]) / (2*eps)
    dy = (evaluate_porous(np.array([x, y+eps, z]))[0] - evaluate_porous(np.array([x, y-eps, z]))[0]) / (2*eps)
    dz = (evaluate_porous(np.array([x, y, z+eps]))[0] - evaluate_porous(np.array([x, y, z-eps]))[0]) / (2*eps)
    return val, np.array([dx, dy, dz])

class ImplicitGeometry:
    def __init__(self, bounding_box):
        self.bounding_box = bounding_box

    def sample(self, p):
        if not self.bounding_box.contains(p):
            return float('inf'), np.array([0.0, 0.0, 0.0])
        
        p_local = p - self.bounding_box.center
        # Dispatch to specific function based on type
        # This is a placeholder for the specific geometry type
        raise NotImplementedError("Subclass must implement sample")

class GenusTwoSurface(ImplicitGeometry):
    def sample(self, p):
        p_local = p - self.bounding_box.center
        return evaluate_genus_two(p_local)

class WineGlassSurface(ImplicitGeometry):
    def sample(self, p):
        p_local = p - self.bounding_box.center
        return evaluate_wine_glass(p_local)

class PorousSurface(ImplicitGeometry):
    def sample(self, p):
        p_local = p - self.bounding_box.center
        return evaluate_porous(p_local)
```

**Parameters:**
*   **Bounding Box:** Defines the spatial extent. Not given in the sources.
*   **Epsilon:** Used for numerical gradient estimation if analytic gradients are unavailable. Typical value $10^{-5}$ [G].

**Edge Cases and Pitfalls:**
*   **Domain Boundaries:** For WineGlassSurface, if $z + 3.2 \le 0$, the function returns $\infty$ for value and $(0,0,0)$ for gradient [S1, unofficial].
*   **Gradient Accuracy:** If analytic gradients are not provided (e.g., PorousSurface, WineGlassSurface in sources), numerical differentiation introduces truncation and rounding errors. Central differences are recommended [S1, unofficial].
*   **Precision:** Implicit functions can be sensitive to floating-point precision near the zero-crossing surface.

## Subtopics

### GenusTwoSurface
A specific implicit function defining a surface of genus two. Chosen when a complex, toroidal topology is required. [GenusTwoSurface](#)

### WineGlassSurface
A specific implicit function defined for $z > -3.2$, resembling a wine glass shape. Chosen for vase-like or cup-like geometries. [WineGlassSurface](#)

### PorousSurface
A specific implicit function with complex polynomial terms, suitable for porous or sponge-like structures. Chosen for organic, high-frequency detail. [PorousSurface](#)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point, dependent on the complexity of the implicit function evaluation.
*   **Memory Complexity:** $O(1)$ for the function definition. If discretized into a `Volume` class, $O(N)$ where $N$ is the number of voxels.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with resolution since no resampling is needed; higher resolution simply requires more samples.
*   **Strengths:** Resolution-independent, smooth gradients (if analytic), easy composition.
*   **Limitations:** No analytic gradients for all functions (PorousSurface, WineGlassSurface), potential for numerical instability near boundaries.
*   **When to use:** When resolution independence and smoothness are critical, and the geometry can be described by an implicit function.
*   **When not to use:** When the geometry is highly complex and better represented by meshes or when performance is critical and the function evaluation is expensive.

## Background
Implicit Geometry Sampling is part of the broader field of procedural and implicit volume rendering. The specific functions (GenusTwoSurface, WineGlassSurface, PorousSurface) are documented in a CUDA-based interactive volume renderer repository [S1, unofficial].

## Sources
[S1] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
