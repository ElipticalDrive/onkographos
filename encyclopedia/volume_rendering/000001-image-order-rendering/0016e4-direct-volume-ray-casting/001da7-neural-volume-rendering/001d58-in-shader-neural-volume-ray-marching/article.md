# In-Shader Neural Volume Ray Marching

Also called: in-shader rendering

## Overview
In-Shader Neural Volume Ray Marching is a direct volume rendering technique where the neural network (MLP) is evaluated directly inside the CUDA kernel for each sample point during ray marching, rather than pre-decoding the volume into a 3D texture or using sample-streaming [S1, unofficial]. It supports multiple shading modes, including gradient shading and single-shade heuristics, allowing for real-time interactive visualization of neural radiance fields [S1, unofficial]. Unlike [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md), which queries the network for accumulated multiple scattering contributions based on a global shading configuration, this method evaluates the network per sample point during the ray march [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained neural network wrapped in a CUDA device class, camera parameters (position, orientation, field of view), volume bounding box, transfer function, and rendering parameters (mode, shading scale, gradient step) [S1, unofficial].
*   **Outputs:** A 2D framebuffer containing the composited color and alpha for each pixel [S1, unofficial].

**Data Structures:**
The neural network is encapsulated in a CUDA device class [S1, unofficial]. Two primary implementations are supported [S1, unofficial]:
1.  `TcnnDeviceVolume<N, N_FEATURES_PER_LEVEL>`: Wraps a standard MLP. Template parameters define network width ($N \in \{16, 32, 64\}$) and features per level [S1, unofficial].
2.  `FvsrnDeviceVolume<N>`: Wraps an fV-SRN network. Used when `N_FEATURES_PER_LEVEL == -2`. Supports widths from $16 \times 1$ to $16 \times 8$ [S1, unofficial].

Both classes provide `init()`, `launch()`/`launch2D()`, and `sample(p)` methods [S1, unofficial]. The `sample(p)` method performs the neural network inference at position $\mathbf{p}$, returning a scalar value representing the neural volume's output [S1, unofficial].

**Parameters:**
*   `params.mode`: Shading mode. Supported values: `GRADIENT_SHADING`, `SINGLE_SHADE_HEURISTIC` [S1, unofficial].
*   `params.scivis_shading_scale`: Scalar float controlling the interpolation weight between raw sample color and shaded color in gradient mode [S1, unofficial].
*   `params.grad_step`: Step size for numerical gradient estimation via finite differences [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization:**
    Initialize ray origin and direction in world space. The ray origin is transformed from camera space to world space using `xfmPoint(wto, camera.position)`. The ray direction is constructed from the camera's forward, horizontal, and vertical vectors, shifted by the normalized screen position `(screen.x - 0.5f)` and `(screen.y - 0.5f)`, then normalized [S1, unofficial].

2.  **Bounding Box Intersection:**
    Intersect the ray with the volume bounding box using `intersectVolume(t0, t1, ray.org, ray.dir, self)`. If no intersection is found (`!block_any(...)`), return the background color and alpha [S1, unofficial].

3.  **Ray Marching Loop:**
    Iterate through the ray using `network_raymarching_iterator`. For each step interval $[t.x, t.y]$:
    *   **Sample Position:** Compute the sample point $\mathbf{p}$ using jittered linear interpolation: $\mathbf{p} = \mathbf{r}_{org} + \text{lerp}(\text{jitter}, t.x, t.y) \cdot \mathbf{r}_{dir}$ [S1, unofficial].
    *   **Neural Query:** Sample the neural volume at position $\mathbf{p}$ using the sampler: $\text{value} = \text{neuralvolume.sample}(\mathbf{p})$ [S1, unofficial].
    *   **Gradient Estimation (Optional):** If `params.mode == GRADIENT_SHADING`, compute the volume gradient numerically using `sampleGradient(p, value, gradientStep, sampler)` and negate it; otherwise, the gradient is zero [S1, unofficial].
    *   **Transfer Function:** Apply the transfer function to the sampled value to get `sampleColor` and `sampleAlpha` using `sampleTransferFunction(self.tfn, value, sampleColor, sampleAlpha)`, then apply opacity correction for the step size $t.y - t.x$ [S1, unofficial].
    *   **Shading (Optional):** If `params.mode == GRADIENT_SHADING`, transform the gradient to world space $\mathbf{N}_w = \text{xfmNormal}(otw, \text{gradient})$ and the ray direction to world space. Compute the shaded color using `shade_scivis_light` with ambient and directional light parameters. Interpolate between the raw sample color and shaded color using `shadingScale` [S1, unofficial]. If `params.mode == SINGLE_SHADE_HEURISTIC`, track the point with the highest density $\alpha_{max}$ and its position $\mathbf{p}_{max}$ [S1, unofficial].
    *   **Compositing:** Update accumulated color $\mathbf{C}$ and alpha $\alpha$ using front-to-back blending: $\mathbf{C} \leftarrow \mathbf{C} + \tau \cdot \text{sampleColor} \cdot \text{sampleAlpha}$ and $\alpha \leftarrow \alpha + \tau \cdot \text{sampleAlpha}$, where $\tau = 1 - \alpha$ [S1, unofficial].
    *   **Termination Check:** Continue marching while `alive` is true and $\alpha < \text{nearly\_one}$. The iterator callback returns `alive && (alpha < nearly_one)` [S1, unofficial].

4.  **Single Shade Heuristic (Post-March):**
    After marching, if `highestAlpha > 0.f` (Single Shade Heuristic mode), calculate the transmittance from the highest density point to the light source using `network_raymarching_transmittance`, which involves a secondary ray march from `highestOrg` towards the light direction [S1, unofficial]. Apply the single shade heuristic result by interpolating the final color: $\mathbf{C}_{final} = \text{lerp}(\text{shadingScale}, \mathbf{C}, \text{highestColor} \cdot \alpha \cdot \text{transmittance})$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

GRADIENT_SHADING = 0
SINGLE_SHADE_HEURISTIC = 1
NEARLY_ONE = 0.999

def zeros(shape):
    return [[0.0]*shape[1] for _ in range(shape[0])]

def normalize(v):
    mag = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/mag, v[1]/mag, v[2]/mag)

def lerp(a, b, t):
    return a + t * (b - a)

def transform_point(M, p):
    w = M[3][0]*p[0] + M[3][1]*p[1] + M[3][2]*p[2] + M[3][3]
    return (
        (M[0][0]*p[0] + M[0][1]*p[1] + M[0][2]*p[2] + M[0][3]) / w,
        (M[1][0]*p[0] + M[1][1]*p[1] + M[1][2]*p[2] + M[1][3]) / w,
        (M[2][0]*p[0] + M[2][1]*p[1] + M[2][2]*p[2] + M[2][3]) / w
    )

def transform_normal(M, n):
    # Transform normal using inverse-transpose of upper-left 3x3
    inv_t = (
        (M[0][0], M[1][0], M[2][0]),
        (M[0][1], M[1][1], M[2][1]),
        (M[0][2], M[1][2], M[2][2])
    )
    return normalize((
        inv_t[0][0]*n[0] + inv_t[0][1]*n[1] + inv_t[0][2]*n[2],
        inv_t[1][0]*n[0] + inv_t[1][1]*n[1] + inv_t[1][2]*n[2],
        inv_t[2][0]*n[0] + inv_t[2][1]*n[1] + inv_t[2][2]*n[2]
    ))

def intersect_volume_bbox(ray_org, ray_dir, bbox):
    # Simple slab method for AABB [min, max]
    t0, t1 = 0.0, float('inf')
    for i in range(3):
        inv_d = 1.0 / ray_dir[i]
        t_min = (bbox.min[i] - ray_org[i]) * inv_d
        t_max = (bbox.max[i] - ray_org[i]) * inv_d
        t0 = max(t0, min(t_min, t_max))
        t1 = min(t1, max(t_min, t_max))
    return t0, t1

def ray_marching_iterator(t_start, t_end, step_size=0.01):
    t = t_start
    while t < t_end:
        yield (t, min(t + step_size, t_end))
        t += step_size

def random_float():
    return random.random()

def sample_gradient(p, value, grad_step, sampler):
    # Numerical gradient via finite differences
    eps = grad_step
    dx = sampler((p[0]+eps, p[1], p[2])) - value
    dy = sampler((p[0], p[1]+eps, p[2])) - value
    dz = sampler((p[0], p[1], p[2]+eps)) - value
    return (dx/eps, dy/eps, dz/eps)

def shade_scivis_light(N_w, D_w, raw_color):
    # Simple ambient + diffuse shading
    ambient = 0.2
    light_dir = normalize((0.0, 1.0, 0.5))
    diff = max(0.0, N_w[0]*light_dir[0] + N_w[1]*light_dir[1] + N_w[2]*light_dir[2])
    return (
        raw_color[0] * (ambient + diff),
        raw_color[1] * (ambient + diff),
        raw_color[2] * (ambient + diff)
    )

def compute_transmittance(start_p, light_dir, volume):
    # Secondary ray march towards light
    trans = 1.0
    t = 0.0
    while t < 10.0: # Arbitrary max distance
        p = (start_p[0] + t*light_dir[0], start_p[1] + t*light_dir[1], start_p[2] + t*light_dir[2])
        val = volume.sample(p)
        trans *= math.exp(-val * 0.01) # Simplified step
        t += 0.01
    return trans

def apply_tfn(value, step_size):
    # Simplified transfer function + opacity correction
    # Maps scalar value to color and alpha, correcting for step size
    alpha = 1.0 - math.exp(-value * step_size)
    color = (value, value, value) # Placeholder mapping
    return color, alpha

def in_shader_neural_volume_ray_marching(camera, volume, tfn, params):
    width, height = camera.resolution
    framebuffer = zeros((height, width, 4))
    
    cam_pos = camera.position
    f = camera.forward
    h = camera.horizontal
    v = camera.vertical
    W2C = camera.world_to_camera_matrix
    C2W = camera.camera_to_world_matrix
    
    for y in range(height):
        for x in range(width):
            nx = x - 0.5
            ny = y - 0.5
            
            ray_org = transform_point(W2C, cam_pos)
            ray_dir = normalize(f + nx * h + ny * v)
            
            t0, t1 = intersect_volume_bbox(ray_org, ray_dir, volume.bbox)
            
            if t1 <= t0:
                framebuffer[y][x] = (0.0, 0.0, 0.0, 0.0)
                continue
            
            acc_color = (0.0, 0.0, 0.0)
            acc_alpha = 0.0
            highest_alpha = 0.0
            highest_color = (0.0, 0.0, 0.0)
            highest_pos = (0.0, 0.0, 0.0)
            
            alive = True
            
            for t_start, t_end in ray_marching_iterator(t0, t1):
                if not alive:
                    break
                
                jitter = random_float()
                p = (
                    ray_org[0] + lerp(jitter, t_start, t_end) * ray_dir[0],
                    ray_org[1] + lerp(jitter, t_start, t_end) * ray_dir[1],
                    ray_org[2] + lerp(jitter, t_start, t_end) * ray_dir[2]
                )
                
                val = volume.sample(p)
                
                gradient = (0.0, 0.0, 0.0)
                if params.mode == GRADIENT_SHADING:
                    gradient = (-sample_gradient(p, val, params.grad_step, volume.sample)[0],
                                -sample_gradient(p, val, params.grad_step, volume.sample)[1],
                                -sample_gradient(p, val, params.grad_step, volume.sample)[2])
                
                sample_color, sample_alpha = apply_tfn(val, t_end - t_start)
                
                if params.mode == GRADIENT_SHADING:
                    N_w = transform_normal(C2W, gradient)
                    shaded_color = shade_scivis_light(N_w, ray_dir, sample_color)
                    sample_color = (
                        lerp(params.scivis_shading_scale, sample_color[0], shaded_color[0]),
                        lerp(params.scivis_shading_scale, sample_color[1], shaded_color[1]),
                        lerp(params.scivis_shading_scale, sample_color[2], shaded_color[2])
                    )
                elif params.mode == SINGLE_SHADE_HEURISTIC:
                    if sample_alpha > highest_alpha:
                        highest_alpha = sample_alpha
                        highest_color = sample_color
                        highest_pos = p
                
                tr = 1.0 - acc_alpha
                acc_color = (
                    acc_color[0] + tr * sample_color[0] * sample_alpha,
                    acc_color[1] + tr * sample_color[1] * sample_alpha,
                    acc_color[2] + tr * sample_color[2] * sample_alpha
                )
                acc_alpha += tr * sample_alpha
                
                if acc_alpha >= NEARLY_ONE:
                    alive = False
            
            if params.mode == SINGLE_SHADE_HEURISTIC and highest_alpha > 0.0:
                trans = compute_transmittance(highest_pos, (0.0, 1.0, 0.5), volume)
                final_contrib = (
                    highest_color[0] * highest_alpha * trans,
                    highest_color[1] * highest_alpha * trans,
                    highest_color[2] * highest_alpha * trans
                )
                acc_color = (
                    lerp(params.scivis_shading_scale, acc_color[0], final_contrib[0]),
                    lerp(params.scivis_shading_scale, acc_color[1], final_contrib[1]),
                    lerp(params.scivis_shading_scale, acc_color[2], final_contrib[2])
                )
            
            framebuffer[y][x] = (acc_color[0], acc_color[1], acc_color[2], acc_alpha)
    
    return framebuffer
```

## Subtopics

### [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md)
A neural volume rendering technique that uses a multilayer perceptron (MLP) to predict radiance for multiple scattering events in high-order scattering media, such as atmospheric clouds. Unlike [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md), which evaluates the network per sample point during ray marching, RPNN queries the network for the accumulated multiple scattering contribution based on a global shading configuration, enabling results nearly identical to path tracing but approximately a thousand times faster.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of samples per ray. The cost per sample is dominated by the neural network inference (MLP evaluation) [S1, unofficial].
*   **Memory Complexity:** $O(N_{params})$, where $N_{params}$ is the number of parameters in the neural network. The volume data itself is not stored in a 3D texture, reducing VRAM usage compared to voxel-based methods [S1, unofficial].
*   **Performance:** On an NVIDIA GeForce RTX 4080, the 4×64 architecture achieves 3.7 ms frame time on a high-poly mesh and 0.41 ms on a low-poly mesh [S2]. Image quality reaches 22.315 dB PSNR and 0.059 FLIP error for the 4×64 architecture with thickness [S2].
*   **Strengths:** Eliminates pre-computation and texture decoding overhead, supports complex neural representations directly, and provides flexible shading modes [S1, unofficial].
*   **Limitations:** Performance depends heavily on GPU inference speed, requires strict network widths (16, 32, 64 for standard; multiples of 16 for fV-SRN), and numerical gradient estimation adds overhead [S1, unofficial].
*   **When to use:** Interactive visualization of trained neural radiance fields where pre-computation is undesirable [S1, unofficial].
*   **When not to use:** Offline path-traced multi-scattering or when inference latency exceeds the target frame budget [S1, unofficial].

## Background
In-Shader Neural Volume Ray Marching is implemented in the instantvnr library by VIDILabs [S1, unofficial]. It was developed to render neural volume representations directly on the GPU without decoding large volumes into textures, leveraging the tiny-cuda-nn library for efficient MLP evaluation on CUDA [S1, unofficial]. It supports both standard MLPs and fV-SRN architectures [S1, unofficial].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] Environmental Volumetric Neural Shading of Clouds for Real-Time Rendering, https://doi.org/10.1145/3820020, fetched 2026-10-28
