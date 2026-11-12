# Linearly Separable Piecewise Polynomial Interpolation Gradient

Also called: Gradient Estimation via Gradient Filters, Linearly separable filter

## Overview
This method estimates the 3D gradient vector at each voxel in a scalar volume by applying optimized linearly separable filters along each axis, rather than computing gradients via standard finite differences and then interpolating. It is distinguished from sibling methods by performing only one interpolation step per gradient component, which yields higher quality results for illumination calculations where gradients are normalized to compute surface normals [S1, unofficial]. Unlike [Analytic Gradient Computation](2167-analytic-gradient-computation.md) which requires analytic derivatives of the transfer function, or [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) which uses regression over a local neighborhood, this approach relies on fixed coefficients derived from piecewise polynomial interpolation theory [S2].

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar volume $V$ with dimensions $D_x \times D_y \times D_z$, where $V(x,y,z)$ is the scalar value at voxel coordinates $(x,y,z)$.
- Output: A 3D vector field $G$ with dimensions $D_x \times D_y \times D_z$, where $G(x,y,z) = [g_x, g_y, g_z]$ is the estimated gradient.

**Data Structures:**
- The volume $V$ is stored as a contiguous array or 3D grid.
- The gradient field $G$ is stored as a separate array of 3-component floating-point vectors.

**Coefficients:**
The filter uses specific coefficients derived from optimized piecewise polynomial interpolation filters (specifically using 3EF constraints) [S2]. For each axis, the gradient is computed using samples at offsets $-2, -1, +1, +2$. The coefficients are:
- Offset $-2$: $-1/12$
- Offset $-1$: $2/3$
- Offset $+1$: $-2/3$
- Offset $+2$: $1/12$

**Numbered Steps:**
1. Initialize the gradient field $G$ to zeros.
2. Iterate over every voxel $(x, y, z)$ in the volume.
3. Compute the x-component $g_x$ using the linearly separable filter along the x-axis:
   $$g_x = -\frac{1}{12}V(x-2, y, z) + \frac{2}{3}V(x-1, y, z) - \frac{2}{3}V(x+1, y, z) + \frac{1}{12}V(x+2, y, z)$$
4. Compute the y-component $g_y$ using the linearly separable filter along the y-axis:
   $$g_y = -\frac{1}{12}V(x, y-2, z) + \frac{2}{3}V(x, y-1, z) - \frac{2}{3}V(x, y+1, z) + \frac{1}{12}V(x, y+2, z)$$
5. Compute the z-component $g_z$ using the linearly separable filter along the z-axis:
   $$g_z = -\frac{1}{12}V(x, y, z-2) + \frac{2}{3}V(x, y, z-1) - \frac{2}{3}V(x, y, z+1) + \frac{1}{12}V(x, y, z+2)$$
6. Store $G(x,y,z) = [g_x, g_y, g_z]$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_linearly_separable_gradient(volume, dims):
    """
    Computes the gradient field using linearly separable piecewise polynomial interpolation filters.
    
    Args:
        volume: 3D array of shape (Dx, Dy, Dz) containing scalar values.
        dims: Tuple (Dx, Dy, Dz) representing the dimensions of the volume.
        
    Returns:
        gradient_field: 3D array of shape (Dx, Dy, Dz, 3) containing gradient vectors.
    """
    Dx, Dy, Dz = dims
    gradient_field = np.zeros((Dx, Dy, Dz, 3))
    
    # Coefficients for the filter
    # Offsets: -2, -1, +1, +2
    # Weights: -1/12, 2/3, -2/3, 1/12
    
    for x in range(Dx):
        for y in range(Dy):
            for z in range(Dz):
                # Compute gx
                # Handle boundary conditions by clamping or assuming zero outside [G]
                # Note: The source does not specify boundary handling. 
                # A common implementation assumes values outside are 0 or clamps to nearest edge.
                # Here we clamp to nearest edge for safety, though [G] indicates this is an implementation detail.
                
                def get_val(dx, dy, dz):
                    # Clamp coordinates to valid range
                    cx = max(0, min(x + dx, Dx - 1))
                    cy = max(0, min(y + dy, Dy - 1))
                    cz = max(0, min(z + dz, Dz - 1))
                    return volume[cx, cy, cz]

                gx = (-1/12 * get_val(-2, 0, 0) + 
                      2/3 * get_val(-1, 0, 0) - 
                      2/3 * get_val(1, 0, 0) + 
                      1/12 * get_val(2, 0, 0))
                
                gy = (-1/12 * get_val(0, -2, 0) + 
                      2/3 * get_val(0, -1, 0) - 
                      2/3 * get_val(0, 1, 0) + 
                      1/12 * get_val(0, 2, 0))
                
                gz = (-1/12 * get_val(0, 0, -2) + 
                      2/3 * get_val(0, 0, -1) - 
                      2/3 * get_val(0, 0, 1) + 
                      1/12 * get_val(0, 0, 2))
                
                gradient_field[x, y, z] = [gx, gy, gz]
                
    return gradient_field
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The filter requires samples at offsets $\pm 2$. Voxels near the volume boundaries (within 2 voxels of any face) do not have sufficient support. The sources do not specify how to handle these. Implementers must choose a strategy, such as clamping, mirroring, or zero-padding [G].
- **Precision:** The coefficients involve fractions ($1/12, 2/3$). Using single-precision floats may introduce small errors, but this is generally acceptable for visualization.
- **Softness:** The method produces slightly softer edges in regions of rapid change compared to central differences [S2]. This is a characteristic, not a bug, but may be undesirable if sharp edges are required.

