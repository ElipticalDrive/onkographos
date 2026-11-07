# Distance Field Representations and Sampling

Also called: Signed Distance Field (SDF) Sampling, Distance Texture Ray Marching, Probabilistic Directed Distance Fields (PDDFs)

## Overview
This family covers the representation of geometry and volumetric data as distance fields and the methods for sampling them during ray marching. It solves the problem of efficiently querying implicit surfaces and volumetric properties without explicit polygonal meshes. It is distinguished from sibling methods by its reliance on scalar or vector distance metrics to drive step sizes and surface detection, rather than opacity accumulation or Gaussian splatting.

## Shared principles
The core principle is the evaluation of a function $f(\mathbf{p})$ that maps a 3D point $\mathbf{p}$ to a distance value. In the context of this family, the "distance" may be Euclidean (SDF), directed (ray-based), or probabilistic. The sampling process involves querying this function at specific points along a ray or in world space to determine geometry presence, surface location, or volumetric density.

## Subtopics

### [Bounded 3D Distance Texture Ray Marching](1115-bounded-3d-distance-texture-ray-marching.md)
This method renders implicit surfaces defined by a pre-computed 3D signed distance texture. It solves the problem of invalid or noisy distance values at the edges of a finite texture volume by defining an inner "green" region for safe lookups and an outer bounding box that acts as a hard stop for the ray marcher, preventing the algorithm from sampling invalid data. Unlike primitive SDFs, this method allows complex mesh-derived data (e.g., the Stanford Bunny) to be integrated into ray marching scenes using standard boolean operators like union.

The algorithm uses a 3D texture (e.g., $192 \times 192 \times 192$) with an inner valid region of $128 \times 128 \times 128$. It employs parameters `textureBoundary` (half-dimensions of the inner valid region, e.g., $(0.5, 0.5, 0.5)$) and `textureShell` (thickness of the boundary region, e.g., $(0.05, 0.05, 0.05)$).

The distance function determines if the query point $x$ is inside the valid texture region or outside it:
1.  **Calculate Distance to Inner Boundary:** Compute the vector $d_i$ representing the distance from $x$ to the inner valid region box, adjusted by the shell thickness:
    $$d_i = |x| - \text{textureBoundary} + \text{textureShell}$$
    Here, $|x|$ denotes the component-wise absolute value of $x$.
2.  **Calculate Distance to Outer Boundary:** If the point is outside the inner region, the distance to the outer bounding box is calculated. This serves as the SDF value for points outside the texture volume, ensuring the ray marcher stops at the texture boundary:
    $$res = \| \max(d_i + \text{textureShell}, 0) \|$$
    The notation $\| \cdot \|$ refers to the Euclidean norm. The term $\max(d_i + \text{textureShell}, 0)$ ensures that if the point is inside the shell region, the distance contribution is zeroed out before taking the norm.
3.  **Check for Inner Region Entry:** Determine if the point $x$ is inside the inner valid region (the "green box"):
    $$c = \max(d_{ix}, \max(d_{iy}, d_{iz}))$$
    If $c < 0$, the point is inside the inner valid region.
4.  **Lookup Distance:** If the point is inside the inner region ($c < 0$):
    *   Adjust the point coordinates to align with the texture coordinate space:
        $$x = x + \text{textureBoundary}$$
    *   Perform the texture lookup:
        $$res = \text{lookupTexture}(\text{texture}, x)$$
    *   Return $res$.
