# Gaussian Splat Vertex Shader

Also called: FGaussianSplatVS, OpenGL Gaussian Splat Renderer, SplatRenderer

## Overview
The Gaussian Splat Vertex Shader renders Gaussian splats as screen-space quads [S2, unofficial]. It computes the projected 2D covariance and conic parameters for each splat, determines the bounding box in screen space, and passes the necessary data to the fragment shader for alpha blending. It differs from sibling methods by handling the full projection pipeline (view-space to screen-space covariance) and degenerate case culling within the vertex stage, relying on the parent [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) for quad assembly and transparency handling.

## How it works
### Inputs and Outputs
**Inputs:**
- `gaussian_data` (SSBO, binding=0): Packed float array containing splat attributes [S1, unofficial].
- `gaussian_order` (SSBO, binding=1): Integer array `gi[]` mapping instance ID to splat index [S1, unofficial].
- `ViewDataBuffer` (StructuredBuffer): Per-splat view data [S2, unofficial].
- `SortKeysBuffer` (StructuredBuffer<uint>): Pre-computed depth sort keys [S2, unofficial].
- Uniforms: `SplatCount` (uint32), `DebugMode` (uint32), `EnableNanite` (uint32), `wh` (viewport width/height) [S2, unofficial].

**Outputs:**
- Clip-space position of the quad center.
- `vQuadSize`: Screen-space width and height.
- `vConic`: Conic matrix parameters (`vec3`) for the fragment shader.
- `vAlpha`: Base opacity.
- `vColor`: Color data.

### Data Structures
The `gaussian_data` SSBO layout per splat is a packed float array with the following offsets [S1, unofficial]:
- `POS_IDX` = 0: Position (3 floats: x, y, z).
- `ROT_IDX` = 3: Rotation quaternion (4 floats: w, x, y, z).
- `SCALE_IDX` = 7: Scale (3 floats: x, y, z).
- `OPACITY_IDX` = 10: Opacity (1 float).
- `SH_IDX` = 11: Spherical Harmonics coefficients (variable dimension, `sh_dim`).
- Total dimension per splat: $3 + 4 + 3 + 1 + \text{sh\_dim}$ [S1, unofficial].

