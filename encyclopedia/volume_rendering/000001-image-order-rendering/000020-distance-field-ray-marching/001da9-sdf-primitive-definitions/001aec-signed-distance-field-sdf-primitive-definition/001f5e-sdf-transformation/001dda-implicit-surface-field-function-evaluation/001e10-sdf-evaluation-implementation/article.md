# SDF Evaluation Implementation

Also called: SDF Evaluation and Gradient, SDF Handler Implementation, SDF Query Handler, SDF Value and Gradient Evaluation

## Overview
This method implements the runtime evaluation of a Signed Distance Field (SDF) defined on a 3D voxel grid, providing both the scalar distance value and its spatial gradient (normal) at arbitrary query points. It solves the problem of efficiently querying implicit geometry for ray marching or differentiable rendering pipelines by supporting configurable interpolation orders. It differs from [Batched SDF Inference](2329-batched-sdf-inference.md) by operating on single arbitrary points rather than fixed batches, and from [DeepSDF Neural Network](3051-deepsdf-neural-network.md) by using explicit grid-based storage rather than neural network inference.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid representing the SDF, typically stored in `.sdf`, `.npy`, or `.vol` formats. The grid is defined by a resolution $R \times R \times R$ and contains scalar distance values. A query point $p = (x, y, z)$ in 3D space.
*   **Output:** The signed distance value $d = f(p)$ and the gradient vector $\nabla f(p) = (\frac{\partial d}{\partial x}, \frac{\partial d}{\partial y}, \frac{\partial d}{\partial z})$.

**Data Structures:**
*   **SDF Grid:** A 3D array of floating-point values. In implementation [S2, unofficial], this is wrapped in a texture object (e.g., `mi.Texture3f`) with acceleration disabled (`use_accel=False`) to allow direct coordinate-based sampling. The shape is $(R, R, R, 1)$.
*   **Configuration:** A string parameter `SDF_MODE` determining the interpolation method: `'linear'` or `'cubic'` [S2, unofficial].
*   **Parameters:**
    *   `epsilon` ($\epsilon$): The step size for finite difference gradient computation when in linear mode. Default value: $10^{-5}$ [S2, unofficial].

**Steps:**

1.  **Initialization and Validation:**
    *   Validate the SDF identifier/name using a helper function `is_valid_sdf_name(name)` [S1, unofficial]. If invalid, raise an error (e.g., `std::runtime_error`) [S1, unofficial].
    *   Load the raw grid data from the source file (`.sdf`, `.npy`, or `.vol`) [S2, unofficial].
    *   Parse the resolution $R$ from the first element of the data stream [S2, unofficial].
    *   Construct the internal grid representation (e.g., `mi.TensorXf` with shape `(res, res, res, 1)`) and wrap it in a texture sampler [S2, unofficial].

2.  **Value Evaluation $f(p)$:**
    *   Given query point $p$, determine the interpolation mode based on `SDF_MODE` [S2, unofficial].
    *   **If `SDF_MODE` is 'linear':**
        *   Perform trilinear interpolation on the voxel grid at $p$ [S2, unofficial].
        *   Formula: $d = \text{trilinear\_interp}(grid, p)$ [G].
    *   **If `SDF_MODE` is 'cubic':**
        *   Perform tricubic interpolation on the voxel grid at $p$ [S2, unofficial].
        *   Formula: $d = \text{tricubic\_interp}(grid, p)$ [G].

3.  **Gradient Evaluation $\nabla f(p)$:**
    *   **If `SDF_MODE` is 'linear':**
        *   Compute the gradient using central finite differences with step size $\epsilon = 10^{-5}$ [S2, unofficial].
        *   Define perturbed points:
            $$p_x^+ = p + (\epsilon, 0, 0), \quad p_x^- = p - (\epsilon, 0, 0)$$
            $$p_y^+ = p + (0, \epsilon, 0), \quad p_y^- = p - (0, \epsilon, 0)$$
            $$p_z^+ = p + (0, 0, \epsilon), \quad p_z^- = p - (0, 0, \epsilon)$$
        *   Evaluate values at these points using the linear interpolation step (Step 2) [S2, unofficial].
        *   Compute components:
            $$\frac{\partial d}{\partial x} = \frac{f(p_x^+) - f(p_x^-)}{2\epsilon}$$
            $$\frac{\partial d}{\partial y} = \frac{f(p_y^+) - f(p_y^-)}{2\epsilon}$$
            $$\frac{\partial d}{\partial z} = \frac{f(p_z^+) - f(p_z^-)}{2\epsilon}$$
    *   **If `SDF_MODE` is 'cubic':**
        *   Use the analytical gradient provided by the cubic interpolation kernel [S2, unofficial].
        *   Formula: $\nabla f(p) = \text{tricubic\_gradient}(grid, p)$ [G].

