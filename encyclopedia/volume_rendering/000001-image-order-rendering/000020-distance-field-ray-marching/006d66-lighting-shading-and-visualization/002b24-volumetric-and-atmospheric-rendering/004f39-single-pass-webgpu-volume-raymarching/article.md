# Single-pass WebGPU volume raymarching

Also called: raymarchScene, volume raymarcher

## Overview
This method renders volumetric data (such as magnetic field magnitude) in a single GPU pass using WebGPU, performing analytic ray-box intersection, early-alpha termination, and optional central-difference Phong shading [S1, unofficial]. It differs from standard sphere tracing [Distance-field ray marching](21-distance-field-ray-marching.md) by operating on explicit 3D texture volumes rather than implicit Signed Distance Functions (SDFs), allowing for complex, non-uniform density distributions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D texture containing volume data (e.g., density or field magnitude), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), and rendering parameters (steps, density scale, shading flags) [S1, unofficial].
*   **Output:** A 2D color buffer (RGBA) representing the rendered volume [S1, unofficial].

**Data Structures:**
*   **Volume Texture:** A 3D texture (e.g., `texture_3d<f32>` in WGSL) [S1, unofficial]. The object space is defined as the unit box $[-0.5, 0.5]^3$ [S1, unofficial]. Texture coordinates are computed as $\mathbf{pos} + 0.5$ to map the box to $[0, 1]^3$ [S1, unofficial].
*   **Transfer Function:** Maps sampled density values to color and opacity, supporting linear, logarithmic, or symlog modes [S1, unofficial].

**Parameters:**
*   `steps`: Number of samples per ray across the clipped segment. Default: 256 [S1, unofficial].
*   `density`: Opacity scale factor for the emission-absorption transfer function [S1, unofficial].
*   `shaded`: Boolean flag to enable Phong shading. Default: `false` [S1, unofficial].
*   `EARLY_ALPHA`: Threshold for early ray termination. Default: 0.98 [S1, unofficial].
*   `SHADE_T_FLOOR`: Minimum accumulated opacity threshold to enable gradient computation for shading. Default: $1/255$ [S1, unofficial].

**Algorithm Steps:**

1.  **Ray-Box Intersection:**
    Compute the entry ($t_0$) and exit ($t_1$) distances along the ray for the unit box $[-0.5, 0.5]^3$ [S1, unofficial].
    *   Calculate inverse direction components: $inv\_dir_i = 1.0 / d_i$ [S1, unofficial].
    *   **Pitfall Handling:** If any component of $\mathbf{d}$ is near zero (e.g., $|d_i| < 1.0e-8$), substitute $inv\_dir_i$ with a large finite slope ($1.0e30$) to avoid $0 \times \infty = \text{NaN}$ results from axis-parallel rays [S1, unofficial].
    *   Compute axis-aligned intersections:
        $$ t_{axis\_min} = (\mathbf{box\_min} - \mathbf{o}) \cdot inv\_dir $$
        $$ t_{axis\_max} = (\mathbf{box\_max} - \mathbf{o}) \cdot inv\_dir $$
    *   Determine entry and exit:
        $$ t_0 = \max(t_{axis\_min}) $$
        $$ t_1 = \min(t_{axis\_max}) $$
    *   If $t_0 > t_1$, the ray misses the box [S1, unofficial].

2.  **Ray Marching Loop:**
    March along the ray from $t_0$ to $t_1$ in $N$ steps (where $N = \text{steps}$), with step size $\Delta t = (t_1 - t_0) / N$ [S1, unofficial].
    For each step $i$:
    *   **Position:** $\mathbf{p} = \mathbf{o} + (t_0 + i \cdot \Delta t) \mathbf{d}$ [S1, unofficial].
    *   **Sampling:** Sample the volume texture at normalized coordinates $\mathbf{p} + 0.5$ to get density $D$ [S1, unofficial].
    *   **Transfer Function:** Compute opacity $\alpha$ and color $C$ from $D$ using the transfer function scaled by `density` [S1, unofficial].
    *   **Accumulation:** Use front-to-back blending:
        $$ C_{out} = C_{in} + C \cdot \alpha \cdot (1 - \alpha_{in}) $$
        $$ \alpha_{in} = \alpha_{in} + \alpha \cdot (1 - \alpha_{in}) $$
    *   **Early Termination:** If accumulated opacity exceeds `EARLY_ALPHA` (0.98), break the loop [S1, unofficial].

3.  **Shading (Optional):**
    If `shaded` is true and current sample opacity $t \ge \text{SHADE\_T\_FLOOR}$, compute the gradient $\nabla D$ at the current sample position using central differences (6 gradient taps per step) to generate a Phong lighting normal [S1, unofficial].
    *   Normalize $\nabla D$ to get the surface normal $\mathbf{n}$ [G].
    *   Apply Phong lighting model using $\mathbf{n}$, view direction, and light direction [G].
    *   If accumulated opacity $< \text{SHADE\_T\_FLOOR}$, skip gradient computation to stay within the 8 ms budget [S1, unofficial].