## Subtopics

### Gradient Map Generation
Generates a lookup table for gradient magnitudes. This method provides the underlying gradient vectors used to populate such maps. [Gradient Map Generation](1204-gradient-map-generation.md)

### Analytic Gradient Computation
Uses derivatives of the transfer function. This method is data-driven and does not require analytic forms. [Analytic Gradient Computation](2167-analytic-gradient-computation.md)

### Gradient Estimation from Samples
Uses linear regression over a neighborhood. This method is faster and uses fixed filters, whereas regression is more computationally expensive. [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md)

### Precomputed Gradient Volume
Stores precomputed gradients. This method is a technique for computing those gradients. [Precomputed Gradient Volume](2425-precomputed-gradient-volume.md)

### Gradient-based Surface Projection
Uses gradients for surface projection. This method provides the gradient field used in such projections. [Gradient-based Surface Projection](2668-gradient-based-surface-projection.md)

### Per-Region Gradient Averaging
Averages gradients in regions. This method computes per-voxel gradients without averaging. [Per-Region Gradient Averaging](2715-per-region-gradient-averaging.md)

### Gradient Magnitude Computation
Computes the magnitude of the gradient. This method computes the vector components. [Gradient Magnitude Computation](3184-gradient-magnitude-computation.md)

### Scanline Gradient Computation
Computes gradients along scanlines. This method is voxel-wise and separable. [Scanline Gradient Computation](3582-scanline-gradient-computation.md)

### Finite Difference Stencils (Central/Forward/Backward)
Standard finite difference methods. This method uses optimized filters instead of simple central differences. [Finite Difference Stencils (Central/Forward/Backward)](3858-finite-difference-stencils-central-forward-backward.md)

### Second Derivative Computation
Computes Hessian or second derivatives. This method computes first derivatives. [Second Derivative Computation](6353-second-derivative-computation.md)

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of voxels, as each voxel requires a constant number of operations (12 reads and 12 multiplications/additions per voxel).
- **Memory Complexity**: $O(N)$ for storing the gradient field.
- **Performance**: The method is efficient because it is linearly separable, allowing the 3D gradient to be computed as three 1D passes. It is faster than non-separable filters or regression-based methods [S2].
- **Strengths**: Provides high-quality gradients suitable for illumination calculations. It performs only one interpolation step, which is more direct than methods that interpolate gradients after computing them at sample points [S1, unofficial].
- **Limitations**: Produces softer edges than central differences [S2]. Requires handling of boundary conditions which are not specified in the source.
- **When to use**: When high-quality gradient estimation is needed for volume rendering, particularly for illumination and normal calculation.
- **When not to use**: When sharp edges are critical and the softening effect is undesirable, or when memory is extremely constrained and storing a separate gradient field is not feasible.

## Background
The method was proposed by Möller et al. as a methodology for designing linearly separable derivative filters with specific accuracy and continuity characteristics using optimized piecewise polynomial interpolation filters [S2]. The approach is based on the analysis of linearly separable gradient filters by researchers such as those cited in [13] and [26] of the source [S2]. It is presented as an alternative to central differences and regression-based methods in volume rendering contexts [S1, unofficial][S2].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
