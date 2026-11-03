# 3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)

Also called: 3DGS Weekend Renderer, Depth-based Splat Sorting, Instanced Gaussian Splatting with Back-to-Front Sorting

## Overview
This method renders trained 3D Gaussian Splatting scenes by projecting 3D Gaussians into 2D screen-space quads, sorting them by camera-space depth on the CPU, and drawing them using instanced rendering with standard alpha blending [S1, unofficial]. It solves the transparency ordering problem inherent in translucent splatting by ensuring fragments are blended from back to front [S1, unofficial]. Unlike the official tile-based CUDA rasterizer which sorts fragments on the GPU, this approach prioritizes implementation simplicity and CPU-side control, making it suitable for display-only renderers or environments where GPU sorting is unavailable [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `gs`: A collection of $M$ 3D Gaussians. Each Gaussian $i$ has:
  - `means`: 3D position $(x, y, z)$ in world space.
  - `covariances`: 3D covariance matrix $\Sigma_{3D}$ (or scale/rotation parameters).
  - `opacities`: Scalar opacity $\alpha_i$.
  - `sh_coefficients`: Spherical Harmonics coefficients for view-dependent color.
- `camera`: View matrix $V$ and projection matrix $P$.
- `image_size`: Width $W$ and Height $H$ in pixels.

**Outputs:**
- Rendered image buffer of size $W \times H \times 3$ (RGB) [S3, unofficial].

### Data Structures
**GpuGaussian Struct** (uploaded to GPU for instancing) [S1, unofficial]:
- `centroid`: `vec3` (12 bytes)
- `opacity`: `float` (4 bytes)
- `scale`: `array<float, 3>` (12 bytes)
- `rotation`: `array<float, 4>` (16 bytes)
- Total size: 44 bytes.

**SplatSortEntry Struct** (used for CPU sorting) [S1, unofficial]:
- `cameraZ`: `float` (4 bytes)
- `splatIndex`: `size_t` (8 bytes)

**Depth Mix Buffer** (alternative JS-based implementation) [S2, unofficial]:
- `depthMix`: A `BigInt64Array` of size `vertexCount`.
- Viewed as `Float32Array` (`floatMix`) and `Uint32Array` (`indexMix`) [S2, unofficial].
- Each splat $j$ occupies two consecutive floats: index at `2*j`, depth at `2*j+1` [S2, unofficial].
- Depth calculation: `depth = 10000 + viewProj[2] * x + viewProj[6] * y + viewProj[10] * z`, where $x, y, z$ are splat center coordinates [S2, unofficial].

### Numbered Steps
1. **Transform and Project Gaussians**: Transform 3D means to camera space: $\mathbf{p}_{cam} = V \mathbf{p}_{world}$ [S3, unofficial]. Project to 2D image plane to get screen coordinates $(u, v)$ [S3, unofficial]. Transform 3D covariances to camera space: $\Sigma_{cam} = R_{view} \Sigma_{world} R_{view}^T$ [S3, unofficial]. Project 3D covariance to 2D using the Jacobian of the perspective projection [G]. Add a minimum epsilon (e.g., 0.3) to the diagonal to stabilize inversion [S3, unofficial].
2. **Frustum Culling and Filtering**: Remove Gaussians outside the camera frustum (image bounds and near/far planes) [S3, unofficial]. Skip Gaussians with opacity below a threshold (e.g., 0.01 or 0.05) [S3, unofficial][S1, unofficial].
3. **CPU Sorting**: Sort the remaining Gaussians by their camera-space depth $z_{cam}$ in increasing order (far-to-near, since camera looks down negative Z) [S1, unofficial]. Create an array of `SplatSortEntry` containing `cameraZ` and original index [S1, unofficial]. Sort using standard library sort (e.g., `std::sort` or JavaScript `Array.sort`) [S1, unofficial][S2, unofficial]. Reorder the `GpuGaussian` data and SH coefficients according to the sorted indices [S1, unofficial].
4. **GPU Rendering Setup**: Disable depth testing: `glDisable(GL_DEPTH_TEST)` [S1, unofficial]. Enable alpha blending: `glEnable(GL_BLEND)` with `glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)` [S1, unofficial].
5. **Vertex Shader (Instanced)**: For each instance $i$ and vertex $v$ of the unit quad: Compute clip space position: $\mathbf{c} = P V \mathbf{p}_{centroid}$, $\mathbf{ndc} = \mathbf{c} / \mathbf{c}.w$ [S1, unofficial]. Compute 2D covariance components with jitter (add 0.3 to diagonals) [S1, unofficial]: $a = \Sigma_{2D}[0][0] + 0.3$, $b = \Sigma_{2D}[1][0]$, $d = \Sigma_{2D}[1][1] + 0.3$. Eigen-decompose the $2 \times 2$ covariance to get eigenvalues $\lambda_1, \lambda_2$ and eigenvectors $\mathbf{e}_1, \mathbf{e}_2$ [S1, unofficial]: $\text{det} = ad - b^2$, $\text{mid} = 0.5(a+d)$, $\text{rad} = \sqrt{\max(\text{mid}^2 - \text{det}, 0.0)}$, $\lambda_1 = \max(\text{mid} + \text{rad}, 0.01)$, $\lambda_2 = \max(\text{mid} - \text{rad}, 0.01)$. $\mathbf{e}_1 = \text{normalize}(b, \lambda_1 - a)$ if $|b| > 10^{-5}$, else $(1,0)$ if $a \ge d$, else $(0,1)$. $\mathbf{e}_2 = (-\mathbf{e}_1.y, \mathbf{e}_1.x)$. Compute 3-sigma half-axes in pixels: $\mathbf{b}_1 = 3\sqrt{\lambda_1} \mathbf{e}_1$, $\mathbf{b}_2 = 3\sqrt{\lambda_2} \mathbf{e}_2$ [S1, unofficial]. Compute vertex offset in NDC: $\Delta \mathbf{pixel} = v.x \mathbf{b}_1 + v.y \mathbf{b}_2$, $\Delta \mathbf{ndc} = \Delta \mathbf{pixel} / (0.5 \times \text{viewport\_size})$ [S1, unofficial]. Set final position: $\mathbf{gl\_Position} = (\mathbf{ndc}.xy + \Delta \mathbf{ndc}, \mathbf{ndc}.z, 1.0)$ [S1, unofficial]. Pass varyings: `splatCoord` (quad corner $v$), `splatColor` (evaluated SH color), `splatOpacity` (sigmoid of stored opacity) [S1, unofficial].
6. **Fragment Shader**: For each fragment with local coordinates $(p_x, p_y)$: Compute power term: $\text{power} = -4.5 (p_x^2 + p_y^2)$ [S1, unofficial]. Discard if `power < -9.0` (approx $e^{-9} \approx 0$) [S1, unofficial]. Compute alpha: $\alpha = \min(0.99, \text{splatOpacity} \times e^{\text{power}})$ [S1, unofficial]. Discard if `alpha < 0.001` [S1, unofficial]. Output color: `FragColor = vec4(splatColor, alpha)` [S1, unofficial].

### Formulas
- **Closed-form eigen-decomposition of 2x2 covariance**: $\text{determinant} = a \cdot d - b^2$, $\text{mid} = 0.5(a + d)$, $\text{radius} = \sqrt{\max(\text{mid}^2 - \text{determinant}, 0.0)}$, $\lambda_1 = \max(\text{mid} + \text{radius}, 0.01)$, $\lambda_2 = \max(\text{mid} - \text{radius}, 0.01)$ [S1, unofficial].
- **Gaussian value at fragment position $(p_x, p_y)$**: $G(p_x, p_y) = \exp(-4.5(p_x^2 + p_y^2))$. Derived from substituting $d_i = 3\sqrt{\lambda_i}p_i$ into $\exp(-d_i^2 / (2\lambda_i))$ [S1, unofficial].
- **Depth packing (JS variant)**: $\text{depth} = 10000 + \text{viewProj}[2] \cdot x + \text{viewProj}[6] \cdot y + \text{viewProj}[10] \cdot z$, where $x, y, z$ are splat center coordinates [S2, unofficial].

### Parameters
- Jitter added to 2D covariance diagonal elements: $0.3$ [S1, unofficial].
- Minimum eigenvalue clamp: $0.01$ [S1, unofficial].
- Alpha clamping maximum: $0.99$ [S1, unofficial].
- Alpha discard threshold: $0.001$ [S1, unofficial].
- Power discard threshold: $-9.0$ [S1, unofficial].
- Quad corners: Unit quad with corners at $(-1,-1)$, $(1,-1)$, $(-1,1)$, $(1,1)$ [S1, unofficial].
- Opacity threshold for culling: $0.01$ or $0.05$ [S3, unofficial][S1, unofficial].

### Edge Cases and Pitfalls
- **Precision issues**: Fragments with `power < -9.0` are discarded to avoid precision issues and unnecessary computation at quad corners [S1, unofficial].
- **Near-transparent edges**: Fragments with `alpha < 0.001` are discarded to handle soft Gaussian edges [S1, unofficial].
- **Minor camera movements**: Cause redundant sorting. Handled by computing dot product of viewProj components 2, 6, 10 against lastProj. If $\text{Math.abs}(\text{dot} - 1) < 0.01$, the function returns early without sorting [S2, unofficial].
- **Pixel centering**: $0.5$ is added to pixel coordinates when computing distance to center the Gaussian within the pixel [S3, unofficial].

### Pseudocode (CPU-Sorted Rendering Loop)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-x))

def cpu_sort_and_prepare(gs_means, gs_opacities, view_matrix, sh_coeffs):
    """
    CPU-side sorting and data preparation for instanced rendering.
    gs_means: (N, 3) world space means
    gs_opacities: (N,) scalar opacities
    view_matrix: (4, 4) view matrix
    sh_coeffs: (N, K) spherical harmonic coefficients
    Returns sorted arrays ready for GPU upload.
    """
    N = gs_means.shape[0]
    
    # 1. Transform means to camera space
    ones = np.ones((N, 1))
    means_hom = np.hstack([gs_means, ones])
    means_cam_hom = (view_matrix @ means_hom.T).T
    means_cam = means_cam_hom[:, :3]
    
    # 2. Filter low-opacity Gaussians
    visible_mask = gs_opacities >= 0.01
    visible_indices = np.where(visible_mask)[0]
    means_cam_vis = means_cam[visible_indices]
    opacities_vis = gs_opacities[visible_indices]
    sh_vis = sh_coeffs[visible_indices]
    
    if len(visible_indices) == 0:
        return np.empty((0, 3)), np.empty((0,)), np.empty((0, 0))
        
    # 3. CPU Sorting (Back-to-Front)
    # Sort by camera Z (increasing = far to near in OpenGL camera space)
    sort_order = np.argsort(means_cam_vis[:, 2])
    
    # 4. Reorder data for instanced drawing
    sorted_means_cam = means_cam_vis[sort_order]
    sorted_opacities = opacities_vis[sort_order]
    sorted_sh = sh_vis[sort_order]
    
    return sorted_means_cam, sorted_opacities, sorted_sh
```

## Subtopics

### [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md)
A GPU-parallel rasterization method that divides the image into tiles (default 32x32 pixels) to manage memory and processing load, assigning Gaussians to tiles based on their axis-aligned bounding boxes [2061-tiled-2d-gaussian-splatting-rasterization.md]. It differs from CPU-sorted instancing by performing sorting and blending within each tile on the GPU, allowing for early ray termination and better cache locality [2061-tiled-2d-gaussian-splatting-rasterization.md]. Choose this for high-performance real-time rendering on modern GPUs.

### [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md)
Renders 3D Gaussian splats as screen-space quads using a pre-sorted order buffer and view-space projection data, handling transparency via alpha blending and discarding fragments behind the camera or with negligible alpha [2075-gaussian-splat-rendering-screen-space-quad.md]. It is a variant of the instanced approach that may use different sorting mechanisms or quad generation strategies [2075-gaussian-splat-rendering-screen-space-quad.md].

### Depth-First Splat Sorting
Sorts the list of staged splats by depth *before* creating per-tile fragments, unlike 3DGS which sorts fragments after splitting [S4, unofficial]. This reduces sorting costs because the number of fragments is 1.5 to 4x higher than visible splats [S4, unofficial]. It uses a 32-bit radix sort with depth as the key, whereas standard 3DGS uses a 48-bit key (tile ID + depth) [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(M \log M)$ for CPU sorting, where $M$ is the number of Gaussians [S1, unofficial]. Rendering is $O(M \cdot A)$ where $A$ is the average area of a splat in pixels, but effectively $O(W \cdot H)$ due to fragment shader execution per pixel covered by splats [G].
- **Memory Complexity:** $O(M)$ for storing Gaussian attributes and sorted indices [G].
- **Performance:** Significantly slower than tile-based GPU rasterization for large scenes due to CPU sorting overhead and lack of early ray termination [S1, unofficial]. Suitable for scenes with $< 10^6$ Gaussians or for debugging/prototyping [S1, unofficial].
- **Strengths:** Simple implementation, no custom GPU kernels required, works on any OpenGL/Vulkan/DirectX renderer with instancing support [S1, unofficial].
- **Limitations:** Poor scaling with Gaussian count, no early termination, sorting latency on every frame [S1, unofficial].

## Background
This method is based on the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" by Kerbl et al. (2023) [S1, unofficial]. It was popularized by the "3D Gaussian Splatting in a Weekend" tutorial, which provided a complete, display-only renderer implementation using standard OpenGL and CPU sorting [S1, unofficial]. It serves as an accessible entry point for understanding 3DGS rendering without the complexity of CUDA-based tile rasterization [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S3] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S4] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31 (unofficial)
