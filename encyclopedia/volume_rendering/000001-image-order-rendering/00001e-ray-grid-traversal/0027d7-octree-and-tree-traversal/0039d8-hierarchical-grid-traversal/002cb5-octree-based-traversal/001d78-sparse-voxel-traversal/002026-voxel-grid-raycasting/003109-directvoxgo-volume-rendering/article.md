# DirectVoxGO Volume Rendering

Also called: DirectVoxGO, Voxel Grid Neural Rendering

## Overview
DirectVoxGO is a volume rendering method that accelerates neural radiance field training and inference by sampling points directly on a structured voxel grid rather than uniformly along rays [S1, unofficial]. It solves the problem of inefficient ray marching in sparse scenes by using a mask cache to skip free space and a direct lookup of density and color features from the voxel grid [S1, unofficial]. It sets itself apart from standard NeRF by replacing uniform sampling with grid-based point selection and from other sparse voxel methods by using a dense voxel grid with a binary mask cache for acceleration [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins `rays_o` ([N, 3]), ray directions `rays_d` ([N, 3]), view directions `viewdirs` ([N, 3]), near/far distances, and rendering parameters [S1, unofficial].
*   **Outputs:** Rendered RGB images `rgb_marched` ([N, 3]), accumulated transmittance `alphainv_last` ([N]), and intermediate weights `weights` ([M]) [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D tensor storing density and color features, defined by `xyz_min` and `xyz_max` bounding boxes and a resolution `num_voxels` [S1, unofficial].
*   **Mask Cache:** A binary tensor indicating which voxels contain geometry (true) or are free space (false) [S1, unofficial].
*   **Ray Points:** `ray_pts` ([M, 3]) stores the 3D coordinates of sampled points. `ray_id` ([M]) maps each point to its originating ray. `step_id` ([M]) maps each point to its step index along the ray [S1, unofficial].

**Parameters:**
*   `stepsize`: Number of voxels per sample step [S1, unofficial].
*   `fast_color_thres`: Threshold for filtering points based on alpha or weight values; if > 0, enables optimization [S1, unofficial].
*   `voxel_size_ratio`: Ratio used to compute interval for density-to-alpha conversion [S1, unofficial].
*   `rgbnet_direct`: Boolean flag determining if the RGB network outputs direct RGB or residual features [S1, unofficial].

**Steps:**

1.  **Ray Setup:** Compute ray origins and directions from camera parameters. If NDC (Normalized Device Coordinates) is enabled, transform rays to NDC space [S1, unofficial].
    *   Ray direction computation (for `inverse_y=False`, `flip_y=False`, `mode='center'`):
        $$ \text{dirs} = \left[ \frac{i - K_{0,2}}{K_{0,0}}, \frac{j - K_{1,2}}{K_{1,1}}, 1 \right] $$
    *   NDC ray transformation:
        $$ o_0 = -\frac{1}{W/(2 \cdot \text{focal})} \frac{o_x}{o_z}, \quad d_0 = -\frac{1}{W/(2 \cdot \text{focal})} \left(\frac{d_x}{d_z} - \frac{o_x}{o_z}\right) $$

2.  **Sample Points:** Calculate step distance `stepdist = stepsize * voxel_size`. Sample points along rays within the bounding box `[xyz_min, xyz_max]`. Filter points outside the bounding box [S1, unofficial].
    *   Input: `rays_o`, `rays_d`, `near`, `far`, `stepdist`.
    *   Output: `ray_pts`, `ray_id`, `step_id`, `mask_outbbox`.

3.  **Free Space Culling:** Query the mask cache for the sampled points. Filter out points where the mask is false (free space) [S1, unofficial].

4.  **Density Query:** Query the voxel grid for density values at the remaining points [S1, unofficial].

5.  **Alpha Conversion:** Convert density to alpha using the `Raw2Alpha` function:
    $$ \alpha = 1 - (1 + \exp(\text{density} + \text{shift}))^{-\text{interval}} $$
    where `interval = stepsize * voxel_size_ratio` [S1, unofficial].

6.  **First Color Threshold:** If `fast_color_thres > 0`, filter points where `alpha > fast_color_thres` [S1, unofficial].

7.  **Weight Computation:** Compute accumulated transmittance and weights using `Alphas2Weights.apply(alpha, ray_id, N)`. This function performs a segmented sum over rays to compute weights and the final transmittance `alphainv_last` [S1, unofficial].

8.  **Second Color Threshold:** If `fast_color_thres > 0`, filter points where `weights > fast_color_thres` [S1, unofficial].

9.  **Color Query:** Query the voxel grid for color features `k0` at the remaining points [S1, unofficial].

10. **RGB Computation:**
    *   If `rgbnet is None`: `rgb = sigmoid(k0)` [S1, unofficial].
    *   If `rgbnet is not None`:
        *   If `rgbnet_direct`: `rgb_feat = k0`.
        *   Else: `k0_view = k0[:, 3:]`, `k0_diffuse = k0[:, :3]`.
        *   Compute view direction embeddings: `viewdirs_emb = (viewdirs.unsqueeze(-1) * viewfreq).flatten(-2)`, then concatenate with sin/cos of viewdirs.
        *   `rgb_feat = cat([k0_view, viewdirs_emb], -1)`.
        *   `rgb_logit = rgbnet(rgb_feat)`.
        *   If `rgbnet_direct`: `rgb = sigmoid(rgb_logit)`.
        *   Else: `rgb = sigmoid(rgb_logit + k0_diffuse)` [S1, unofficial].

11. **Accumulation:** Accumulate RGB using segmented sum:
    $$ \text{rgb\_marched} = \sum_{i \in \text{ray}} \text{weights}_i \cdot \text{rgb}_i $$
    Add background color: `rgb_marched += alphainv_last * bg` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sample_pts_on_rays(rays_o, rays_d, xyz_min, xyz_max, near, far, stepdist):
    N = rays_o.shape[0]
    max_steps = int((far - near) / stepdist) + 1
    ray_pts, ray_id, step_id, mask_outbbox = [], [], [], []
    t_min = torch.full((N,), far)
    t_max = torch.full((N,), near)

    for i in range(N):
        o = rays_o[i]
        d = rays_d[i]
        t1 = (xyz_min - o) / d
        t2 = (xyz_max - o) / d
        t_near = torch.max(torch.min(t1, t2))
        t_far = torch.min(torch.max(t1, t2))
        t_near = torch.max(t_near, torch.tensor(near))
        t_far = torch.min(t_far, torch.tensor(far))

        if t_near >= t_far:
            mask_outbbox.append(torch.ones(max_steps, dtype=bool))
            continue

        t_min[i] = t_near
        t_max[i] = t_far
        num_steps = int((t_far - t_near) / stepdist) + 1
        ts = torch.arange(num_steps) * stepdist + t_near
        pts = o.unsqueeze(0) + ts.unsqueeze(1) * d.unsqueeze(0)

        ray_pts.append(pts)
        ray_id.append(torch.full((num_steps,), i))
        step_id.append(torch.arange(num_steps))
        mask_outbbox.append(torch.zeros(num_steps, dtype=bool))

    ray_pts = torch.cat(ray_pts, dim=0) if ray_pts else torch.empty(0, 3)
    ray_id = torch.cat(ray_id, dim=0) if ray_id else torch.empty(0, dtype=torch.long)
    step_id = torch.cat(step_id, dim=0) if step_id else torch.empty(0, dtype=torch.long)
    mask_outbbox = torch.cat(mask_outbbox, dim=0) if mask_outbbox else torch.empty(0, dtype=bool)
    N_steps = max_steps
    return ray_pts, mask_outbbox, ray_id, step_id, N_steps, t_min, t_max

def alphas2weights(alpha, ray_id, N):
    M = alpha.shape[0]
    weights = torch.zeros(M, dtype=alpha.dtype)
    alphainv_last = torch.zeros(N, dtype=alpha.dtype)
    ray_indices = [[] for _ in range(N)]
    for i, rid in enumerate(ray_id):
        if rid < N:
            ray_indices[rid].append(i)

    for r in range(N):
        if not ray_indices[r]:
            continue
        T_curr = 1.0
        for idx in ray_indices[r]:
            a = alpha[idx].item()
            weights[idx] = T_curr * a
            T_curr *= (1.0 - a)
        alphainv_last[r] = 1.0 - T_curr
    return weights, alphainv_last

def segment_coo(src, index, out, reduce='sum'):
    for i in range(src.shape[0]):
        r = index[i].item()
        if r < out.shape[0]:
            if reduce == 'sum':
                out[r] += src[i]
    return out

def render_directvoxgo(rays_o, rays_d, viewdirs, model, render_kwargs):
    stepdist = render_kwargs['stepsize'] * model.voxel_size
    ray_pts, mask_outbbox, ray_id, step_id, N_steps, t_min, t_max = sample_pts_on_rays(
        rays_o, rays_d, model.xyz_min, model.xyz_max,
        render_kwargs['near'], render_kwargs['far'], stepdist
    )

    mask_inbbox = ~mask_outbbox
    ray_pts = ray_pts[mask_inbbox]
    ray_id = ray_id[mask_inbbox]
    step_id = step_id[mask_inbbox]

    if model.mask_cache is not None:
        mask = model.mask_cache(ray_pts)
        ray_pts = ray_pts[mask]
        ray_id = ray_id[mask]
        step_id = step_id[mask]

    density = model.density(ray_pts)
    interval = render_kwargs['stepsize'] * model.voxel_size_ratio
    alpha = 1.0 - (1.0 + torch.exp(density + model.shift)) ** (-interval)

    if model.fast_color_thres > 0:
        mask = (alpha > model.fast_color_thres)
        ray_pts = ray_pts[mask]
        ray_id = ray_id[mask]
        step_id = step_id[mask]
        density = density[mask]
        alpha = alpha[mask]

    weights, alphainv_last = alphas2weights(alpha, ray_id, len(rays_o))

    if model.fast_color_thres > 0:
        mask = (weights > model.fast_color_thres)
        weights = weights[mask]
        alpha = alpha[mask]
        ray_pts = ray_pts[mask]
        ray_id = ray_id[mask]
        step_id = step_id[mask]

    k0 = model.k0(ray_pts)

    if model.rgbnet is None:
        rgb = torch.sigmoid(k0)
    else:
        if model.rgbnet_direct:
            k0_view = k0
        else:
            k0_view = k0[:, 3:]
            k0_diffuse = k0[:, :3]

        viewdirs_emb = (viewdirs.unsqueeze(-1) * model.viewfreq).flatten(-2)
        viewdirs_emb = torch.cat([viewdirs, viewdirs_emb.sin(), viewdirs_emb.cos()], -1)
        viewdirs_emb = viewdirs_emb.flatten(0, -2)[ray_id]

        rgb_feat = torch.cat([k0_view, viewdirs_emb], -1)
        rgb_logit = model.rgbnet(rgb_feat)

        if model.rgbnet_direct:
            rgb = torch.sigmoid(rgb_logit)
        else:
            rgb = torch.sigmoid(rgb_logit + k0_diffuse)

    rgb_marched = segment_coo(
        src=(weights.unsqueeze(-1) * rgb),
        index=ray_id,
        out=torch.zeros((len(rays_o), 3), dtype=rgb.dtype),
        reduce='sum'
    )

    rgb_marched += (alphainv_last.unsqueeze(-1) * render_kwargs['bg'])
    return rgb_marched, alphainv_last
```

**Edge Cases and Pitfalls:**
*   **Far Distance:** The `far` distance can be too small, causing rays to stop prematurely when hitting the scene bounding box. The code sets `far = 1e9` to prevent this [S1, unofficial].
*   **View Directions:** View directions are passed as input and must be normalized for view-dependent color computation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of sampled points and the efficiency of the mask cache. The mask cache significantly reduces the number of points processed in free space [S1, unofficial].
*   **Memory Complexity:** Proportional to the voxel grid size and the number of rays. The mask cache adds a small memory overhead [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the resolution of the voxel grid and the number of rays [S1, unofficial].
*   **Strengths:** Efficient sampling in sparse scenes due to free space culling. Direct lookup of features from the voxel grid [S1, unofficial].
*   **Limitations:** Requires a predefined bounding box. Performance may degrade in very dense scenes if the mask cache is less effective [S1, unofficial].
*   **When to Use:** Suitable for scenes with significant free space and where efficient sampling is critical [S1, unofficial].
*   **When Not to Use:** Not suitable for scenes without a clear bounding box or where high-frequency details require very fine voxel grids [S1, unofficial].

## Background
The method is implemented in the DirectVoxGO repository [S1, unofficial]. It builds on the idea of using voxel grids for neural rendering, aiming to improve efficiency over standard NeRF by leveraging spatial acceleration structures [S1, unofficial].

## Sources
[S1] DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
