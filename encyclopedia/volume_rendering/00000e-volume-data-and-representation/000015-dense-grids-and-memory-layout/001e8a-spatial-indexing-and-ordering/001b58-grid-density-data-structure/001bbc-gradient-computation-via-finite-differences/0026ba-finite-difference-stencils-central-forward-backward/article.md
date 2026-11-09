# Finite Difference Stencils (Central/Forward/Backward)


## Overview
Finite difference stencils compute the spatial gradient $\nabla V$ of a scalar volume field $V$ at each voxel by approximating partial derivatives using values from neighboring voxels [G]. This method solves the problem of estimating surface normals for lighting calculations in volume rendering using minimal data access [G]. It is distinguished from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by requiring additional sample evaluations (typically six for central differences) rather than deriving gradients from basis function coefficients, and from [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) by operating on a structured grid rather than unstructured ray samples [G].

## Shared principles
All finite difference gradient methods approximate the partial derivative $\frac{\partial V}{\partial x_i}$ at a voxel center using a linear combination of scalar values $V$ at discrete grid points [G]. The general form for a stencil is:
$$ \frac{\partial V}{\partial x_i} \approx \frac{1}{h} \sum_{k} c_k V(\mathbf{p} + \delta_k) $$
where $h$ is the voxel spacing (assumed isotropic unless specified), $c_k$ are integer weights, and $\delta_k$ are integer offsets from the current voxel index [G]. The gradient vector $\mathbf{G} = (G_x, G_y, G_z)$ is constructed by computing these approximations for each axis independently [G]. Boundary conditions must be handled for voxels where neighbors fall outside the volume bounds, typically via clamping, zero-padding, or extrapolation [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_stencil(volume, weights, offsets, voxel_size):
    """
    Applies a finite difference stencil to a 3D volume.
    weights: list of float coefficients c_k.
    offsets: list of 3D integer tuples (dx, dy, dz) for delta_k.
    voxel_size: float h.
    Returns: 3D array of gradient vectors.
    """
    gx = np.zeros_like(volume)
    gy = np.zeros_like(volume)
    gz = np.zeros_like(volume)
    for z in range(volume.shape[2]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[0]):
                sum_x = 0.0
                sum_y = 0.0
                sum_z = 0.0
                for w, (dx, dy, dz) in zip(weights, offsets):
                    # Boundary handling: clamp indices to valid range [G]
                    vx = clamp(x + dx, 0, volume.shape[0] - 1)
                    vy = clamp(y + dy, 0, volume.shape[1] - 1)
                    vz = clamp(z + dz, 0, volume.shape[2] - 1)
                    val = volume[vx, vy, vz]
                    sum_x += w * val
                    sum_y += w * val
                    sum_z += w * val
                gx[x, y, z] = sum_x / voxel_size
                gy[x, y, z] = sum_y / voxel_size
                gz[x, y, z] = sum_z / voxel_size
    return gx, gy, gz
```

## Subtopics

### [Central Difference Stencils](9968-central-difference-stencils.md)
Approximates the first-order partial derivative by computing the difference between values at symmetrically offset neighbors, divided by twice the step size. It is distinguished from its siblings by its second-order accuracy ($O(h^2)$ truncation error) compared to the first-order accuracy ($O(h)$) of forward and backward differences, while maintaining the same computational cost of two samples per axis. It serves as the foundational mechanism for higher-order methods and vector field analyses. Choose it when a balance of accuracy and performance is required for smooth fields.

### [Forward Difference Gradient](3362-forward-difference-gradient.md)
Approximates the spatial gradient by subtracting the value at the current voxel from the value at the immediately preceding voxel in each axis direction. It is distinguished from its siblings by its lower computational cost and directional bias, as it only requires one neighbor per axis rather than two. Choose it when minimal data access is prioritized and directional bias is acceptable.

### [4th-Order Finite Difference Gradient](4889-4th-order-finite-difference-gradient.md)
Computes the gradient of a Signed Distance Function (SDF) grid at every voxel center using a 4th-order accurate finite difference scheme. It sets itself apart from standard central differences by using a wider stencil and boundary extrapolation to maintain high-order accuracy even at volume edges. Choose it when smooth normals essential for ray marching and shading are required, particularly for SDF volumes.

### [Neumann Gradient](3342-neumann-gradient.md)
Approximates the local density function using a 3D regression hyperplane fitted to 26 neighboring voxels. Unlike simple finite difference stencils, it produces nearly isotropic gradients with significantly reduced fringing artifacts and smoother appearance in regions of rapid gradient change. Choose it when visual quality and smoothness are prioritized over computational performance, particularly for non-binary volumes where scene-space normal estimation is required.

### [Brick-local Gradient Estimation](8810-brick-local-gradient-estimation.md)
Computes the 3D gradient vector for each voxel in a scalar volume using central differences, specifically optimized for brick-based out-of-core rendering architectures. It avoids indirection lookups in the hot rendering loop by computing gradients relative to the current brick's atlas slot and clamping sample positions to the brick's valid range. Choose it when rendering brick-based volumes and reducing memory bandwidth is critical.

### [Gradient Selection](5889-gradient-selection.md)
A runtime dispatch mechanism that chooses between specific finite difference gradient estimation algorithms, primarily Central Difference and Sobel-based methods, based on a configuration uniform. It is distinguished from its siblings by acting as a selector rather than a standalone estimator, allowing dynamic switching between lower-cost central differences and higher-robustness Sobel filters. Choose it when adapting gradient computation to the noise characteristics and visual requirements of the volume data at runtime.

### [Sobel-based Gradient Estimation](9967-sobel-based-gradient-estimation.md)
Uses fixed convolution kernels to estimate gradients. It is distinguished from simple finite difference stencils by its inherent smoothing properties, which reduce sensitivity to high-frequency noise. Choose it when the volume data is noisy and robustness is prioritized over strict geometric accuracy.

### [Gauss-Seidel with Successive Over-Relaxation (SOR)](6904-gauss-seidel-with-successive-over-relaxation-sor.md)
An iterative numerical method used to solve the steady-state heat equation (Laplace equation $\nabla^2 T = 0$) on a 3D uniform Cartesian grid. It updates grid points using the most recently computed neighbor values and accelerates convergence by applying a relaxation factor $\omega > 1$. Unlike standard Jacobi iteration, SOR utilizes updated values immediately, reducing the number of iterations required to reach a solution. Choose it when computing smooth gradient fields from noisy or sparse data by solving a global optimization problem.

### WENO5 Interpolation
Implementation of nominally fifth-order finite-difference WENO that returns the numerical flux for interpolation [S1]. Given five input values $v_1 = f(x-2dx)$, $v_2 = f(x-dx)$, $v_3 = f(x)$, $v_4 = f(x+dx)$ and $v_5 = f(x+2dx)$, it returns an interpolated value $f(x+dx/2)$ with the special property that $(f(x+dx/2) - f(x-dx/2)) / dx = df/dx(x) + \text{error}$, where the error is fifth-order in smooth regions: $O(dx) \le \text{error} \le O(dx^5)$ [S1]. It uses a scaling factor `scale2` (default 1.0) and five input values for the stencil [S1]. Introduced by Chi-Wang Shu, referenced in 'High Order Finite Difference and Finite Volume WENO Schemes and Discontinuous Galerkin Methods for CFD' (ICASE Report No 2001-11, page 6) and ICASE No 97-65 (Shu, 1997) [S1].

## Background
Finite difference methods are a cornerstone of numerical analysis for approximating derivatives of functions defined on discrete grids [G]. In the context of volume rendering, they were adopted to provide surface normals for Phong shading and other lighting models when the volume data was represented as a dense grid of scalar values without explicit surface extraction [G]. The central difference scheme, being second-order accurate and unbiased, became the standard default due to its balance of accuracy and simplicity [G]. Variants like forward and backward differences are used when boundary conditions or directional bias are acceptable, while higher-order schemes (like 4th-order or WENO) are employed for high-fidelity applications such as SDF rendering or fluid simulation [G]. The method is widely used in scientific visualization, medical imaging, and physics-based rendering pipelines [G].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
