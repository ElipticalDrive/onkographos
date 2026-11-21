# Differentiable Ray Marching (Taichi)
Also called: Taichi Differentiable Volume Rendering
## Overview
A differentiable volume rendering technique implemented in Taichi using fixed-step ray marching and trilinear interpolation on a 3D density grid [S1, unofficial]. It differs from sibling methods by relying on a fixed-step traversal with explicit alpha accumulation rather than expected terms or thresholding, and is implemented using Taichi's autodiff engine rather than PyTorch [S1, unofficial].
## How it works
**Inputs:** Camera position (`cam_pos`), camera rotation matrix (`cam_rot`), pixel coordinates (`pixel_coords`), and a 4D density grid (`density`) containing RGBA values.
**Outputs:** A rendered image (`target_image`) with RGBA channels.

**Data Structures:**
*   `density`: Shape `[8, 128, 128, 128, 4]` (batch, x, y, z, rgba) [S1, unofficial].
*   `target_image`: Shape `[8, 4, 128, 128]` (batch, rgba, height, width) [S1, unofficial].
*   `cam_pos`: Shape `[8, 3]` [S1, unofficial].
*   `cam_rot`: Shape `[8, 3, 3]` [S1, unofficial].
*   `pixel_coords`: Shape `[8, 128, 128, 2]` (batch, height, width, xy) [S1, unofficial].

**Parameters:**
*   `dx = 0.015625` (fixed step size) [S1, unofficial].
*   `density_res = 128` (grid resolution) [S1, unofficial].
*   `focal_x = 875.`, `focal_y = 875.` [S1, unofficial].
*   `princpt_x = 400.`, `princpt_y = 400.` [S1, unofficial].

**Steps:**
1.  For each pixel $(x, y)$ and batch index $n$:
2.  Compute ray direction components: $a = (y - \text{princpt}_y) / \text{focal}_y$, $b = (x - \text{princpt}_x) / \text{focal}_x$ [S1, unofficial].
3.  Compute direction vector: $\mathbf{dir} = \text{cam\_rot}[n] \cdot [a, b, 1]^T$ and normalize it [S1, unofficial].
4.  Iterate $k$ from 0 to 299:
    a. Compute sample point: $\mathbf{point} = \text{cam\_pos}[n] + (k + 1) \cdot dx \cdot \mathbf{dir}$ [S1, unofficial].
    b. Check if $\mathbf{point}$ is inside the box $[-1, 1]^3$. If so, set `flag = 1`, else `flag = 0` [S1, unofficial].
    c. Map point to grid coordinates: $t_x = (\mathbf{point}[0] + 1.) \cdot 127 / 2.$, similarly for $t_y, t_z$ [S1, unofficial].
    d. Perform trilinear interpolation on `density` at $(t_x, t_y, t_z)$ to get $r, g, b, a$ [S1, unofficial].
    e. Compute alpha contribution: $\Delta \alpha = (\min(\text{current\_alpha} + a \cdot dx, 1.) - \text{current\_alpha}) \cdot \text{flag}$ [S1, unofficial].
    f. Accumulate color: $\text{target\_image}[n, c, x, y] += r_c \cdot \Delta \alpha$ for $c \in \{r, g, b\}$ [S1, unofficial].
    g. Update alpha: $\text{target\_image}[n, 3, x, y] += \Delta \alpha$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import taichi as ti

ti.init(default_fp=ti.f32, arch=ti.cuda)

density_res = 128
dx = 0.015625
focal_x = 875.
focal_y = 875.
princpt_x = 400.
princpt_y = 400.

# Fields
density = ti.field(dtype=ti.f32, shape=(8, 128, 128, 128, 4))
target_image = ti.field(dtype=ti.f32, shape=(8, 4, 128, 128))
cam_pos = ti.field(dtype=ti.f32, shape=(8, 3))
cam_rot = ti.field(dtype=ti.f32, shape=(8, 3, 3))
pixel_coords = ti.field(dtype=ti.i32, shape=(8, 128, 128, 2))

@ti.func
def in_box(x, y, z):
    return x >= -1.0 and x <= 1.0 and y >= -1.0 and y <= 1.0 and z >= -1.0 and z <= 1.0

@ti.func
def clip(x):
    return int(ti.max(0, ti.min(int(x), density_res - 1)))

@ti.func
def trilinear_interpolation(n, z, y, x, channel):
    # Extract integer and fractional parts
    ix = int(x)
    iy = int(y)
    iz = int(z)
    fx = x - ix
    fy = y - iy
    fz = z - iz
    
    # Clip indices
    ix = clip(ix)
    iy = clip(iy)
    iz = clip(iz)
    
    # Get 8 neighbors
    c000 = density[n, ix, iy, iz, channel]
    c100 = density[n, ix+1, iy, iz, channel]
    c010 = density[n, ix, iy+1, iz, channel]
    c110 = density[n, ix+1, iy+1, iz, channel]
    c001 = density[n, ix, iy, iz+1, channel]
    c101 = density[n, ix+1, iy, iz+1, channel]
    c011 = density[n, ix, iy+1, iz+1, channel]
    c111 = density[n, ix+1, iy+1, iz+1, channel]
    
    # Interpolate
    c00 = c000 * (1 - fx) + c100 * fx
    c10 = c010 * (1 - fx) + c110 * fx
    c01 = c001 * (1 - fx) + c101 * fx
    c11 = c011 * (1 - fx) + c111 * fx
    
    c0 = c00 * (1 - fy) + c10 * fy
    c1 = c01 * (1 - fy) + c11 * fy
    
    c = c0 * (1 - fz) + c1 * fz
    return c

