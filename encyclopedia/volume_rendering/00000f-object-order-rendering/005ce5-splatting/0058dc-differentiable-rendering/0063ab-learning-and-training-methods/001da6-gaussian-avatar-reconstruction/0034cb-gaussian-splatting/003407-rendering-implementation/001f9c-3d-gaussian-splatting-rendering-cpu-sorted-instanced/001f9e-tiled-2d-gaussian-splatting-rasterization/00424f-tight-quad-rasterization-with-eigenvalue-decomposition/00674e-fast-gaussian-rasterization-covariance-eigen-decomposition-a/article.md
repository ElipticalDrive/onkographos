# Fast Gaussian Rasterization (Covariance Eigen-decomposition and Quad Emission)
Also called: dendenxu/fast-gaussian-rasterization

## Overview
A GPU-accelerated rasterization method for 3D Gaussian splats that projects the 3D covariance to 2D, computes eigenvalues/vectors to determine basis vectors, and emits a tight quad per splat to the fragment shader for alpha-based blending [S1, unofficial]. It differs from software-based CUDA rasterization by leveraging the fixed-function OpenGL rasterizer via a vertex-geometry-fragment shader pipeline, offloading clipping, interpolation, and blending from software to hardware [S1, unofficial]. This approach reduces CPU/GPU synchronization overhead and explicit sorting stalls, achieving 5-10x speedup over the original software CUDA rasterizer for high pixel-to-point ratios [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians, each defined by a mean position, a 3D covariance matrix (or its decomposition), and color/opacity attributes.
*   **Output:** A 2D image with alpha-blended colors.

**Data Structures:**
The method relies on standard OpenGL shader inputs/outputs.
*   **Geometry Shader Inputs:**
    *   `in vec2 basisVector0[]`: First basis vector scaled by $\sqrt{8}$ standard deviations.
    *   `in vec2 basisVector1[]`: Second basis vector scaled by $\sqrt{8}$ standard deviations.
    *   `in vec4 gColor[]`: Color and opacity of the splat.
    *   `in float gDepth[]`: Depth of the splat center.
*   **Geometry Shader Outputs:**
    *   `out vec2 vPosition`: Normalized position within the quad (-1 to 1).
    *   `flat out vec4 vColor`: Interpolated color and opacity.
    *   `flat out float vDepth`: Interpolated depth.
*   **Fragment Shader Uniforms:**
    *   `uniform bool useDepth`: Flag for depth visualization mode.
    *   `uniform bool solidMode`: Flag for solid opacity mode.
    *   `uniform bool edgeMode`: Flag for edge visualization mode.
    *   `uniform float eight`: Constant value 8.0.
    *   `uniform float minAlpha`: Minimum opacity threshold for discard (Default: 0.004).
    *   `uniform float maxAlpha`: Maximum opacity cap (Default: 0.99).
    *   `uniform float edgeThreshold`: Threshold for edge mode (Default: 0.025).

**Steps:**

1.  **Project Covariance and Compute Eigenvalues:**
    In the vertex shader (or pre-processing step), extract the 2D covariance components from the projected covariance matrix `cov2Dv`:
    $$ a = \text{cov2Dv}.x $$
    $$ d = \text{cov2Dv}.z $$
    $$ b = \text{cov2Dv}.y $$
    
    Compute the determinant $D$:
    $$ D = a \cdot d - b^2 $$
    
    Compute the trace and the term for eigenvalue calculation:
    $$ \text{trace} = a + d $$
    $$ \text{traceOver2} = 0.5 \cdot \text{trace} $$
    $$ \text{term2} = \sqrt{\max(0.1, \text{traceOver2}^2 - D)} $$
    
    Compute the eigenvalues $\lambda_0$ and $\lambda_1$ (where $\lambda_0 \ge \lambda_1$):
    $$ \lambda_0 = \text{traceOver2} + \text{term2} $$
    $$ \lambda_1 = \text{traceOver2} - \text{term2} $$

2.  **Compute Eigenvectors:**
    Compute the first eigenvector $\mathbf{e}_0$:
    $$ \mathbf{e}_0 = \text{normalize}(\text{vec2}(b, \lambda_0 - a)) $$
    
    Derive the orthogonal second eigenvector $\mathbf{e}_1$:
    $$ \mathbf{e}_1 = \text{vec2}(\mathbf{e}_0.y, -\mathbf{e}_0.x) $$

3.  **Compute Basis Vectors:**
    Scale the eigenvectors by $\sqrt{8}$ standard deviations. The factor $\sqrt{8}$ is used instead of the more common 3 standard deviations to eliminate more of the splat with very low opacity [S1, unofficial].
    $$ \text{basisVector0} = \mathbf{e}_0 \cdot \min(\sqrt{8} \cdot \sqrt{\lambda_0}, \text{maxScreenSpaceSplatSize}) $$
    $$ \text{basisVector1} = \mathbf{e}_1 \cdot \min(\sqrt{8} \cdot \sqrt{\lambda_1}, \text{maxScreenSpaceSplatSize}) $$
    *Note: `maxScreenSpaceSplatSize` is a hardware-specific cap to prevent excessively large quads.*

4.  **Validate and Prune:**
    Check for illegal covariance matrices or degenerate splats. If any of the following conditions are met, prune the splat by setting `gColor.a = 0.0` and returning early:
    *   $D \le 0.0$
    *   $a \le 0.0$
    *   $d \le 0.0$
    *   $\lambda_0 \le 0.01$
    *   $\lambda_1 \le 0.01$
    *   $\lambda_0 < \lambda_1$ (should not happen with correct ordering)
    *   Aspect ratio $\frac{\lambda_0}{\lambda_1} > 10000.0$

5.  **Emit Quad in Geometry Shader:**
    For each of the 4 corners of the quad, defined by `vPosition` in $\{-1, 1\} \times \{-1, 1\}$:
    
    *   **Discard Check:** If `gColor[0].a < discardAlpha` (0.0001), return without emitting vertices [S1, unofficial].
    
    *   **Compute NDC Offset:**
        $$ \text{ndcOffset} = (\text{vPosition}.x \cdot \text{basisVector0} + \text{vPosition}.y \cdot \text{basisVector1}) \cdot \text{basisViewport} \cdot 2.0 $$
        Here, `basisViewport` scales the basis vectors to Normalized Device Coordinates (NDC) based on the viewport dimensions.
    
    *   **Set Position:**
        $$ \text{gl\_Position} = \text{vec4}(\text{ndcCenter}.xy + \text{ndcOffset}, \text{ndcCenter}.z, 1.0) $$
        where `ndcCenter` is the projected center of the Gaussian in NDC space.
    
    *   **Pass Attributes:**
        Scale `vPosition` by $\sqrt{8}$ and pass through `vColor` (from `gColor`) and `vDepth` (from `gDepth`).

6.  **Apply Mode-Specific Overrides (Geometry Shader):**
    *   If `solidMode` is true, set opacity = 1.0.
    *   If `useDepth` is true, set color = `vec3(vDepth)`.
    *   If `edgeMode` is true and $\exp(\text{power}) < \text{edgeThreshold}$, set color = `vec3(0.0, 0.0, 0.0)` and opacity = 1.0.

7.  **Fragment Shader Processing:**
    *   **Alpha Clipping:** If the interpolated alpha `vColor.a < minAlpha` (0.004), discard the fragment.
    *   **Alpha Capping:** If `vColor.a > maxAlpha` (0.99), cap it to `maxAlpha`.
    *   **Blending:** Perform standard alpha blending using the fragment's color and alpha against the existing framebuffer content.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Pre-processing Step
def compute_eigen_and_basis(cov2Dv, maxScreenSpaceSplatSize):
    """
    cov2Dv: vec4 (x=a, y=b, z=d, w=unused)
    Returns: basisVector0, basisVector1, eigenValue0, eigenValue1
    """
    a = cov2Dv.x
    d = cov2Dv.z
    b = cov2Dv.y
    
    # Determinant
    D = a * d - b * b
    
    # Eigenvalues
    trace = a + d
    traceOver2 = 0.5 * trace
    # Use max(0.1, ...) to avoid numerical instability for very small traces
    term2 = sqrt(max(0.1, traceOver2 * traceOver2 - D))
    eigenValue0 = traceOver2 + term2
    eigenValue1 = traceOver2 - term2
    
    # Eigenvectors
    # e0 = normalize(vec2(b, eigenValue0 - a))
    # Handle potential zero vector if eigenValue0 == a and b == 0
    if abs(b) < 1e-6 and abs(eigenValue0 - a) < 1e-6:
        eigenVector0 = vec2(1.0, 0.0)
    else:
        eigenVector0 = normalize(vec2(b, eigenValue0 - a))
    
    # e1 is orthogonal to e0
    eigenVector1 = vec2(eigenVector0.y, -eigenVector0.x)
    
    # Basis vectors scaled by sqrt(8)
    sqrt8 = sqrt(8.0)
    scale0 = min(sqrt8 * sqrt(eigenValue0), maxScreenSpaceSplatSize)
    scale1 = min(sqrt8 * sqrt(eigenValue1), maxScreenSpaceSplatSize)
    
    basisVector0 = eigenVector0 * scale0
    basisVector1 = eigenVector1 * scale1
    
    return basisVector0, basisVector1, eigenValue0, eigenValue1

# Geometry Shader Step
def emit_quad(basisVector0, basisVector1, gColor, gDepth, ndcCenter, basisViewport, discardAlpha):
    """
    Emits 4 vertices for the quad.
    Returns list of vertex attributes: (vPosition, vColor, vDepth)
    """
    vertices = []
    
    # Check for early discard
    if gColor.a < discardAlpha:
        return vertices
    
    # Corners of the quad in local space {-1, 1} x {-1, 1}
    corners = [(-1, -1), (1, -1), (1, 1), (-1, 1)]
    
    for vx, vy in corners:
        # NDC Offset calculation
        # ndcOffset = (vx * basisVector0 + vy * basisVector1) * basisViewport * 2.0
        offset_x = (vx * basisVector0.x + vy * basisVector1.x) * basisViewport * 2.0
        offset_y = (vx * basisVector0.y + vy * basisVector1.y) * basisViewport * 2.0
        
        # Final NDC Position
        final_x = ndcCenter.x + offset_x
        final_y = ndcCenter.y + offset_y
        final_z = ndcCenter.z
        final_w = 1.0
        
        # vPosition is scaled by sqrt(8) for fragment shader evaluation
        vPosition = vec2(vx * sqrt(8.0), vy * sqrt(8.0))
        
        # Pass through color and depth
        vColor = gColor
        vDepth = gDepth
        
        vertices.append((vPosition, vColor, vDepth))
        
    return vertices

# Fragment Shader Step
def fragment_shader(vPosition, vColor, vDepth, useDepth, solidMode, edgeMode, edgeThreshold, minAlpha, maxAlpha):
    """
    Returns final color and alpha.
    """
    color = vColor.rgb
    alpha = vColor.a
    
    # Mode overrides
    if solidMode:
        alpha = 1.0
    elif useDepth:
        color = vec3(vDepth)
    elif edgeMode:
        # Assuming 'power' is computed from vPosition magnitude in the actual shader
        # For this pseudocode, we assume edge check is done or power is available
        # If edgeMode and exp(power) < edgeThreshold:
        #    color = vec3(0.0)
        #    alpha = 1.0
        pass
        
    # Alpha capping
    if alpha > maxAlpha:
        alpha = maxAlpha
        
    # Alpha clipping
    if alpha < minAlpha:
        discard_fragment()
        
    return color, alpha
```

## Subtopics

### [Gaussian Rasterization via OpenGL Geometry Shader](19838-gaussian-rasterization-via-opengl-geometry-shader.md)
This method renders 3D Gaussian splats as screen-space quads by leveraging the fixed-function OpenGL rasterizer via a vertex-geometry-fragment shader pipeline, offloading clipping, interpolation, and blending from software to hardware. It differs from CUDA-based software rasterization by reducing CPU/GPU synchronization overhead and explicit sorting stalls, achieving 5-10x speedup over the original software CUDA rasterizer for high pixel-to-point ratios.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ splats, dominated by the geometry shader emission and fragment shader blending. The eigen-decomposition is $O(1)$ per splat.
*   **Memory Complexity:** $O(N)$ for storing splat attributes and intermediate buffers.
*   **Measured Performance:** Achieves 5-10x speedup over the original software CUDA rasterizer for high pixel-to-point ratios [S1, unofficial].
*   **Strengths:**
    *   Leverages hardware rasterization, reducing CPU/GPU sync overhead.
    *   Tight quad bounding box ($\sqrt{8}$ std devs) reduces overdraw compared to larger bounding boxes.
    *   Simple eigen-decomposition avoids iterative solvers.
*   **Limitations:**
    *   Requires OpenGL support for geometry shaders (though widely supported).
    *   Pruning conditions (aspect ratio, eigenvalue thresholds) may discard valid splats in edge cases if thresholds are too aggressive.
    *   Not directly compatible with Vulkan/Metal without adaptation (geometry shader stage differences).
*   **When to Use:** Real-time rendering of 3D Gaussian Avatars where performance is critical and OpenGL is available.
*   **When Not to Use:** When maximum compatibility with non-OpenGL APIs is required, or when the scene has extremely low pixel-to-point ratios where software sorting might be more efficient.

## Background
This method is part of the "Fast Gaussian Rasterization" implementation by dendenxu, which provides an efficient CUDA/OpenGL hybrid approach for rendering 3D Gaussian Splatting [S1, unofficial]. It builds on the mathematical foundation of 3D Gaussian Splatting [S1, unofficial] and optimizes the rasterization step specifically for real-time performance by utilizing the GPU's geometry shader to emit tight bounding quads.

## Sources
[S1] dendenxu/fast-gaussian-rasterization, https://github.com/dendenxu/fast-gaussian-rasterization, fetched 2026-10-26 (unofficial)
