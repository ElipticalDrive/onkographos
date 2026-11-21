# Mirror-NeRF

Also called: Mirror-NeRF: Learning Neural Radiance Fields for Mirrors with Whitted-Style Ray Tracing

## Overview
Mirror-NeRF is a neural rendering framework that learns accurate mirror geometry and reflection by unifying the radiance field and tracing rays according to Whitted-style light transport, enabling novel view synthesis and scene manipulation [S1]. Unlike NeRF and NeRFReN, which treat mirror reflections as separate virtual scenes or interpolate memorized reflections, Mirror-NeRF uses a unified radiance field with explicit reflection probability and Whitted-style ray tracing to synthesize physically consistent reflections from novel viewpoints [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses [S1].
*   **Outputs:** A rendered image for a target camera pose, including the direct view and physically accurate reflections [S1].

**Data Structures:**
The method uses a unified set of Multi-Layer Perceptrons (MLPs) to represent scene properties at any 3D spatial location $\mathbf{x}$ and view direction $\mathbf{d}$ [S1]. The fields are:
1.  **Geometry MLP ($F_{geo}$):** Takes positional encoded position $\gamma_x(\mathbf{x})$ to output a geometry feature vector $f_{geo}$ [S1].
2.  **Density MLP ($F_\sigma$):** Takes $f_{geo}$ to output volume density $\sigma$ [S1].
3.  **Radiance MLP ($F_c$):** Takes $f_{geo}$ and positional encoded view direction $\gamma_d(\mathbf{d})$ to output radiance color $c$ [S1].
4.  **Normal MLP ($F_n$):** Takes $f_{geo}$ to output a smoothed surface normal $\tilde{\mathbf{n}}$ [S1]. This is used to avoid noise from analytical gradients of volume density [S1].
5.  **Reflection Probability MLP ($F_m$):** Takes $f_{geo}$ to output a scalar reflection probability $m \in [0, 1]$ [S1].

**Numbered Steps:**
1.  **Ray Generation:** For each pixel in the target image, generate a camera ray $\mathbf{r}$ originating from the camera center with direction $\mathbf{d}$ [S1].
2.  **Sampling:** Sample $N$ points along the ray $\mathbf{r}$ at positions $\mathbf{x}_i$ and distances $\delta_i$ [S1].
3.  **Field Query:** For each sample point $\mathbf{x}_i$:
    *   Compute geometry feature: $f_{geo} = F_{geo}(\gamma_x(\mathbf{x}_i))$ [S1].
    *   Compute density: $\sigma_i = F_\sigma(f_{geo})$ [S1].
    *   Compute radiance: $c_i = F_c(f_{geo}, \gamma_d(\mathbf{d}))$ [S1].
    *   Compute reflection probability: $m_i = F_m(f_{geo})$ [S1].
    *   Compute normal: $\tilde{\mathbf{n}}_i = F_n(f_{geo})$ [S1].
4.  **Ray Tracing and Reflection:**
    *   Determine if the ray hits a mirror surface by evaluating the reflection probability $m_i$ and density $\sigma_i$ [S1].
    *   When a ray intersects a surface point with high reflection probability (indicating a mirror), trigger physical reflection [S1].
    *   Calculate the reflected ray direction $\mathbf{r}'$ using the surface normal $\tilde{\mathbf{n}}$ and the incident ray direction, following Whitted-style ray tracing [S1].
    *   Trace the reflected ray $\mathbf{r}'$ through the scene to accumulate radiance from the reflection [S1].
5.  **Volume Rendering Accumulation:**
    *   Compute transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].
    *   Compute opacity $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S1].
    *   The final color $\hat{C}(\mathbf{r})$ is a blend of the direct view color and the reflected color, weighted by the reflection probability [S1]. The standard volume rendering equation is modified to include the reflection component [S1]:
        $$ \hat{C}(\mathbf{r}) = (1 - m_{hit}) C_{direct} + m_{hit} C_{refl} $$
        where $C_{direct} = \sum_{i=1}^N T_i \alpha_i c_i$ is the volume-rendered direct color, $m_{hit}$ is the reflection probability at the hit point, and $C_{refl}$ is the accumulated color of the reflected ray [S1].
6.  **Optimization:**
    *   Optimize the parameters of all MLPs ($F_{geo}, F_\sigma, F_c, F_n, F_m$) using a loss function [S1].
    *   The loss includes photometric reconstruction error ($L_{photo}$) between rendered and ground truth images [S1].
    *   **Plane Consistency Constraint:** Enforce that the mirror surface lies on a plane to reduce ambiguity in mirror geometry reconstruction [S1].
    *   **Forward-Facing Normal Constraint:** Enforce that the surface normals of the mirror face towards the camera to reduce reflection ambiguity [S1].
    *   **Progressive Training:** Apply a progressive training strategy to stabilize the optimization of the mirror geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def positional_encoding(x, num_freqs=10):
    pe = []
    for i in range(num_freqs):
        freq = 2**i * math.pi
        pe.append(math.sin(freq * x))
        pe.append(math.cos(freq * x))
    return sum(pe)  # Concatenation of sin/cos terms