@ti.kernel
def ray_march():
    for n, x, y in target_image:
        # Initialize alpha to 0
        target_image[n, 3, x, y] = 0.0
        
        for pixel in range(128 * 128):
            px = pixel // 128
            py = pixel - px * 128
            
            # Get pixel coords
            px_val = pixel_coords[n, px, py, 0]
            py_val = pixel_coords[n, px, py, 1]
            
            # Compute ray direction
            a = (py_val - princpt_y) / focal_y
            b = (px_val - princpt_x) / focal_x
            
            dir_x = cam_rot[n, 0, 0] * a + cam_rot[n, 1, 0] * b + cam_rot[n, 2, 0]
            dir_y = cam_rot[n, 0, 1] * a + cam_rot[n, 1, 1] * b + cam_rot[n, 2, 1]
            dir_z = cam_rot[n, 0, 2] * a + cam_rot[n, 1, 2] * b + cam_rot[n, 2, 2]
            
            length = ti.sqrt(dir_x**2 + dir_y**2 + dir_z**2)
            dir_x /= length
            dir_y /= length
            dir_z /= length
            
            camera_origin_x = cam_pos[n, 0]
            camera_origin_y = cam_pos[n, 1]
            camera_origin_z = cam_pos[n, 2]
            
            current_alpha = target_image[n, 3, x, y]
            
            for k in range(300):
                point_x = camera_origin_x + (k + 1) * dx * dir_x
                point_y = camera_origin_y + (k + 1) * dx * dir_y
                point_z = camera_origin_z + (k + 1) * dx * dir_z
                
                flag = 0
                if in_box(point_x, point_y, point_z):
                    flag = 1
                
                template_x = (point_x + 1.) * 127 / 2.
                template_y = (point_y + 1.) * 127 / 2.
                template_z = (point_z + 1.) * 127 / 2.
                
                r_interp = trilinear_interpolation(n, template_z, template_y, template_x, 0)
                g_interp = trilinear_interpolation(n, template_z, template_y, template_x, 1)
                b_interp = trilinear_interpolation(n, template_z, template_y, template_x, 2)
                a_interp = trilinear_interpolation(n, template_z, template_y, template_x, 3)
                
                contribution_r = r_interp * flag
                contribution_g = g_interp * flag
                contribution_b = b_interp * flag
                contribution_a = a_interp * flag
                
                # Compute alpha contribution
                new_alpha = ti.min(current_alpha + contribution_a * dx, 1.0)
                delta_alpha = new_alpha - current_alpha
                
                # Accumulate color
                target_image[n, 0, x, y] += contribution_r * delta_alpha
                target_image[n, 1, x, y] += contribution_g * delta_alpha
                target_image[n, 2, x, y] += contribution_b * delta_alpha
                
                # Update alpha
                target_image[n, 3, x, y] = new_alpha
                current_alpha = new_alpha
```

**Edge Cases and Pitfalls:**
*   Gradients computed by Taichi's autodiff may be rotated/translated compared to PyTorch due to implementation differences in differentiation rules for ray marching [S1, unofficial].
*   Points outside the $[-1, 1]^3$ box are ignored via `flag` [S1, unofficial].
*   Grid coordinates are clipped to $[0, 127]$ to avoid out-of-bounds access [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot H \cdot W \cdot K)$ where $N$ is batch size, $H, W$ are image dimensions, and $K$ is number of steps (300) [S1, unofficial].
*   **Memory Complexity:** $O(N \cdot R^3 \cdot C)$ for the density grid, where $R$ is resolution (128) and $C$ is channels (4) [S1, unofficial].
*   **Strengths:** Simple implementation, differentiable via Taichi's autodiff [S1, unofficial].
*   **Limitations:** Fixed step size may lead to aliasing or inefficiency; gradient mismatches with PyTorch [S1, unofficial].
*   **When to use:** When using Taichi for differentiable volume rendering and PyTorch compatibility is not critical for gradients [S1, unofficial].
*   **When not to use:** When precise gradient alignment with PyTorch is required [S1, unofficial].

## Background
Originated from a GitHub issue discussing gradient mismatches between Taichi and PyTorch implementations of differentiable ray marching [S1, unofficial]. The code is based on examples from the Taichi repository [S1, unofficial].

## Sources
[S1] Gradients Computed by Taichi Don't Match Gradients Computed by PyTorch, https://github.com/taichi-dev/taichi/issues/2241, fetched 2026-10-27 (unofficial)
