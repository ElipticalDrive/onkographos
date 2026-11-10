# Light Space Rasterization for T_light

Also called: Front Tau Rasterization, Light Space Shadow Rasterization, T_light Rasterization (Light Space)

## Overview
Light Space Rasterization computes $T_{light}$, the front-facing occlusion transmittance for each 3D Gaussian along a directional sun direction [S1, unofficial]. It differs from voxel-based fallbacks by providing full continuous gradients and zero parameter expansion, enabling differentiable relighting without requiring specific training hyperparameters [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Gaussian splats (position, covariance/scale, rotation, opacity $\alpha$, color/SH coefficients) and a sun direction vector $\mathbf{d}_{sun}$ [S1, unofficial].
*   **Output:** Per-Gaussian scalar $T_{light} \in [0, 1]$, representing the accumulated transmittance from the sun direction to the Gaussian center [S1, unofficial].

**Data Structures:**
*   **Gaussian Attributes:** Standard 3DGS attributes [G]. The method relies on the projected position and scale in light space [S1, unofficial].
*   **Accumulator:** A per-pixel accumulation buffer for optical depth $\tau$ [S1, unofficial]. In the CUDA kernel, this is managed via the `record_front_tau` channel [S1, unofficial].

**Algorithm Steps:**
1.  **Light Camera Setup:** Construct a virtual perspective camera to simulate the directional sun light [S1, unofficial]. Place it at a large distance along $\mathbf{d}_{sun}$ with a narrow field of view (FOV) [S1, unofficial]. The distance and FOV are chosen so that parallax error remains below $2\%$, avoiding changes to the EWA Jacobian [S1, unofficial].
2.  **Light-Space Projection:** Project all 3D Gaussians into the light camera's clip space [S1, unofficial]. Transform Gaussian centers $\mathbf{p}_i$ using the light camera's view-projection matrix $M_{light}$ [G]. Compute the projected 2D covariance matrix $\Sigma_{2D}$ in screen space using standard EWA approximation with the light camera's intrinsics and extrinsics [G].
3.  **Depth Sorting:** Sort the Gaussians by their depth $z$ in light space from front (closest to the light source) to back [S1, unofficial]. This ensures correct front-to-back accumulation for transmittance [S1, unofficial].
4.  **Rasterization Pass:** Iterate over screen pixels covered by the sorted Gaussians [S1, unofficial]. For each Gaussian $i$ covering pixel $(u, v)$, update the accumulated optical depth $\tau_{acc}$ [S1, unofficial]. Each Gaussian records the accumulated front $\tau$ weighted by $\alpha \cdot T$ [S1, unofficial]. The method computes an energy-weighted mean over the entire sun-facing footprint rather than a center-point sample [S1, unofficial]. The transmittance $T_{light}$ for a Gaussian is derived from the accumulated optical depth of all Gaussians in front of it along the sun direction [S1, unofficial].
5.  **Fallback and Edge Cases:** If a Gaussian is deeply buried such that the accumulated weight sum $w_{sum} \approx 0$ (early termination), it is explicitly mapped to full shadow ($T_{light} = 0$) to prevent inversion artifacts [S1, unofficial]. Full continuous gradients are enabled by default for $\tau$, weights, normalization, weak density backup transmittance, and projected position/scale/rotation [S1, unofficial]. The sun camera framing, sorting, coverage, and threshold branches are treated as constants [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_tlight_rasterization(gaussians, sun_direction, screen_width, screen_height):
    # 1. Setup Light Camera
    cam_pos = [sun_direction[0] * 1e6, sun_direction[1] * 1e6, sun_direction[2] * 1e6]
    cam_lookat = [-sun_direction[0], -sun_direction[1], -sun_direction[2]]
    cam_up = [0.0, 1.0, 0.0] if abs(sun_direction[1]) < 0.9 else [1.0, 0.0, 0.0]
    cam_right = cross(cam_lookat, cam_up)
    cam_right = normalize(cam_right)
    cam_up = cross(cam_right, cam_lookat)
    
    view_matrix = look_at(cam_pos, cam_lookat, cam_up)
    fov_rad = math.radians(1.0)  # Narrow FOV [S1, unofficial]
    aspect = screen_width / screen_height
    near, far = 0.1, 1e7
    proj_matrix = perspective_projection(fov_rad, aspect, near, far)
    
    # 2. Project Gaussians to Light Space
    projected = []
    for i, g in enumerate(gaussians):
        center_h = [g['center'][0], g['center'][1], g['center'][2], 1.0]
        center_cam = mat_mul(view_matrix, center_h)
        center_clip = mat_mul(proj_matrix, center_cam)
        
        if center_clip[3] <= 0:
            continue  # Behind camera [G]
            
        center_ndc = [center_clip[0]/center_clip[3], center_clip[1]/center_clip[3], center_clip[2]/center_clip[3]]
        depth = center_ndc[2]
        
        cov_2d = project_covariance(g, view_matrix, proj_matrix)  # Standard EWA [G]
        
        projected.append({
            'index': i, 'center_ndc': center_ndc, 'cov_2d': cov_2d,
            'alpha': g['alpha'], 'depth': depth
        })
        
    # 3. Sort by Depth (Front to Back)
    projected.sort(key=lambda x: x['depth'])
    
    # 4. Rasterization Pass
    tau_buffer = [[0.0] * screen_width for _ in range(screen_height)]
    t_light_values = [0.0] * len(gaussians)
    
    for g in projected:
        u, v, z = g['center_ndc']
        px, py = int((u + 1) * screen_width / 2), int((v + 1) * screen_height / 2)
        
        # Sample accumulated tau at this Gaussian's center
        tau_before = tau_buffer[py][px]
        
        # Compute T_light for this Gaussian
        t_light = math.exp(-tau_before)
        
        # Handle deeply buried Gaussians (early termination wsum=0)
        if t_light < 1e-6:
            t_light = 0.0
                
        t_light_values[g['index']] = t_light
        
        # Update tau_buffer with this Gaussian's contribution
        # Contribution = alpha * T (energy-weighted mean over footprint) [S1, unofficial]
        contribution = g['alpha'] * t_light
        
        # Add to covered pixels (simplified bounding box rasterization)
        sigma_x = math.sqrt(g['cov_2d'][0][0]) * 3.0
        sigma_y = math.sqrt(g['cov_2d'][1][1]) * 3.0
        min_px, max_px = max(0, int(px - sigma_x)), min(screen_width, int(px + sigma_x))
        min_py, max_py = max(0, int(py - sigma_y)), min(screen_height, int(py + sigma_y))
        
        for y in range(min_py, max_py):
            for x in range(min_px, max_px):
                dx, dy = x - px, y - py
                pdf = math.exp(-0.5 * (dx**2 / g['cov_2d'][0][0] + dy**2 / g['cov_2d'][1][1]))
                if pdf > 0.01:  # Threshold [G]
                    tau_buffer[y][x] += contribution
                    
    return t_light_values

def dot(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def cross(a, b): return [a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0]]
def normalize(v): l = math.sqrt(dot(v,v)); return [x/l for x in v]
def look_at(eye, target, up):
    z = normalize([target[i]-eye[i] for i in range(3)])
    x = normalize(cross(up, z))
    y = cross(z, x)
    m = [[x[0],y[0],z[0],-dot(x,eye)], [x[1],y[1],z[1],-dot(y,eye)], [x[2],y[2],z[2],-dot(z,eye)], [0,0,0,1]]
    return m
def perspective_projection(fov, aspect, near, far):
    f = 1.0 / math.tan(fov/2)
    return [[f/aspect,0,0,0], [0,f,0,0], [0,0,-(far+near)/(far-near),-2*far*near/(far-near)], [0,0,-1,0]]
def mat_mul(m, v): return [sum(m[i][j]*v[j] for j in range(4)) for i in range(4)]
def project_covariance(g, view, proj):
    # Simplified EWA: scale by view/proj derivatives [G]
    return [[g['scale'][0]**2, 0], [0, g['scale'][1]**2]]
```

**Parameters:**
*   `LARGE_DISTANCE_CONSTANT`: Distance of light camera from origin. Not given in the sources.
*   `NARROW_FOV_RADIANS`: FOV of light camera. Not given in the sources, but must be narrow to minimize parallax error (<2%) [S1, unofficial].
*   `THRESHOLD_MIN_TRANSMITTANCE`: Threshold for mapping deeply buried Gaussians to full shadow. Not given in the sources.
*   `SCALE_FACTOR`: Scaling factor for density contribution. Not given in the sources.

**Pitfalls:**
*   **Parallax Error:** If the light camera is not far enough or FOV is too wide, parallax error exceeds 2%, affecting the EWA Jacobian accuracy [S1, unofficial].
*   **Inversion Artifacts:** Deeply buried Gaussians must be explicitly mapped to full shadow to prevent artifacts [S1, unofficial].
*   **Gradient Discontinuities:** The sun camera framing, sorting, and threshold branches are treated as constants. Gradients do not flow through these steps [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of Gaussians and pixels covered, similar to standard 3DGS rasterization [S1, unofficial]. The additional cost is the light-space projection and sorting [S1, unofficial].
*   **Memory Complexity:** Linear in the number of Gaussians [S1, unofficial]. No additional voxel grid expansion is required, unlike the voxel fallback [S1, unofficial].
*   **Performance:** Provides full continuous gradients and zero expansion by default, making it more efficient and easier to train than the voxel fallback (`--tlight_voxel`) [S1, unofficial].
*   **Strengths:** Differentiable, no parameter expansion, handles relighting for held-out sun directions [S1, unofficial].
*   **Limitations:** Approximates parallel rays, which may introduce small errors for very large scenes or extreme sun angles [S1, unofficial].

## Background
This method is part of the 4DGS-Volume-Cloud project, which extends 3D Gaussian Splatting to volume cloud rendering [S1, unofficial]. It enables relighting of volume clouds by computing self-shadowing via light-space rasterization [S1, unofficial]. It is designed to work with datasets like `CloudDatasetUniform`, which includes multiple sun directions for training and testing relighting generalization [S1, unofficial].

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