4.  **Normal Computation (Optional Helper):**
    *   To compute the surface normal $n$ at point $q$, normalize the gradient:
        $$n = \frac{\nabla f(q)}{\|\nabla f(q)\|}$$
    *   To compute the directional derivative along direction $d$, calculate the dot product:
        $$D_d = \nabla f(q) \cdot d$$
        Then normalize if required by specific pipeline constraints [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class SDFEvaluator:
    def __init__(self, grid_data, resolution, mode='linear', epsilon=1e-5):
        """
        Initialize SDF evaluator.
        
        Args:
            grid_data: 1D array of SDF values.
            resolution: Integer R, grid size is R x R x R.
            mode: 'linear' or 'cubic'.
            epsilon: Step size for finite differences.
        """
        self.resolution = resolution
        self.mode = mode
        self.epsilon = epsilon
        
        # Reshape to 3D grid (R, R, R)
        # Note: Source [S2, unofficial] mentions shape (res, res, res, 1) for texture, 
        # but logical access is 3D. We assume standard 3D array layout here.
        self.grid = grid_data.reshape(resolution, resolution, resolution)
        
        # Precompute bounds for clamping if necessary [G]
        self.min_val = np.min(self.grid)
        self.max_val = np.max(self.grid)

    def _trilinear(self, p):
        """
        Evaluate SDF value at point p using trilinear interpolation.
        p is assumed to be normalized [0, 1] or mapped to grid indices.
        For implementation completeness, we assume p is in grid coordinates [0, R-1].
        """
        x, y, z = p
        
        # Clamp to valid range
        x = np.clip(x, 0, self.resolution - 1)
        y = np.clip(y, 0, self.resolution - 1)
        z = np.clip(z, 0, self.resolution - 1)
        
        # Get integer and fractional parts
        ix, iy, iz = np.floor([x, y, z]).astype(int)
        fx, fy, fz = [x - ix, y - iy, z - iz]
        
        # Handle boundary indices
        ix_next = np.clip(ix + 1, 0, self.resolution - 1)
        iy_next = np.clip(iy + 1, 0, self.resolution - 1)
        iz_next = np.clip(iz + 1, 0, self.resolution - 1)
        
        # Fetch 8 corner values
        v000 = self.grid[ix, iy, iz]
        v100 = self.grid[ix_next, iy, iz]
        v010 = self.grid[ix, iy_next, iz]
        v110 = self.grid[ix_next, iy_next, iz]
        v001 = self.grid[ix, iy, iz_next]
        v101 = self.grid[ix_next, iy, iz_next]
        v011 = self.grid[ix, iy_next, iz_next]
        v111 = self.grid[ix_next, iy_next, iz_next]
        
        # Interpolate
        # Lerp in x
        v00 = v000 * (1 - fx) + v100 * fx
        v01 = v010 * (1 - fx) + v110 * fx
        v10 = v001 * (1 - fx) + v101 * fx
        v11 = v011 * (1 - fx) + v111 * fx
        
        # Lerp in y
        v0 = v00 * (1 - fy) + v01 * fy
        v1 = v10 * (1 - fy) + v11 * fy
        
        # Lerp in z
        return v0 * (1 - fz) + v1 * fz

    def _tricubic(self, p):
        """
        Evaluate SDF value and gradient using tricubic interpolation.
        This is a placeholder for the specific kernel implementation 
        provided by the underlying library (e.g., mi.Texture3f).
        """
        # Source [S2, unofficial] uses self.grid.eval_cubic(p)[0] for value
        # and self.grid.eval_cubic_grad(p)[1][0] for gradient.
        # Since we cannot call external library internals, we note this step.
        raise NotImplementedError("Tricubic kernel implementation depends on specific library bindings.")

    def evaluate(self, p):
        """
        Evaluate SDF value and gradient at point p.
        
        Args:
            p: numpy array [x, y, z] in grid coordinates.
            
        Returns:
            d: float, signed distance.
            grad: numpy array [dx, dy, dz], gradient.
        """
        if self.mode == 'linear':
            d = self._trilinear(p)
            
            # Central finite differences
            p_x_plus = p + np.array([self.epsilon, 0, 0])
            p_x_minus = p - np.array([self.epsilon, 0, 0])
            p_y_plus = p + np.array([0, self.epsilon, 0])
            p_y_minus = p - np.array([0, self.epsilon, 0])
            p_z_plus = p + np.array([0, 0, self.epsilon])
            p_z_minus = p - np.array([0, 0, self.epsilon])
            
            dx = (self._trilinear(p_x_plus) - self._trilinear(p_x_minus)) / (2 * self.epsilon)
            dy = (self._trilinear(p_y_plus) - self._trilinear(p_y_minus)) / (2 * self.epsilon)
            dz = (self._trilinear(p_z_plus) - self._trilinear(p_z_minus)) / (2 * self.epsilon)
            
            grad = np.array([dx, dy, dz])
            
        elif self.mode == 'cubic':
            # Placeholder for cubic evaluation
            # Source [S2, unofficial] uses self.grid.eval_cubic(p)[0]
            d = self._tricubic(p) # Implementation detail omitted
            # Source [S2, unofficial] uses self.grid.eval_cubic_grad(p)[1][0]
            grad = np.zeros(3) # Implementation detail omitted
        else:
            raise ValueError(f"Unknown SDF mode: {self.mode}")
            
        return d, grad

    def query_normal(self, q):
        """
        Compute the normalized normal vector at point q.
        """
        _, grad = self.evaluate(q)
        norm = np.linalg.norm(grad)
        if norm < 1e-8:
            return np.array([0.0, 0.0, 0.0]) # Degenerate case
        return grad / norm
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Points outside the voxel grid bounds must be handled. Linear interpolation typically clamps or extrapolates; cubic interpolation may produce artifacts or NaNs if not clamped [G].
*   **Precision:** Finite differences in linear mode are sensitive to the choice of $\epsilon$. Too small $\epsilon$ leads to numerical noise; too large $\epsilon$ reduces gradient accuracy [S2, unofficial].
*   **Cubic Performance:** Cubic interpolation is theoretically more robust and produces smoother geometry but is significantly slower than linear interpolation [S2, unofficial].

## Subtopics

### [Batched SDF Inference](2329-batched-sdf-inference.md)
Evaluates SDF values for large sets of samples in chunks to optimize GPU throughput via vectorized operations. Choose when processing many points simultaneously.

### [SDF Validity Check with Relaxation](2399-sdf-validity-check-with-relaxation.md)
Filters SDF samples to distinguish valid near-surface values from noise using a relaxation threshold. Choose when ray marching through ambiguous regions to prevent artifacts.

### [SDF Training with Eikonal Loss](2553-sdf-training-with-eikonal-loss.md)
Optimizes a neural implicit SDF using Eikonal constraints. Choose when training neural networks for SDFs.

### [DeepSDF Neural Network](3051-deepsdf-neural-network.md)
Maps latent codes and coordinates to SDF values using an MLP. Choose for continuous implicit surface reconstruction from sparse data.

### [SDF Optimization via Finite Differences](3086-sdf-optimization-via-finite-differences.md)
Optimizes SDF values and deformation weights using mesh-based loss gradients. Choose when analytical gradients are unavailable.

### [Mandelbulb SDF Evaluation](4108-mandelbulb-sdf-evaluation.md)
Computes SDF for Mandelbulb fractals. Choose for rendering fractal geometry.

### [SDF and Material Optimization Pipeline](4458-sdf-and-material-optimization-pipeline.md)
Jointly optimizes voxelized SDF and material volumes. Choose for inverse rendering with material optimization.

### [Augmented SDF](4725-augmented-sdf.md)
Accelerates queries using bounding volumes. Choose for performance optimization on complex scenes.

## Operating characteristics
*   **Time Complexity:**
    *   Linear Interpolation: $O(1)$ per query, with a constant factor proportional to the number of neighbors (8 for trilinear) and finite difference steps (6 evaluations for gradient).
    *   Cubic Interpolation: $O(1)$ per query, but with a higher constant factor due to the complexity of the cubic kernel (27 neighbors for tricubic) and potentially more expensive derivative calculations.
*   **Memory Complexity:** $O(R^3)$ for storing the SDF grid, where $R$ is the resolution.
*   **Performance:** Linear interpolation is significantly faster than cubic interpolation [S2, unofficial]. Cubic interpolation provides smoother geometry and is theoretically more robust for gradient-based optimization [S2, unofficial].
*   **Strengths:** Supports arbitrary query points; provides analytical or high-accuracy numerical gradients; flexible interpolation modes.
*   **Limitations:** Memory usage scales cubically with resolution; cubic interpolation is slow; linear interpolation gradients are noisy due to finite differences.
*   **When to Use:** When explicit grid-based SDFs are required for differentiable rendering or ray marching, and the scene geometry fits within memory constraints.
*   **When Not to Use:** For very high-resolution scenes where memory is a bottleneck (consider [DeepSDF Neural Network](3051-deepsdf-neural-network.md) or [Augmented SDF](4725-augmented-sdf.md)); when real-time performance is critical and linear interpolation is insufficient (consider [Batched SDF Inference](2329-batched-sdf-inference.md) for throughput).

## Background
The method originates from implementations of differentiable rendering with Signed Distance Fields, specifically as described in "A Simple Approach to Differentiable Rendering of SDFs" by Wang et al. (SIGGRAPH Asia 2024) [S2, unofficial]. It is also used in subgrid ray marching pipelines for efficient intersection queries [S1, unofficial]. The approach leverages standard voxel grid interpolation techniques adapted for gradient computation, enabling the use of SDFs in optimization-based rendering pipelines.

## Sources
[S1] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
[S2] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