### Steps
1. **Retrieve Splat Data**: Load position, rotation, scale, and opacity from `gaussian_data` using the index from `gaussian_order` [S1, unofficial].
2. **Compute 3D Covariance**: Construct the 3D covariance matrix $M$ from scale $S$ and rotation $R$ (derived from quaternion $q$): $M = S \cdot R$; $\Sigma_{3D} = M^T M$ [S1, unofficial].
3. **Project to Screen Space**: Compute the 2D covariance $\Sigma_{2D}$ using the Jacobian $J$ of the projection and view matrix $W$: $T = W \cdot J$; $\Sigma_{2D} = T^T \Sigma_{3D} T$ [S1, unofficial].
4. **Regularize Covariance**: Add regularization to the diagonal of $\Sigma_{2D}$: $\Sigma_{2D}[0][0] += 0.3f$; $\Sigma_{2D}[1][1] += 0.3f$ [S1, unofficial].
5. **Compute Conic Matrix**: Calculate the inverse of the 2D covariance components: $\text{det} = \Sigma_{2D}[0][0] \cdot \Sigma_{2D}[2][2] - \Sigma_{2D}[0][2]^2$; $\text{det\_inv} = 1.0 / \text{det}$; $\text{conic} = (\Sigma_{2D}[2][2] \cdot \text{det\_inv}, -\Sigma_{2D}[0][2] \cdot \text{det\_inv}, \Sigma_{2D}[0][0] \cdot \text{det\_inv})$ [S1, unofficial].
6. **Determine Quad Size**: Calculate screen-space width and height: $\text{quadwh\_scr} = (3.0 \cdot \sqrt{\Sigma_{2D}[0][0]}, 3.0 \cdot \sqrt{\Sigma_{2D}[2][2]})$ [S1, unofficial].
7. **Clamp Quad Size**: Prevent giant splats: $\text{quadwh\_scr} = \min(\text{quadwh\_scr}, \text{wh} \cdot 0.5)$ [S1, unofficial].
8. **Cull Degenerate Splats**: Discard if max scale $> 1.0$, anisotropy ratio $> 50$, or determinant $\le 0$ [S1, unofficial].
9. **Pass Data to Fragment Shader**: Output `vQuadSize`, `vConic`, `vAlpha`, and `vColor`. If `DebugMode` is 1, color is set to cluster colors (Nanite-style debug). If `EnableNanite` is 1, Nanite LOD compaction may be applied [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z

# Inputs
gaussian_data: list[float]
gaussian_order: list[int]
view_data: list
sort_keys: list[int]
splat_count: int
debug_mode: int
enable_nanite: int
wh: tuple[float, float]

# Constants
SH_C0: float = 0.28209479177387814
REGULARIZATION: float = 0.3
MAX_SCALE: float = 1.0
MAX_ANISOTROPY: float = 50.0

# Offsets
POS_IDX = 0
ROT_IDX = 3
SCALE_IDX = 7
OPACITY_IDX = 10
SH_IDX = 11

def main(instance_id: int) -> dict:
    # 1. Retrieve Splat Index
    splat_idx = gaussian_order[instance_id]

    # 2. Load Splat Data
    base = splat_idx * (3 + 4 + 3 + 1 + sh_dim)
    pos = gaussian_data[base + POS_IDX : base + POS_IDX + 3]
    quat = gaussian_data[base + ROT_IDX : base + ROT_IDX + 4]
    scale = gaussian_data[base + SCALE_IDX : base + SCALE_IDX + 3]
    alpha = gaussian_data[base + OPACITY_IDX]
    sh_coeffs = gaussian_data[base + SH_IDX : base + SH_IDX + sh_dim]

    # 3. Compute 3D Covariance
    R = quaternion_to_rotation_matrix(quat)
    M = [[scale[i] * R[i][j] for j in range(3)] for i in range(3)]
    cov3d = transpose(M) * M

    # 4. Project to Screen Space
    T = mat_mul(W, J)
    cov2d = transpose(T) * transpose(cov3d) * T

    # 5. Regularize Covariance
    cov2d[0][0] += REGULARIZATION
    cov2d[1][1] += REGULARIZATION

    # 6. Compute Conic Matrix
    det = cov2d[0][0] * cov2d[2][2] - cov2d[0][2] * cov2d[0][2]
    if det <= 0.0:
        return {"discard": True}
    det_inv = 1.0 / det
    conic = Vec3(cov2d[2][2] * det_inv, -cov2d[0][2] * det_inv, cov2d[0][0] * det_inv)

    # 7. Determine Quad Size
    quadwh_scr = Vec3(3.0 * math.sqrt(cov2d[0][0]), 3.0 * math.sqrt(cov2d[2][2]), 0.0)

    # 8. Clamp Quad Size
    quadwh_scr.x = min(quadwh_scr.x, wh[0] * 0.5)
    quadwh_scr.y = min(quadwh_scr.y, wh[1] * 0.5)

    # 9. Cull Degenerate Splats
    max_scale = max(scale)
    min_scale = min(scale)
    anisotropy = max_scale / min_scale if min_scale > 0 else float('inf')
    if max_scale > MAX_SCALE or anisotropy > MAX_ANISOTROPY:
        return {"discard": True}

    # 10. Pass Data to Fragment Shader
    vQuadSize = (quadwh_scr.x, quadwh_scr.y)
    vConic = (conic.x, conic.y, conic.z)
    vAlpha = alpha
    vColor = compute_color(sh_coeffs, SH_C0)

    if debug_mode == 1:
        vColor = cluster_color(splat_idx)
    if enable_nanite == 1:
        # Apply Nanite LOD compaction if enabled
        pass

    clip_pos = compute_clip_space_position(pos)
    return {"clip_pos": clip_pos, "vQuadSize": vQuadSize, "vConic": vConic, "vAlpha": vAlpha, "vColor": vColor}
```

### Parameters
- `SH_C0`: 0.28209479177387814f [S1, unofficial].
- `Regularization`: 0.3f [S1, unofficial].
- `Max Scale`: 1.0 [S1, unofficial].
- `Max Anisotropy`: 50 [S1, unofficial].
- `DebugMode`: 0=off, 1=cluster colors (Nanite-style debug) [S2, unofficial].
- `EnableNanite`: 1=Nanite enabled, 0=disabled [S2, unofficial].

### Edge Cases and Pitfalls
- **Degenerate Splats**: Splats with zero determinant in 2D covariance or extreme anisotropy must be discarded to prevent rendering artifacts [S1, unofficial].
- **Giant Splats**: Quad size must be clamped to prevent splats from covering the entire screen [S1, unofficial].
- **Numerical Stability**: Regularization is added to the covariance diagonal to prevent division by zero or negative determinants [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per splat [G].
- **Memory Complexity**: $O(N)$ for the `gaussian_data` and `gaussian_order` buffers [G].
- **Strengths**: Efficient culling of degenerate splats, robust handling of projection and regularization [G].
- **Limitations**: Relies on pre-sorted order buffer from parent method [S2, unofficial]. Performance depends on the efficiency of the Jacobian computation [G].
- **When to Use**: When rendering large numbers of Gaussian splats with high performance requirements [G].
- **When Not to Use**: When dynamic updates to splat attributes are frequent and require frequent buffer updates [G].

## Background
The Gaussian Splat Vertex Shader is implemented in various open-source projects such as `dust3r-3dgs` [S1, unofficial] and `OpenSplat4D` [S2, unofficial]. It operates as a core component of the GPU rendering pipeline for differentiable Gaussian splatting, handling per-splat projection and culling before fragment shading.

## Sources
[S1] SpaceCollectors/dust3r-3dgs, https://github.com/SpaceCollectors/dust3r-3dgs, fetched 2026-10-27 (unofficial)
[S2] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
