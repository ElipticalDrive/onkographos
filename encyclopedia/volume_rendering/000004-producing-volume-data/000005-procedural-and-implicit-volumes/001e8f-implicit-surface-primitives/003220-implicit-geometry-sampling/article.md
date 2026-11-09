# Implicit Geometry Sampling

Also called: GenusTwoSurface, PorousSurface, WineGlassSurface

## Overview
Implicit Geometry Sampling is a technique for generating volume data from mathematical implicit functions, providing value and gradient at any point in space without relying on discrete voxel grids [S1, unofficial]. It solves the problem of defining complex, continuous geometry for volume rendering by evaluating a scalar field function $f(\vec{x})$ on-demand [S1, unofficial]. Unlike sibling methods that may rely on precomputed grids or specific primitive compositions, this method provides an abstract interface for sampling arbitrary implicit geometries, returning position, scalar value, and gradient [S1, unofficial]. It builds on [Implicit Surface Primitives](1800-implicit-surface-primitives.md) by implementing the core evaluation mechanism for specific algebraic and transcendental functions.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point $\vec{p} \in \mathbb{R}^3$ in world space.
*   **Output:** A structure containing the scalar field value $f(\vec{p})$ and the gradient vector $\nabla f(\vec{p})$ at that point.

**Data Structures:**
The implementation relies on an abstract base class `ImplicitGeometry` and specific derived classes.
*   `ImplicitGeometry`: Contains a protected member `AABB bounding_box` defining the spatial extent of the geometry [S1, unofficial].
*   `Volume` (for discrete variants): Contains protected members `grid_data` (pointer to `VolumeSampleData`), `grid_dimension` (`Eigen::Vector3i`), and `cell_size` (`Eigen::Vector3f`) [S1, unofficial].

**Evaluation Steps:**
1.  **Coordinate Transformation:** Translate the input point $\vec{p}$ to local coordinates relative to the bounding box center $\vec{c}$:
    $$ \vec{p}_{local} = \vec{p} - \vec{c} $$
    [S1, unofficial]
2.  **Domain Check:** If the point lies outside the defined domain of the specific implicit function (e.g., logarithmic singularities), return infinity for the value and zero for the gradient [S1, unofficial].
3.  **Function Evaluation:** Evaluate the specific implicit function $f(\vec{x})$ and its partial derivatives $\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z}$ at $\vec{p}_{local}$ [S1, unofficial].
4.  **Gradient Computation (Discrete Fallback):** If the geometry is stored as a discrete volume and gradients are not precomputed, compute them using central differences [S1, unofficial].
5.  **Voxel Retrieval (Discrete Fallback):** For discrete volumes, identify the voxel containing the position using `Voxel getVoxel(const Eigen::Vector3f& position)` [S1, unofficial].

**Formulas for Specific Primitives:**
The following formulas define the value $f$ and gradient components $(dx, dy, dz)$ for the named variants [S1, unofficial]:

1.  **GenusTwoSurface:**
    $$ f(x,y,z) = 2y(y^2 - 3x^2)(1 - z^2) + (x^2 + y^2)^2 - (9z^2 - 1)(1 - z^2) $$
    $$ dx = -12(1-z^2)yx + 4x^3 + 4y^2x $$
    $$ dy = 6(1-z^2)(y^2-x^2) + 4y^3 + 4x^2y $$
    $$ dz = -4(y^2-3x^2)yz - (20z - 36z^3) $$

2.  **WineGlassSurface:**
    Defined for $z > -3.2$:
    $$ f(x,y,z) = x^2 + y^2 - \log(z + 3.2)^2 - 0.09 $$
    *Note: If $z + 3.2 \le 0$, $f$ returns INFINITY and gradient returns $(0,0,0)$ [S1, unofficial].*

3.  **PorousSurface:**
    $$ f(x,y,z) = -0.02 + (y^2-0.88)^2(2.92(x^3-x)x + 1.7y^2)^2 + (z^2-0.88)^2(2.92(y^3-y)y + 1.7z^2)^2 + (x^2-0.88)^2(1.7x^2 + 2.92(z^3-z)z)^2 $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ImplicitGeometry:
    def __init__(self, bounding_box):
        self.bounding_box = bounding_box

    def sample(self, p):
        """
        Evaluates the implicit function and its gradient at point p.
        Returns: (value, gradient_vector)
        """
        # 1. Translate to local coordinates relative to bounding box center
        center = self.bounding_box.center()
        p_local = p - center
        
        # 2. Evaluate specific function and gradient
        # This is implemented in derived classes
        value, gradient = self.evaluate(p_local)
        
        return value, gradient

