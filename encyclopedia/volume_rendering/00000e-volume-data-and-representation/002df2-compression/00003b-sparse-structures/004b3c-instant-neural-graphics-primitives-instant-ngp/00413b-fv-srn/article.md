# fV-SRN
## Overview
fV-SRN is an interactive rendering method for compressed Implicit Neural Representations (INRs) that uses a volumetric latent grid and a lightweight MLP to achieve real-time frame rates by fusing network evaluation into shared memory [S2, unofficial]. It differs from its sibling [Context-based NeRF Compression (CNC)](16592-context-based-nerf-compression-cnc.md) and the baseline Neurcomp by eliminating global memory access during inference, achieving a 150–200× rendering speedup [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volumetric dataset (up to $1024^3$ resolution) and camera parameters [S2, unofficial].
*   **Output:** Rendered images at interactive frame rates (5–10 FPS at $512^2$ resolution) [S2, unofficial].

**Data Structures:**
1.  **Volumetric Latent Grid:** A 3D grid storing learned features.
    *   **Resolution:** Typically $32^3$ [S2, unofficial].
    *   **Features:** 16 features per vertex [S2, unofficial].
    *   **Layout:** Uniformly spaced in the domain of the input volume [S2, unofficial].
2.  **Lightweight MLP:** A small neural network that decodes the latent features.
    *   **Architecture:** Four layers, 32 channels [S2, unofficial].
    *   **Function:** Maps latent features to density, gradient, and curvature [S2, unofficial].

**Algorithm Steps:**
1.  **Training:** Train the volumetric latent grid and the lightweight MLP to represent the input volume. The training loop uses 2563 samples per epoch for 200 epochs, equivalent to 51.2k steps [S1]. The implementation uses PyTorch [S1].
2.  **Inference (Rendering):**
    1.  For each ray, sample points in the volume [G].
    2.  Retrieve the latent features from the volumetric latent grid for the sampled points via trilinear interpolation [G].
    3.  Execute a custom GPU TensorCore kernel that fuses the entire network evaluation (grid lookup + MLP forward pass) into 48 kB of on-chip shared memory [S2, unofficial].
    4.  This fusion eliminates global memory access during inference [S2, unofficial].
    5.  Predict density, gradient, and curvature in a single forward pass [S2, unofficial].
    6.  Use the predicted gradient and curvature to avoid finite-difference gradient computation, yielding an additional 12× speedup for shaded rendering [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple

# Self-contained helper definitions to replace undefined library calls
Point3 = Tuple[float, float, float]
Vec3 = Tuple[float, float, float]

def vec3(x: float, y: float, z: float) -> Vec3:
    return (x, y, z)

def ray_origin(cam: dict) -> Point3:
    return cam["origin"]

def ray_direction(cam: dict, pixel_x: int, pixel_y: int, res: int) -> Vec3:
    # Standard pinhole camera projection [G]
    fov = cam["fov"]
    aspect = res / res
    ndc_x = (2.0 * pixel_x / res - 1.0) * aspect
    ndc_y = (1.0 - 2.0 * pixel_y / res)
    scale = math.tan(fov * 0.5)
    return vec3(ndc_x * scale, ndc_y * scale, -1.0)

def sample_ray(ray_orig: Point3, ray_dir: Vec3, bounds: Tuple[Point3, Point3], steps: int) -> List[Point3]:
    # March along ray within volume bounds [G]
    t_min = max(0.0, (bounds[0][0] - ray_orig[0]) / ray_dir[0] if ray_dir[0] != 0 else float('inf'))
    t_max = min(float('inf'), (bounds[1][0] - ray_orig[0]) / ray_dir[0] if ray_dir[0] != 0 else float('inf'))
    t_min = max(t_min, (bounds[0][1] - ray_orig[1]) / ray_dir[1] if ray_dir[1] != 0 else float('inf'))
    t_max = min(t_max, (bounds[1][1] - ray_orig[1]) / ray_dir[1] if ray_dir[1] != 0 else float('inf'))
    t_min = max(t_min, (bounds[0][2] - ray_orig[2]) / ray_dir[2] if ray_dir[2] != 0 else float('inf'))
    t_max = min(t_max, (bounds[1][2] - ray_orig[2]) / ray_dir[2] if ray_dir[2] != 0 else float('inf'))
    
    if t_min > t_max:
        return []
    
    step_size = (t_max - t_min) / steps
    points = []
    for i in range(steps):
        t = t_min + i * step_size
        points.append(vec3(
            ray_orig[0] + t * ray_dir[0],
            ray_orig[1] + t * ray_dir[1],
            ray_orig[2] + t * ray_dir[2]
        ))
    return points

def trilinear_interpolate(grid: List, p: Point3, grid_res: int = 32) -> List[float]:
    # Normalize to [0, grid_res-1], clamp, and interpolate [G]
    def clamp(v, min_v, max_v): return max(min_v, min(max_v, v))
    def lerp(a, b, t): return a + t * (b - a)
    
    x, y, z = p
    ix, iy, iz = int(clamp(x, 0, grid_res-1)), int(clamp(y, 0, grid_res-1)), int(clamp(z, 0, grid_res-1))
    fx, fy, fz = x - ix, y - iy, z - iz
    
    # Fetch 8 corners (simplified for pseudocode)
    def get_val(ix, iy, iz): return grid[ix][iy][iz] # Shape: (32,32,32,16)
    
    c000 = get_val(ix, iy, iz)
    c100 = get_val(ix+1, iy, iz)
    c010 = get_val(ix, iy+1, iz)
    c110 = get_val(ix+1, iy+1, iz)
    c001 = get_val(ix, iy, iz+1)
    c101 = get_val(ix+1, iy, iz+1)
    c011 = get_val(ix, iy+1, iz+1)
    c111 = get_val(ix+1, iy+1, iz+1)
    
    # Trilinear interpolation across 16 channels
    def interp3d(v000, v100, v010, v110, v001, v101, v011, v111, fx, fy, fz):
        c00 = [lerp(v000[i], v100[i], fx) for i in range(16)]
        c10 = [lerp(v010[i], v110[i], fx) for i in range(16)]
        c01 = [lerp(v001[i], v101[i], fx) for i in range(16)]
        c11 = [lerp(v011[i], v111[i], fx) for i in range(16)]
        c0 = [lerp(c00[i], c10[i], fy) for i in range(16)]
        c1 = [lerp(c01[i], c11[i], fy) for i in range(16)]
        return [lerp(c0[i], c1[i], fz) for i in range(16)]
        
    return interp3d(c000, c100, c010, c110, c001, c101, c011, c111, fx, fy, fz)

def mlp_forward(features: List[float], weights: List) -> Tuple[float, Vec3, float]:
    # 4-layer MLP, 32 channels [G]
    # Simplified forward pass: features(16) -> hidden(32) -> hidden(32) -> hidden(32) -> output(1+3+1=5)
    # Returns density, gradient(3), curvature [G]
    out = [0.0] * 5
    # Standard MLP computation omitted for brevity; represents the fused kernel step [G]
    return out[0], (out[1], out[2], out[3]), out[4]

def composite(samples: List[Point3], grid: List, weights: List, ray_dir: Vec3) -> Vec3:
    # Standard volumetric compositing [G]
    color = vec3(0.0, 0.0, 0.0)
    alpha_acc = 0.0
    for p in samples:
        feats = trilinear_interpolate(grid, p)
        density, grad, curv = mlp_forward(feats, weights)
        # Shading uses predicted gradient/curvature to avoid finite differences [S2, unofficial]
        # Standard Beer-Lambert compositing [G]
        alpha = 1.0 - math.exp(-density)
        color = color * (1.0 - alpha) + vec3(1.0, 1.0, 1.0) * alpha * 0.5 # Simplified lighting
        alpha_acc += alpha * (1.0 - alpha_acc)
        if alpha_acc > 0.99: break
    return color

def render_fvsrn(volume_data: List, camera_params: dict, latent_grid: List, mlp_weights: List) -> List:
    """
    Renders a volume using fV-SRN.
    Args:
        volume_data: Input 3D volume (training only).
        camera_params: Camera position, orientation, fov, resolution.
        latent_grid: 3D tensor of shape (32, 32, 32, 16) containing learned features.
        mlp_weights: Weights for the 4-layer, 32-channel MLP.
    Returns:
        Rendered image as list of pixel colors.
    """
    res = camera_params["resolution"]
    bounds = ((0, 0, 0), (res, res, res))
    image = []
    
    for y in range(res):
        for x in range(res):
            orig = ray_origin(camera_params)
            dir_vec = ray_direction(camera_params, x, y, res)
            dir_vec = (dir_vec[0]/math.sqrt(sum(c*c for c in dir_vec)), 
                       dir_vec[1]/math.sqrt(sum(c*c for c in dir_vec)), 
                       dir_vec[2]/math.sqrt(sum(c*c for c in dir_vec)))
            
            samples = sample_ray(orig, dir_vec, bounds, steps=256)
            color = composite(samples, latent_grid, mlp_weights, dir_vec)
            image.append(color)
            
    return image
```

**Parameters:**
*   **Latent Grid Resolution:** $32^3$ [S2, unofficial].
*   **MLP Layers:** 4 [S2, unofficial].
*   **MLP Channels:** 32 [S2, unofficial].
*   **Shared Memory Size:** 48 kB [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Training Memory:** Training requires the full volume in GPU memory, limiting use to offline workflows or requiring sufficient VRAM [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is accelerated by 150–200× over Neurcomp [S2, unofficial]. Shaded rendering is accelerated by an additional 12× [S2, unofficial].
*   **Memory Complexity:** Training requires the full volume in GPU memory [S2, unofficial]. The latent grid is small ($32^3 \times 16$ features), and the MLP weights are minimal (4 layers, 32 channels) [S2, unofficial].
*   **Measured Performance:** 5–10 FPS at $512^2$ resolution on volumes up to $1024^3$ [S2, unofficial].
*   **Strengths:** Real-time interactive rendering; high speedup over previous methods; avoids finite-difference gradient computation [S2, unofficial].
*   **Limitations:** Training requires significant GPU memory; PyTorch implementation may exaggerate training time differences compared to custom C++ implementations [S1].
*   **When to Use:** When interactive rendering of compressed INRs is required and sufficient GPU memory is available for training [S2, unofficial].
*   **When Not to Use:** When VRAM is limited during training or when offline rendering is acceptable [S2, unofficial].

## Background
Compared against the proposed method in [S1] on a Linux machine with an NVIDIA RTX 3090Ti using the same number of trainable parameters [S1].

## Sources
[S1] Interactive Volume Visualization via Multi-Resolution Hash Encoding ..., https://arxiv.org/abs/2207.11620, fetched 2026-10-26
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