5.  **Return Outer Distance:** If the point is not inside the inner region (i.e., $c \ge 0$), return the distance to the outer bounding box calculated in Step 2.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounded_3d_texture_sdf(x, texture, texture_boundary=(0.5, 0.5, 0.5), texture_shell=(0.05, 0.05, 0.05)):
    # Step 1: Compute di = |x| - textureBoundary + textureShell (component-wise)
    abs_x = (abs(x[0]), abs(x[1]), abs(x[2]))
    di = (
        abs_x[0] - texture_boundary[0] + texture_shell[0],
        abs_x[1] - texture_boundary[1] + texture_shell[1],
        abs_x[2] - texture_boundary[2] + texture_shell[2]
    )
    
    # Step 2: Compute fallback distance res = ||max(di + textureShell, 0)||
    adjusted_di = (
        max(di[0] + texture_shell[0], 0.0),
        max(di[1] + texture_shell[1], 0.0),
        max(di[2] + texture_shell[2], 0.0)
    )
    res = (adjusted_di[0]**2 + adjusted_di[1]**2 + adjusted_di[2]**2)**0.5
    
    # Step 3: Check inner region entry c = max(dix, max(diy, diz))
    c = max(di[0], max(di[1], di[2]))
    
    # Step 4: If inside inner region (c < 0), lookup texture
    if c < 0:
        x_adj = (x[0] + texture_boundary[0], x[1] + texture_boundary[1], x[2] + texture_boundary[2])
        # Standard 3D texture fetch (nearest-neighbor sampling)
        res = texture[int(x_adj[0] * texture.shape[0]), int(x_adj[1] * texture.shape[1]), int(x_adj[2] * texture.shape[2])]
        
    # Step 5: Return result
    return res
```

**Edge Cases and Pitfalls**
*   **Invalid Boundary Data:** Texture samples outside the green box but inside the blue box (the boundary region) may contain invalid distance values. The algorithm remedies this by disregarding that region for lookups and using the red box distance instead.
*   **Scale Mismatch:** The distance returned by the texture lookup must be in accordance with the scale of the scene; otherwise, an appropriate bound must be applied.
*   **Normal Estimation:** When estimating normals (e.g., via finite differences), the epsilon value must be adjusted to account for the voxel spacing in the texture. If the value is too small, the representation may appear "grainy".
*   **Mesh Holes:** For accurate distance field generation from meshes (e.g., Stanford Bunny), holes in the mesh must be closed before conversion to a 3D distance texture.

### Sampler Abstraction
#### definition
- [S1, unofficial] A trait-based abstraction for sampling distance fields defined in terms of specific Distance metrics. It allows samplers to abstract sampling across multiple different Distance metrics.
#### step
- [S1, unofficial] Implementation for ScalarSource: If S implements ScalarSource, Sampler implements Sample<Signed>. The sample method calls self.source.sample_scalar(p).
- [S1, unofficial] Implementation for VectorSource: If S implements VectorSource, Sampler implements Sample<Directed>. The sample method calls self.source.sample_vector(p).
- [S1, unofficial] Implementation for HermiteSource: If S implements HermiteSource, Sampler implements sample_normal method which calls self.source.sample_normal(p).
#### layout
- [S1, unofficial] Sampler struct contains: source (&'a S). Implements Sample<D> trait for Signed and Directed distance types based on source implementing ScalarSource or VectorSource.

### Distance to Value Function
#### definition
- [S2, unofficial] A ray function that computes the distance along the ray until a specific scalar value σ is encountered. Useful to emphasize depth where specific tissue is located.

### Probabilistic Directed Distance Fields (PDDFs)
#### definition
- [S3] A variant of DDFs that differentiably models the inherent discontinuities in the underlying field.
- [S3] PDDFs are constructed to handle the discontinuities that make standard DDFs difficult to train with differentiable neural networks.

## Background
Distance fields have been used in computer graphics for decades for shape representation and rendering. The specific technique of bounded 3D distance texture ray marching is documented in open-source implementations such as `isosurface` by Tristam MacDonald [S1, unofficial]. The concept of "Distance to Value" functions in the context of volume visualization, particularly for medical imaging like CT scans, is described in lecture materials from the University of Illinois Urbana-Champaign [S2, unofficial]. More recently, Probabilistic Directed Distance Fields (PDDFs) have been introduced to enable differentiable rendering by modeling discontinuities in the field, allowing for efficient differentiable rendering and 3D reconstruction [S3].

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S2] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S3] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
