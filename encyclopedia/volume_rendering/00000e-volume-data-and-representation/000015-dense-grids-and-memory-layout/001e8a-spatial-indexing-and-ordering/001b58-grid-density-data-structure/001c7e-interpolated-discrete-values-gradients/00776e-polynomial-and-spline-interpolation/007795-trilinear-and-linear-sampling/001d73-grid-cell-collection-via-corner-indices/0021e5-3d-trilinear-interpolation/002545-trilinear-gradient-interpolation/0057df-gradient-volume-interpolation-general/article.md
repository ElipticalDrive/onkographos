# Gradient Volume Interpolation (General)
## Overview
Gradient volume interpolation computes the spatial gradient of a scalar field at arbitrary positions within a 3D grid by interpolating pre-computed gradient values stored in a dedicated texture or volume buffer. It solves the problem of obtaining smooth, continuous surface normals for shading in volume rendering, avoiding the artifacts inherent in finite difference approximations. It differs from [Trilinear Gradient Interpolation](3470-trilinear-gradient-interpolation.md) by providing a general framework for gradient access, where the specific interpolation kernel (e.g., trilinear) is a parameter, and from [Normal Approximation with Finite Differences](9736-normal-approximation-with-finite-differences.md) by relying on pre-stored data rather than on-the-fly differentiation.
## How it works
**Inputs and Outputs:**
- Input: A scalar volume $V$ (for context, though gradients are pre-computed), a gradient volume $G$ storing 3D vectors, and a query position $P = (x, y, z)$.
- Output: The interpolated gradient vector $\mathbf{g} = (g_x, g_y, g_z)$ at $P$.

**Data Structures:**
- Gradient Volume $G$: A 3D grid of size $N_x \times N_y \times N_z$. Each cell stores a 3-component vector (e.g., `float3`). In texture memory, this is often stored as a 3D texture or a 2D texture array. Values are typically normalized or stored as raw derivatives depending on the pipeline.

**Steps:**
1. **Pre-computation (Offline/Setup):** Compute the gradient $\nabla V$ for the scalar volume using finite differences (e.g., central differences). Store these vectors in $G$.
2. **Query:** Given position $P$, identify the 8 surrounding grid corners $(i, j, k)$ to $(i+1, j+1, k+1)$.
3. **Interpolation:** Perform trilinear interpolation on the vector components of $G$ at these 8 corners to obtain $\mathbf{g}_{raw}$.
4. **Normalization:** Normalize the resulting vector to obtain the surface normal $\mathbf{n} = \frac{\mathbf{g}_{raw}}{\|\mathbf{g}_{raw}\|}$. [G] If $\|\mathbf{g}_{raw}\| \approx 0$, return a default normal (e.g., $(0,0,1)$) to avoid division by zero.

**Formula:**
Let $w_x, w_y, w_z$ be the trilinear weights for position $P$ within the unit cube defined by the grid indices. The interpolated gradient is:
$$ \mathbf{g}_{raw} = \sum_{c \in \text{corners}} w_c \cdot G(c) $$
Where $G(c)$ is the gradient vector stored at corner $c$.

**Parameters:**
- Grid resolution: Matches the scalar volume resolution.
- Interpolation kernel: Trilinear is standard for this method.

**Edge Cases and Pitfalls:**
- **Unnormalized Artifacts:** Interpolating a gradient volume without subsequent normalization is generally incorrect, though artifacts are not always visible [S1, book]. The magnitude of the interpolated gradient varies with position within the cell, leading to inconsistent shading intensity.
- **Boundary Conditions:** Gradient values at volume boundaries must be handled carefully during pre-computation to avoid edge artifacts.
- **Precision:** Storing gradients in floating-point textures requires sufficient precision to avoid quantization errors in normal direction.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interpolate_gradient(G, P, grid_dims):
    # P is normalized [0, 1]
    # G is a 3D texture or array of shape (Nx, Ny, Nz, 3)
    
    # 1. Convert P to grid coordinates
    x = P.x * (grid_dims.x - 1)
    y = P.y * (grid_dims.y - 1)
    z = P.z * (grid_dims.z - 1)
    
    # 2. Get corner indices
    ix = int(floor(x))
    iy = int(floor(y))
    iz = int(floor(z))
    
    # 3. Get weights
    wx = x - ix
    wy = y - iy
    wz = z - iz
    
    # 4. Fetch 8 gradient vectors
    # G[ix, iy, iz] returns a vector (gx, gy, gz)
    g000 = G[ix, iy, iz]
    g100 = G[ix+1, iy, iz]
    g010 = G[ix, iy+1, iz]
    g110 = G[ix+1, iy+1, iz]
    g001 = G[ix, iy, iz+1]
    g101 = G[ix+1, iy, iz+1]
    g011 = G[ix, iy+1, iz+1]
    g111 = G[ix+1, iy+1, iz+1]
    
    # 5. Trilinear interpolation of vectors
    # Interpolate along x
    g00 = lerp(g000, g100, wx)
    g01 = lerp(g010, g110, wx)
    g10 = lerp(g001, g101, wx)
    g11 = lerp(g011, g111, wx)
    
    # Interpolate along y
    g0 = lerp(g00, g01, wy)
    g1 = lerp(g10, g11, wy)
    
    # Interpolate along z
    g_raw = lerp(g0, g1, wz)
    
    # 6. Normalize
    norm = length(g_raw)
    if norm < 1e-6:
        return (0, 0, 1) # Default normal
    return g_raw / norm
```
## Subtopics

### Shading Cube Texture
A method using a shading cube texture instead of a gradient volume to eliminate the incorrectness of unnormalized gradient interpolation [S1, book]. It maps the gradient direction to a cube map or similar structure, allowing for normalized lookup without post-interpolation normalization steps. This approach trades memory bandwidth for geometric complexity in the lookup structure.

## Operating characteristics
- **Time Complexity:** $O(1)$ per fragment for texture lookup and interpolation.
- **Memory Complexity:** $O(N_x N_y N_z)$ for storing the gradient volume. This doubles the memory requirement compared to scalar-only rendering.
- **Performance:** Fully-shaded semitransparent and iso-surface volume rendering can achieve 1-4 frames per second for $256^3$ volumes on hardware like the NVidia GeForce3 [S1, book].
- **Strengths:** Smooth shading, consistent normal magnitudes after normalization.
- **Limitations:** Increased memory usage; requires pre-computation of gradients; normalization step adds computational cost.
- **When to use:** When high-quality shading is required and memory bandwidth is sufficient.
- **When not to use:** On hardware with limited texture memory or when brute-force rendering performance is critical and shading quality can be compromised.
## Background
Gradient volume interpolation is a technique used in volume visualization and volume graphics. It was discussed in the context of fully-shaded semitransparent and iso-surface volume rendering on early GPU hardware [S1, book]. Meißner and Guthe proposed using a shading cube texture to address the normalization issues inherent in direct gradient interpolation [S1, book]. The method is part of the broader effort to reduce load for fragment shaders and improve performance on specialized volume rendering hardware like VolumePro500 and VolumePro 1000 [S1, book].
## Sources
[S1] Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
