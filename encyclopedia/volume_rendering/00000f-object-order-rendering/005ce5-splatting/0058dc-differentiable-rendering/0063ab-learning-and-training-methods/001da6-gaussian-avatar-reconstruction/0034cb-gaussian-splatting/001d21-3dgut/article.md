# 3DGUT

Also called: 3D Gaussian Splatting with Unscented Transform, 3D Gaussian Uncentred Transform, XPR-3D Gaussian Upsampling Transform, XPR-3DGUT

## Overview
3DGUT is a rendering method that employs an unscented transform to improve the affine approximation of the projection operator, providing more precise Gaussian projection onto the image plane [S1][S3, unofficial]. It uses the same 3D Gaussian primitives as [Gaussian Splatting](946-gaussian-splatting.md) but replaces the Jacobian-based linearization with a sigma-point transformation for projection and evaluates the Gaussian response in 3D along camera rays for per-pixel shading [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Primitive parameters $p$: mean $\mu \in \mathbb{R}^3$, half-extents $s \in \mathbb{R}^3$, unit quaternion $q \in \mathbb{R}^4$, spherical harmonic coefficients $sh$, opacity $o \in \mathbb{R}$.
- Camera parameters: intrinsic matrix (focal lengths $f_x, f_y$, principal point $c_x, c_y$), world-to-camera rotation $R_{w2c}$, translation $t_{w2c}$.
- Configuration: alpha threshold $\alpha_{thresh}$, z-near $z_{near}$.

**Outputs:**
- For each primitive: tile cull data (conic elements, opacity, mean, visibility) and shader data (canonical ray origin, transformation matrix, opacity, color).
- Per-pixel: alpha contribution and validity flag.

### Data Structures
**PrimitiveParams:**
- `mu`: $(3,)$ float, mean in world space.
- `s`: $(3,)$ float, half-extents $(s_x, s_y, s_z)$.
- `q`: $(4,)$ float, unit quaternion for rotation.
- `sh`: $(C, 3)$ float, spherical harmonic coefficients.
- `o`: $(1,)$ float, opacity.

**Tile Cull Data (ptcd) fields for primitive $p$:**
- `con_2D[0,0]`, `con_2D[0,1]`, `con_2D[1,1]`: projected conic elements.
- `o`: opacity.
- `mu_2D`: 2D mean.
- `visible`: boolean flag.

**Shader Data (psd) fields for primitive $p$:**
- `ro_canonical`: flattened ray origin in canonical space.
- `M`: flattened $3 \times 3$ matrix $S^{-1} R^T$.
- `o`: opacity.
- `c`: view-dependent color.

### Steps

1. **Project Function**: Computes the footprint and stores projected data.
    1. Compute world-to-camera projection matrix $R_{w2c}$.
    2. Construct rotation matrix $R$ from quaternion $q$.
    3. Compute sigma points in world space using the Unscented Transform.
    4. Project sigma points to camera space.
    5. Project to pixel space.
    6. Estimate 2D Gaussian mean $\mu_{2D}$ and covariance $\Sigma_{2D}$ from weighted projected points.
    7. Apply low-pass filter to $\Sigma_{2D}$ for antialiasing.
    8. Compute conic 2D from filtered covariance.
    9. Adjust opacity using MipSplatting based on determinant ratio.
    10. Compute extent, radius, and bounding box.
    11. Account for ray offset in bounding box.
    12. Compute tile count.
    13. Compute visibility flag.
    14. Concatenate data into `ptcd` and `psd`.

2. **Tile Cull Function**: Determines which tiles the primitive overlaps.
    1. Adjust tile bounds for ray offset ($tile\_max\_x += 1$, $tile\_max\_y += 1$).
    2. Reuse XPR-3DGS logic for culling.

3. **Pixel Info Function**: Returns ray direction in world space.
    1. Compute canonical ray direction from pixel coordinates.
    2. Rotate to world space.

4. **Evaluate Function**: Computes per-pixel alpha.
    1. Unpack `psd` to get $ro_{canonical}$, $M$, $o$, $c$.
    2. Get ray direction $rd$ from pixel data.
    3. Compute canonical ray direction $rd_{canonical} = M @ rd$.
    4. Compute max response in canonical Gaussian space.
    5. Compute $\alpha = o \cdot response$.
    6. Check validity ($response > 0.0113$).

### Formulas
The depth $\tau$ that maximizes the Gaussian response along a camera ray $r(\tau) = o + \tau d$ is [S2, unofficial]:

$$
\tau_{max} = \frac{d_c^\top \Sigma^{-1} (\mu - o)}{d_c^\top \Sigma^{-1} d_c}
$$

where $o_c = S^{-1} R^T (o - \mu)$ and $d_c = S^{-1} R^T d$ are the ray origin and direction in the canonical space of the Gaussian [S2, unofficial].

The transformation matrix $M$ used in shader data is [S2, unofficial]:
$$
M = \text{diag}(1.0/p.s) @ R^T
$$

The canonical ray origin is [S2, unofficial]:
$$
ro_{canonical} = (M @ (cp - p.\mu)).\text{flatten()}
$$
where $cp = -R_{w2c}[:3,:3]^T @ R_{w2c}[:3,3]$.

### Parameters
- $\alpha_{thresh}$: Alpha threshold for visibility. Typical value: 255 (in 8-bit integer context) or normalized equivalent [S2, unofficial].
- $z_{near}$: Near plane distance for visibility check [S2, unofficial].
- Validity threshold: $0.0113$ for response check [S2, unofficial].

### Edge Cases and Pitfalls
- **Precision**: The unscented transform avoids Jacobian linearization errors, but sigma point selection must be robust [S2, unofficial].
- **Boundaries**: Ray offset adjustment in tile culling is critical for correct tile assignment [S2, unofficial].
- **Degenerate Input**: If determinant of covariance is non-positive, the primitive is marked invisible [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def project(p, cam, view, cfg):
    # Get world to camera projection matrix
    R_w2c = world_2_camera_projection_matrix(view)
    # Construct rotation matrix R from quaternion q
    R = get_R(p.q)
    
    ## Unscented Transform
    # Compute sigma points in world space
    sigma_points = compute_sigma_points(p.mu, p.s, R, cfg)
    # Project sigma points to camera space
    sigma_points_c = vectorize(world_2_cam)(sigma_points, R_w2c)
    mu_c = sigma_points_c[0]
    # Project to pixel space
    sigma_points_2D = vectorize(compute_mu_2D)(sigma_points_c, cam)
    # Estimate mu_2D and cov_2D to approximate 2D Gaussian
    mu_2D, cov_2D = approximate_2D_gaussian(sigma_points_2D, cfg)
    
    det_orig = determinant(cov_2D)
    # Apply low-pass filter to cov_2D (antialiasing)
    cov_2D = apply_low_pass_filter(cov_2D, cfg)
    
    # Compute conic 2D from filtered covariance
    con_2D = compute_conic(cov_2D)
    
    # Adjust opacity using MipSplatting based on determinant ratio
    det = determinant(cov_2D)
    o = p.o * mip_splatting_2D(det_orig, det)
    
    # Compute extent, radius, and bounding box
    ex, ey = compute_extents(cov_2D, det, o, cfg)
    r = compute_radius(ex, ey)
    # Account for ray offset
    aabb = get_bounding_box(mu_2D - 0.5, ex, ey, cfg)
    # Compute tile count
    tc = (aabb.x_max - aabb.x_min) * (aabb.y_max - aabb.y_min)
    
    # Compute view-dependent color c
    c = compute_color(p.mu, p.sh, view)
    
    # Compute visibility flag and adjust tile count
    visible = (det > 0) & (o > cfg.alpha_thresh) & (r > 0) & (tc > 0) & (mu_c[2] > cfg.z_near)
    tc *= float(visible)
    
    # Produce tile_cull_data and shader_data for primitive p
    p_tcd = concat(con_2D[0,0], con_2D[0,1], con_2D[1,1], o, mu_2D, visible)
    
    M = diag(1.0 / p.s) @ R.T
    cp = -R_w2c[:3, :3].T @ R_w2c[:3, 3]
    ro_canonical = (M @ (cp - p.mu)).flatten()
    p_sd = concat(ro_canonical, M.flatten(), o, c)
    
    return ProjectResult(depth=mu_c[2], visible=visible, tile_cull_data=p_tcd, shader_data=p_sd, aabb=aabb, tile_count=tc)

def tile_cull(tile_min, tile_max, p_tcd, cfg):
    # Adjust tile bounds for ray offset
    tile_max[0] += 1 # tile_max_x
    tile_max[1] += 1 # tile_max_y
    # Rest of tile_cull in XPR-3DGS can be reused unchanged
    ...

def pixel_info(px, cam, view):
    # Compute ray direction in pixel space
    dc = [(px[0] + 0.5 - cam.cx) / cam.fx, (px[1] + 0.5 - cam.cy) / cam.fy, 1.0]
    dc /= norm(dc)
    # Rotate to world space
    R_c2w = cam_2_world(view)[:3, :3]
    # Compute ray direction
    rd = R_c2w @ dc
    return PixelInfoResult(rd=rd)

def evaluate(px_data, p_sd):
    ro_canonical, M, o, c = unpack(p_sd)
    rd = px_data[0:3]
    
    # Compute max response in canonical Gaussian space
    rd_canonical = M @ rd
    response = compute_max_response_canonical(ro_canonical, rd_canonical)
    alpha = o * response
    valid = (response > 0.0113)
    
    return EvaluateResult(alpha=alpha, valid=valid, color=c)
```

## Subtopics

### XPR-3DGUT
The implementation of 3DGUT within the XPR framework, demonstrating ease of integration by reusing shared pipeline modules [S2, unofficial].

### MipSplatting
A technique used by 3DGUT to adjust opacity based on the determinant ratio of the projected covariance for antialiasing [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Projection involves computing sigma points and their projections, which is more expensive than the Jacobian-based approach in [Gaussian Splatting](946-gaussian-splatting.md) but provides higher accuracy [S1][S3, unofficial].
- **Memory Complexity**: Similar to [Gaussian Splatting](946-gaussian-splatting.md) as it uses the same primitive structure, but stores additional shader data for canonical space evaluation [S2, unofficial].
- **Strengths**: More precise Gaussian projection onto the image plane by avoiding affine approximation errors [S1][S3, unofficial].
- **Limitations**: Increased computational cost during projection compared to linearization methods [S2, unofficial].
- **When to use**: When high accuracy in Gaussian projection is required, especially for large or highly anisotropic Gaussians [S1][S3, unofficial].
- **When not to use**: When real-time performance is critical and the accuracy gain is negligible [S2, unofficial].

## Background
3DGUT was introduced to address the limitations of affine approximations in Gaussian projection, which can compromise rendering accuracy [S1][S3, unofficial]. It employs the unscented transform to provide a more precise projection of 3D Gaussians onto the image plane [S1][S3, unofficial]. The method is part of the broader effort to improve the fidelity of point-based rendering methods [S1][S3, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] XPR: An Extensible Cross-Platform Point-Based Differentiable Renderer, https://doi.org/10.48550/arxiv.2606.11529, fetched 2026-10-26 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