**Pseudocode (Python-like):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
EARLY_ALPHA = 0.98
SHADE_T_FLOOR = 1.0 / 255.0
MAX_INV_DIR = 1.0e30
EPSILON = 1.0e-8
BOX_MIN = [-0.5, -0.5, -0.5]
BOX_MAX = [0.5, 0.5, 0.5]

def sample_volume(texture, tex_coords):
    # Standard 3D texture lookup with clamping to [0, 1]
    return texture.sample(tex_coords)

def transfer_function(density_val, density_scale):
    # Emission-absorption transfer function scaled by density
    alpha = 1.0 - exp(-density_scale * density_val)
    color = density_val # Simplified linear mapping; actual implementation uses a lookup or function
    return color, alpha

def compute_gradient(texture, pos):
    # Central differences: 6 taps (±1 along each axis)
    eps = 1.0 / texture.resolution
    dx = sample_volume(texture, pos + [eps, 0, 0]) - sample_volume(texture, pos - [eps, 0, 0])
    dy = sample_volume(texture, pos + [0, eps, 0]) - sample_volume(texture, pos - [0, eps, 0])
    dz = sample_volume(texture, pos + [0, 0, eps]) - sample_volume(texture, pos - [0, 0, eps])
    grad = [dx, dy, dz]
    length = sqrt(dx*dx + dy*dy + dz*dz)
    if length > 1e-8:
        return [g/length for g in grad]
    return [0.0, 0.0, 1.0]

def apply_phong(color, normal, view_dir, light_dir):
    # Standard Phong lighting model [G]
    diff = max(0.0, dot(normal, light_dir))
    return color * (0.5 + 0.5 * diff)

def raymarch_volume(ray_origin, ray_dir, volume_texture, steps, density, shaded):
    # Ray-Box Intersection
    inv_dir = [MAX_INV_DIR if abs(d) < EPSILON else 1.0/d for d in ray_dir]
    t_min = [(BOX_MIN[i] - ray_origin[i]) * inv_dir[i] for i in range(3)]
    t_max = [(BOX_MAX[i] - ray_origin[i]) * inv_dir[i] for i in range(3)]
    t0 = max(t_min)
    t1 = min(t_max)

    if t0 > t1:
        return [0.0, 0.0, 0.0, 0.0]

    dt = (t1 - t0) / steps
    acc_color = [0.0, 0.0, 0.0]
    acc_alpha = 0.0

    for i in range(steps):
        t = t0 + i * dt
        pos = ray_origin + ray_dir * t
        tex_pos = [p + 0.5 for p in pos] # Map object space [-0.5, 0.5] to texture space [0, 1]

        density_val = sample_volume(volume_texture, tex_pos)
        color, alpha = transfer_function(density_val, density)

        weight = alpha * (1.0 - acc_alpha)
        acc_color = [acc_color[j] + color[j] * weight for j in range(3)]
        acc_alpha = acc_alpha + weight

        if acc_alpha > EARLY_ALPHA:
            break

        if shaded and acc_alpha >= SHADE_T_FLOOR:
            normal = compute_gradient(volume_texture, tex_pos)
            acc_color = apply_phong(acc_color, normal, ray_dir, [0.0, 0.0, -1.0])

    return [acc_color[0], acc_color[1], acc_color[2], acc_alpha]
```

## Subtopics

### [WebGPU Ray Marching for 4D Volumes](16928-webgpu-ray-marching-for-4d-volumes.md)
Extends this method by adding temporal filtering and explicit 4D data handling for time-varying volumetric fields like plasma simulations.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of steps [S1, unofficial]. With early termination, the average case is often significantly less than $N$ [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per pixel for accumulation; $O(V)$ for the volume texture storage, where $V$ is the texture resolution [S1, unofficial].
*   **Performance:** Designed to run within an 8 ms budget on modern GPUs [S1, unofficial]. Performance is heavily dependent on the `steps` parameter and the density of the volume [S1, unofficial].
*   **Strengths:** Single-pass execution allows for real-time interaction. Analytic ray-box intersection is efficient. Early-alpha termination optimizes rendering of transparent regions [S1, unofficial].
*   **Limitations:** Fixed step size can lead to aliasing in sparse regions or slow convergence in dense regions. Central-difference shading increases texture fetch count by 6x for shaded samples [S1, unofficial].

## Background
This method is implemented in the `webpic` project by Ruslan Saitis, which provides a WebGPU-based visualization tool for scientific data [S1, unofficial]. It is designed to run in the browser, leveraging WebGPU for high-performance parallel computation. The implementation includes features like HMR (Hot Module Replacement) for shader editing and support for various transfer functions [S1, unofficial].

## Sources
[S1] rusaitis/webpic, https://github.com/rusaitis/webpic, fetched 2026-10-27 (unofficial)