class GenusTwoSurface(ImplicitGeometry):
    def evaluate(self, p):
        x, y, z = p.x, p.y, p.z
        
        # Check domain if necessary (not strictly required for this polynomial)
        
        # Value
        term1 = 2 * y * (y**2 - 3 * x**2) * (1 - z**2)
        term2 = (x**2 + y**2)**2
        term3 = (9 * z**2 - 1) * (1 - z**2)
        value = term1 + term2 - term3
        
        # Gradient
        dx = -12 * (1 - z**2) * y * x + 4 * x**3 + 4 * y**2 * x
        dy = 6 * (1 - z**2) * (y**2 - x**2) + 4 * y**3 + 4 * x**2 * y
        dz = -4 * (y**2 - 3 * x**2) * y * z - (20 * z - 36 * z**3)
        
        return value, Vector3f(dx, dy, dz)

class WineGlassSurface(ImplicitGeometry):
    def evaluate(self, p):
        x, y, z = p.x, p.y, p.z
        
        # Domain boundary check
        if z + 3.2 <= 0:
            return INFINITY, Vector3f(0, 0, 0)
        
        # Value
        value = x**2 + y**2 - (math.log(z + 3.2))**2 - 0.09
        
        # Gradient (derived from value formula)
        # df/dx = 2x
        # df/dy = 2y
        # df/dz = -2 * log(z+3.2) * (1/(z+3.2))
        log_val = math.log(z + 3.2)
        dz = -2 * log_val / (z + 3.2)
        
        return value, Vector3f(2 * x, 2 * y, dz)

class PorousSurface(ImplicitGeometry):
    def evaluate(self, p):
        x, y, z = p.x, p.y, p.z
        
        # Value
        term_y = (y**2 - 0.88)**2 * (2.92 * (x**3 - x) * x + 1.7 * y**2)**2
        term_z = (z**2 - 0.88)**2 * (2.92 * (y**3 - y) * y + 1.7 * z**2)**2
        term_x = (x**2 - 0.88)**2 * (1.7 * x**2 + 2.92 * (z**3 - z) * z)**2
        value = -0.02 + term_y + term_z + term_x
        
        # Gradient calculation omitted as explicit formulas not provided in sources [G]
        # Implement central difference or symbolic differentiation as needed
        return value, Vector3f(0, 0, 0) # Placeholder for gradient
```

## Subtopics

### Implicit Function Density Definition
A variant where volume density $\rho(\vec{x})$ is derived directly from the implicit function value. The geometry is converted to a levelset (signed distance function) via the Fast Marching Method [S2, unofficial]. Density is defined as:
$$ \rho(\vec{x}) = \begin{cases} f(\vec{x})/f_{max} & \text{if } f(\vec{x}) > 0 \text{ (inside)} \\ 0 & \text{if } f(\vec{x}) \le 0 \text{ (outside)} \end{cases} $$
[S2, unofficial]. Common implicit functions used in this context include the Torus ($4R^2(x^2+z^2) - (|x|+R-r)^2 = 0$) and Cone ($\cos^{-1}((\vec{x}\cdot\hat{a})/|\vec{x}|) - \theta = 0$) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point, as it involves evaluating a mathematical function. However, complex functions (like PorousSurface) may involve many arithmetic operations.
*   **Memory Complexity:** $O(1)$ for the function definition itself. Does not require storing a voxel grid, unlike discrete volume methods [S1, unofficial].
*   **Strengths:** Infinite resolution; no aliasing artifacts from grid discretization; easy to modify geometry by changing parameters [S1, unofficial].
*   **Limitations:** No precomputed acceleration structures (like BVHs) are inherent to the sampling method itself, though they can be built on top [S1, unofficial]. Performance depends on the complexity of the algebraic/transcendental function [S2, unofficial].
*   **When to use:** When high-quality, alias-free rendering of smooth, complex implicit shapes is required, and memory for large voxel grids is prohibitive.
*   **When not to use:** When the geometry is highly irregular or requires explicit mesh-like features that are difficult to express as a single implicit function.

## Background
Implicit geometry sampling originates from the field of F-Rep (Functional Representation) solids. A seminal work is "The Making of Black-Hole and Nebula Clouds for the Motion Picture 'Sphere' with Volumetric Rendering and the F-Rep of Solids" by Gokhan Kisacikoglu (Siggraph 1998) [S2, unofficial]. The technique is widely used in procedural volume rendering to generate density fields from mathematical definitions, avoiding the storage and interpolation errors associated with discrete voxel grids [S1, S2, unofficial].

## Sources
[S1] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S2] PDF Production Volume Rendering - Clemson University, https://jtessen.people.clemson.edu/reports/papers_files/ProdVolRender.pdf, fetched 2026-10-27 (unofficial)