def query_fields(x, F_geo, F_sigma, F_c, F_n, F_m, d):
    x_pe = positional_encoding(x)
    f_geo = F_geo(x_pe)
    sigma = F_sigma(f_geo)
    c = F_c(f_geo, positional_encoding(d))
    n = F_n(f_geo)
    m = F_m(f_geo)
    return sigma, c, n, m, f_geo

def render_ray(ray_o, ray_d, N_samples, near, far, F_geo, F_sigma, F_c, F_n, F_m):
    # 1. Sample points along ray
    t_vals = [i / (N_samples - 1) for i in range(N_samples)]
    z_vals = [near + t * (far - near) for t in t_vals]
    points = [[ray_o[k] + ray_d[k] * z for k in range(3)] for z in z_vals]
    
    # 2. Query fields at each sample
    sigma_list, c_list, n_list, m_list = [], [], [], []
    for i in range(N_samples):
        s, c, n, m, _ = query_fields(points[i], F_geo, F_sigma, F_c, F_n, F_m, ray_d)
        sigma_list.append(s)
        c_list.append(c)
        n_list.append(n)
        m_list.append(m)
        
    sigma = sigma_list
    c = c_list
    n = n_list
    m = m_list
    
    # 3. Compute deltas, transmittance, opacity
    deltas = [z_vals[i+1] - z_vals[i] for i in range(N_samples - 1)]
    deltas.append(1e10)  # Last interval is infinite
    alpha = [1.0 - math.exp(-sigma[i] * deltas[i]) for i in range(N_samples)]
    
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    T = [1.0]
    for i in range(1, N_samples):
        T.append(T[-1] * math.exp(-sigma[i-1] * deltas[i-1]))
        
    # 4. Direct color accumulation
    weights = [T[i] * alpha[i] for i in range(N_samples)]
    C_direct = [sum(weights[i] * c[i][k] for i in range(N_samples)) for k in range(3)]
    
    # 5. Mirror hit detection & reflection
    # Find point with highest combined reflection probability and density
    hit_idx = max(range(N_samples), key=lambda i: m[i] * sigma[i])
    m_hit = m[hit_idx]
    n_hit = n[hit_idx]
    T_hit = T[hit_idx]
    
    # Reflect ray direction: r' = r - 2(r.n)n
    dot_val = sum(ray_d[k] * n_hit[k] for k in range(3))
    ray_d_reflect = [ray_d[k] - 2.0 * dot_val * n_hit[k] for k in range(3)]
    ray_o_reflect = points[hit_idx]
    
    # Trace reflected ray (simplified to one bounce)
    C_refl = render_reflected_ray(ray_o_reflect, ray_d_reflect, N_samples, near, far, F_geo, F_sigma, F_c, F_n, F_m)
    
    # 6. Blend direct and reflected colors
    C_final = [(1.0 - m_hit) * C_direct[k] + m_hit * C_refl[k] for k in range(3)]
    return C_final

def render_reflected_ray(ray_o, ray_d, N_samples, near, far, F_geo, F_sigma, F_c, F_n, F_m):
    # Reuses render_ray logic for the reflected ray direction
    return render_ray(ray_o, ray_d, N_samples, near, far, F_geo, F_sigma, F_c, F_n, F_m)
```

**Parameters with Typical Values:**
*   Number of samples per ray ($N$): Not given in the sources.
*   Positional encoding frequencies: Not given in the sources.
*   Loss weights ($\lambda_{plane}, \lambda_{normal}$): Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Mirror Ambiguity:** The "appearance" of a mirror comes from other objects and is not consistent from different viewpoints, making reconstruction ambiguous [S1]. This is mitigated by plane consistency and forward-facing normal constraints [S1].
*   **Unobserved Reflections:** Standard NeRF cannot infer reflections unobserved in training views (e.g., missing ceiling) [S1]. Mirror-NeRF handles this by using accurate depth and ray tracing [S1].
*   **Noise in Normals:** Analytical gradients of volume density can be noisy [S1]. A separate MLP is used to parameterize the surface normal distribution to ensure smoothness [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:**
    *   Physically consistent reflections from novel viewpoints [S1].
    *   Accurate mirror geometry reconstruction [S1].
    *   Supports scene manipulation (adding objects/mirrors, controlling roughness) [S1].
*   **Limitations:**
    *   Relies on the assumption that mirrors are planar [S1].
    *   Requires careful tuning of regularization constraints to stabilize training [S1].
*   **When to Use:** When accurate mirror geometry and physically correct reflections are required for novel view synthesis or scene manipulation [S1].
*   **When Not to Use:** When mirrors are non-planar or when computational resources are extremely limited and approximate reflections are acceptable [G].

## Background
Mirror-NeRF was introduced by Junyi Zeng, Chong Bao, et al. from Zhejiang University and the State Key Lab of CAD&CG [S1]. The paper was presented at MM '23, October 29-November 3, 2023, Ottawa, ON, Canada [S1]. It addresses the limitations of NeRF and NeRFReN in handling mirrors by introducing a unified radiance field with explicit reflection probability and Whitted-style ray tracing [S1].

## Sources
[S1] Mirror-NeRF: Learning Neural Radiance Fields for Mirrors with Whitted-Style Ray Tracing, https://arxiv.org/abs/2308.03280v1, fetched 2026-10-26
