# Background Point Sampling
## Overview
Background Point Sampling generates sample points along rays for background regions, defined as the space outside the scene's bounding box [S1, unofficial]. It employs a specific non-uniform sampling distribution controlled by a `bg_preserve` parameter to balance between uniform angular sampling and uniform radial sampling [S1, unofficial]. This method is distinct from sibling methods like Random Point Sampling in Bounds, which typically samples uniformly within a defined volume, by specifically targeting the infinite or large-distance background region with a distribution that adapts based on the ray's geometry relative to the bounding box [S1, unofficial].

## How it works
**Inputs:**
- `rays_o`: Origin coordinates for all rays, shape `(N_rays, 3)`.
- `rays_d`: Direction vectors for all rays, shape `(N_rays, 3)`.
- `t_max`: Array of maximum t-values (intersection distances) for each ray, shape `(N_rays,)`.
- `N_samples`: Integer, number of background samples to generate per ray.
- `bg_preserve`: Float parameter controlling the sampling distribution blend, typically in $[0, 1]$ [S1, unofficial].

**Outputs:**
- `bg_pts`: Sample point coordinates in the original world space, shape `(N_rays * N_samples, 3)`.
- `bg_t`: Sample distances from the ray origin, shape `(N_rays * N_samples,)`.

**Data Structures:**
The implementation flattens the ray-sample grid into a 1D array for efficient GPU processing. The index `idx` maps to a specific ray and sample step:
- `i_ray = idx // N_samples`
- `i_step = idx % N_samples`

**Steps:**
1. For each sample index `idx` from $0$ to $N_{rays} \times N_{samples} - 1$:
2. Calculate the ray index `i_ray` and step index `i_step` as defined above [S1, unofficial].
3. Retrieve the maximum t-value for the current ray: $t_{inner} = t_{max}[i_{ray}]$ [S1, unofficial].
4. Calculate the original outer t-coordinate $t_{outer}^{orig}$ using the preservation parameter:
   $$ t_{outer}^{orig} = t_{inner} - 1.0 + \frac{1.0}{1.0 - \frac{i_{step}}{N_{samples}}} $$
   [S1, unofficial]
5. Calculate the original 3D point coordinates in world space using the ray origin and direction scaled by $t_{outer}^{orig}$:
   $$ \mathbf{p}_{orig} = \mathbf{o}_{ray} + \mathbf{d}_{ray} \times t_{outer}^{orig} $$
   where $\mathbf{o}_{ray} = \text{rays\_o}[i_{ray}]$ and $\mathbf{d}_{ray} = \text{rays\_d}[i_{ray}]$. Let components be $x, y, z$.
   [S1, unofficial]
6. Calculate the Euclidean distance from the origin:
   $$ t_{outer} = \sqrt{x^2 + y^2 + z^2} $$
   [S1, unofficial]
7. Calculate the maximum absolute coordinate component:
   $$ m = \max(|x|, |y|, |z|) $$
   [S1, unofficial]
8. Calculate the radius factor:
   $$ R_{outer} = \frac{t_{outer}}{m} $$
   [S1, unofficial]
9. Calculate the scaling factor $o2i_p$ using `bg_preserve`:
   $$ o2i_p = \left(\frac{R_{outer}^2}{t_{outer}^2}\right) (1 - \text{bg\_preserve}) + \left(\frac{R_{outer}}{t_{outer}}\right) \text{bg\_preserve} $$
   [S1, unofficial]
10. Scale the original point coordinates to get the final background point:
    $$ \mathbf{p}_{final} = \mathbf{p}_{orig} \times o2i_p $$
    [S1, unofficial]
11. Store $\mathbf{p}_{final}$ and $t_{outer}$ for the current `idx` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def background_point_sampling(rays_o, rays_d, t_max, N_samples, bg_preserve):
    N_rays = len(rays_o)
    total_samples = N_rays * N_samples
    bg_pts = [None] * total_samples
    bg_t = [0.0] * total_samples

    for idx in range(total_samples):
        i_ray = idx // N_samples
        i_step = idx % N_samples

        t_inner = t_max[i_ray]
        denom = 1.0 - (i_step / N_samples)
        if denom == 0.0:
            t_outer_orig = float('inf')
        else:
            t_outer_orig = t_inner - 1.0 + 1.0 / denom

        ray_o = rays_o[i_ray]
        ray_d = rays_d[i_ray]
        ori_ray_pts_x = ray_o[0] + ray_d[0] * t_outer_orig
        ori_ray_pts_y = ray_o[1] + ray_d[1] * t_outer_orig
        ori_ray_pts_z = ray_o[2] + ray_d[2] * t_outer_orig

        t_outer = (ori_ray_pts_x**2 + ori_ray_pts_y**2 + ori_ray_pts_z**2)**0.5
        ori_ray_pts_m = max(abs(ori_ray_pts_x), abs(ori_ray_pts_y), abs(ori_ray_pts_z))

        if ori_ray_pts_m == 0.0:
            R_outer = 0.0
        else:
            R_outer = t_outer / ori_ray_pts_m

        if t_outer == 0.0:
            o2i_p = 0.0
        else:
            o2i_p = (R_outer**2 / t_outer**2) * (1.0 - bg_preserve) + (R_outer / t_outer) * bg_preserve

        final_x = ori_ray_pts_x * o2i_p
        final_y = ori_ray_pts_y * o2i_p
        final_z = ori_ray_pts_z * o2i_p

        bg_pts[idx] = (final_x, final_y, final_z)
        bg_t[idx] = t_outer

    return bg_pts, bg_t
```

**Parameters:**
- `bg_preserve`: Float, typically between 0 and 1. Controls the blend between sampling distributions [S1, unofficial].
- `N_samples`: Integer, number of samples per ray [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `ori_ray_pts_m` is 0 (point at origin), `R_outer` is undefined. A guard returning 0.0 is required [G].
- **Denominator Zero:** In the calculation of `t_outer_orig`, if `i_step == N_samples`, the denominator `1.0 - (i_step / N_samples)` becomes zero. Since `i_step` ranges from $0$ to $N_{samples}-1$, this case is avoided. If `N_samples` is 0, the loop does not execute [G].
- **Precision:** Floating point precision issues may arise when `t_outer` is very large or very small. The scaling factor `o2i_p` can become unstable if `t_outer` is close to zero [G].

## Operating characteristics
**Time Complexity:** $O(N_{rays} \times N_{samples})$. Each sample requires a constant number of arithmetic operations [G].
**Memory Complexity:** $O(N_{rays} \times N_{samples})$ to store the output points and distances [G].
**Strengths:**
- Provides a structured way to sample the background, which is crucial for NeRF-based methods that need to account for background color and depth [G].
- The `bg_preserve` parameter allows tuning the sampling density [S1, unofficial].
**Limitations:**
- The non-uniform distribution may require more samples to achieve the same variance reduction as uniform sampling in certain scenarios [G].
- Computationally more expensive than simple uniform sampling due to the per-sample geometric calculations [G].
**When to Use:**
- When rendering scenes with significant background content that needs to be modeled explicitly [G].
**When Not to Use:**
- If the background is simple (e.g., constant color) and does not require detailed sampling [G].
- If computational resources are extremely limited and a simpler background approximation suffices [G].

## Background
Background Point Sampling is used in volume rendering to handle the background region by generating samples outside the bounding box of the scene [G].

## Sources
[S1] sunset1995/DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
