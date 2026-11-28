# UrbanIR

Also called: Urban Scene Inverse Rendering

## Overview
UrbanIR is a method for inverse rendering of large-scale urban scenes from a single monocular video under single illumination, inferring shape, albedo, visibility, and sun/sky illumination to enable relighting and night simulation [S1]. It differs from standard NeRFs by learning intrinsic attributes (albedo, normal) independent of lighting, solving the problem where standard NeRFs bake shading and shadows into observed color, making them non-relightable [S1]. Unlike NeRF-OSR which assumes multiple lighting sources, or FEGR/LightSim which require LiDAR or multiple cameras, UrbanIR uses only single illumination RGB video without LiDAR [S1].

## How it works
**Inputs and Outputs**
Input: A multi-frame video of a scene under single illumination with known camera motion, defined as $\{I_i, E_i, K_i\}$ where $I_i \in \mathbb{R}^{H \times W \times 3}$ is the RGB image, $E_i \in SE(3)$ is the camera pose, and $K_i$ is the camera intrinsic matrix [S1].
Output: A neural scene model encoding albedo, normal, semantics, and density, along with a lighting model (sun, ambient, sky), enabling rendering from novel viewpoints under novel lighting conditions [S1].

**Data Structures**
The scene representation is built on Instant-NGP, a spatial hash-based voxel NeRF representation [S1].
1.  **Dense Feature Hash Table:** Represents the scene geometry and features. It maps 3D positions to high-dimensional feature vectors [S1].
2.  **MLP Headers:** Individual Multi-Layer Perceptrons are used to decode each attribute given a queried feature at point $\mathbf{x}$ [S1]. The attributes are diffuse albedo $\mathbf{a}$, surface normal $\mathbf{n}$, semantic vector $\mathbf{s}$, and density $\sigma$ [S1].
3.  **Lighting Model:**
    *   Sun: 5-DoF representation encoding sun color $\mathbf{L}_{sun}$ along with azimuth $\psi_{sun}$ and zenith $\phi_{sun}$ [S1].
    *   Ambient: 3-DoF Lambert model $\mathbf{L}_{amb}$ [S1].
    *   Sky: Sky dome model inferring sky texture from viewing direction [S1].

**Numbered Steps**
1.  **Ray Sampling:** For each pixel in a target view, generate a ray $r(\lambda) = o_{world} + \lambda d_{world}$ using camera intrinsics and extrinsics [S1]. Sample $N$ points along the ray within the near/far range [S1].
2.  **Feature Query:** For each sample point $\mathbf{x}_i$, query the dense feature hash table to obtain the feature vector [S1].
3.  **Attribute Prediction:** Pass the feature vector through individual MLP headers to predict the intrinsic attributes: albedo $\mathbf{a}_i$, normal $\mathbf{n}_i$, semantics $\mathbf{s}_i$, and density $\sigma_i$ [S1].
4.  **Volume Rendering of Intrinsics:** Accumulate the predicted attributes along the ray using alpha-composition weights [S1]. The rendered albedo $\mathbf{A}(r)$, normal $\mathbf{N}(r)$, and semantics $\mathbf{S}(r)$ are computed as:
    $$ \mathbf{A}(r) = \sum_{i=1}^{N} w_i \mathbf{a}_i, \quad \mathbf{N}(r) = \sum_{i=1}^{N} w_i \mathbf{n}_i, \quad \mathbf{S}(r) = \sum_{i=1}^{N} w_i \mathbf{s}_i $$
    where the weight $w_i$ is defined as:
    $$ w_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) (1 - \exp(-\sigma_i \delta_i)) $$
    and $\delta_i = t_i - t_{i-1}$ [S1].
5.  **Visibility Estimation:** To model shadows without per-scene optimized MLP heads (which are impractical for changing sun positions), compute an estimate $\hat{x}(r)$ of the 3D point being shaded by volume rendering depth (substituting $\hat{t} = \sum w_i t_i$) [S1]. Then, estimate the visibility $V(\hat{x}, \text{sun})$ by computing the transmittance along the ray segment between $\hat{x}$ and the light source using volume rendering [S1].
6.  **Shading and Rendering:** Combine the intrinsic attributes, visibility, and lighting model to predict the final RGB color $\mathbf{C}$:
    $$ \mathbf{C} = \texttt{Shade}(\texttt{Intrinsic}(F_\theta, r), \texttt{Shadow}(F_\theta, r, \mathbf{L}), \mathbf{L}) $$
    where $\mathbf{L}$ is the lighting model [S1].
7.  **Optimization:** Jointly optimize the scene parameters $\theta$ and lighting parameters $\mathbf{L}$ using a loss function combining rendering loss and other regularization terms [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def urbanir_render(ray_origin, ray_direction, lighting_model, scene_hash_table, mlp_headers):
    """
    Renders a single ray using UrbanIR's inverse rendering pipeline.
    """
    # 1. Sample points along the ray
    t_values = sample_points_along_ray(ray_origin, ray_direction, num_samples=128)
    points = ray_origin + ray_direction * t_values[:, None]
    
    # 2. Query features and predict attributes
    features = query_hash_table(scene_hash_table, points)
    albedos = mlp_headers['albedo'](features)
    normals = mlp_headers['normal'](features)
    semantics = mlp_headers['semantics'](features)
    densities = mlp_headers['density'](features)
    
    # 3. Compute alpha-composition weights
    deltas = t_values[1:] - t_values[:-1]
    # Handle first point delta appropriately (e.g., small epsilon or specific near-plane logic)
    deltas = np.insert(deltas, 0, t_values[0] - t_values[0]) # Placeholder for boundary handling [G]
    
    # Compute transmittance and weights
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # w_i = T_i * (1 - exp(-sigma_i * delta_i))
    # Note: Standard NeRF volume rendering formula application [G]
    alphas = 1.0 - torch.exp(-densities * deltas)
    transmittance = torch.cumprod(1.0 - alphas + 1e-10, dim=0)
    weights = alphas * torch.cat([torch.ones_like(transmittance[:1]), transmittance[:-1]])
    
    # 4. Volume render intrinsics
    rendered_albedo = torch.sum(weights[:, None] * albedos, dim=0)
    rendered_normal = torch.sum(weights[:, None] * normals, dim=0)
    rendered_semantics = torch.sum(weights[:, None] * semantics, dim=0)
    
    # 5. Estimate depth for shadowing
    # t_hat = sum(w_i * t_i)
    rendered_depth = torch.sum(weights * t_values)
    shaded_point = ray_origin + ray_direction * rendered_depth
    
    # 6. Estimate visibility to sun
    # Compute transmittance from shaded_point to sun direction
    sun_visibility = estimate_sun_visibility(shaded_point, lighting_model, scene_hash_table, mlp_headers)
    
    # 7. Shade
    final_color = shade_intrinsic(rendered_albedo, rendered_normal, sun_visibility, lighting_model)
    
    return final_color

def estimate_sun_visibility(point, lighting_model, scene_hash_table, mlp_headers):
    """
    Estimates visibility of a point to the sun by volume rendering along the sun ray.
    """
    sun_direction = get_sun_direction(lighting_model)
    # Sample points from point towards sun (or to a large distance)
    sun_ray_t = torch.linspace(0, 100, 64) # Arbitrary large distance [G]
    sun_ray_points = point + sun_direction * sun_ray_t[:, None]
    
    # Query features and densities along sun ray
    sun_features = query_hash_table(scene_hash_table, sun_ray_points)
    sun_densities = mlp_headers['density'](sun_features)
    
    # Compute transmittance (visibility)
    # V = exp(-sum(sigma * delta))
    # Assuming uniform delta for simplicity or compute from t_values [G]
    deltas = sun_ray_t[1:] - sun_ray_t[:-1]
    deltas = np.insert(deltas, 0, 0.0) # Placeholder [G]
    alphas = 1.0 - torch.exp(-sun_densities * deltas)
    transmittance = torch.cumprod(1.0 - alphas + 1e-10, dim=0)
    visibility = transmittance[-1]
    
    return visibility
```

**Parameters**
*   **Input Video:** Single monocular video with known camera motion [S1].
*   **Lighting Model:** Sun (5-DoF), Ambient (3-DoF), Sky [S1].
*   **Scene Representation:** Instant-NGP spatial hash table [S1].

**Edge Cases and Pitfalls**
*   **Visibility Estimation:** Modeling visibility with a per-scene optimized MLP head is impractical because the sun position changes during relighting, but training data only has one sun position [S1]. UrbanIR avoids this by estimating visibility via volume rendering transmittance to the light source [S1].
*   **Standard NeRF Limitations:** Standard NeRFs bake shading, shadow, and specularity into observed color, making them non-relightable. UrbanIR solves this by learning intrinsic attributes independent of lighting [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Uses Instant-NGP's spatial hash-based voxel representation, which offers low memory consumption and high efficiency in training and rendering compared to dense voxel grids [S1].
*   **Measured Performance:** Capable of handling large-scale, unbounded urban scenes from a single video [S1].
*   **Strengths:** Does not require LiDAR or multiple cameras; works with single illumination RGB video; enables relighting and night simulation [S1].
*   **Limitations:** Relies on monocular cues (shadows, surface normals) which may be ambiguous in textureless regions or uniform lighting [S1]. Assumes single illumination during training [S1].

## Background
UrbanIR was introduced by Chih-Hao Lin, Bohan Liu, Yi-Ting Chen, Kuan-Sheng Chen, David Forsyth, Jia-Bin Huang, Anand Bhattad, and Shenlong Wang from the University of Illinois Urbana-Champaign and University of Maryland, College Park [S1]. It was published in 3DV 2025 (arXiv:2306.09349v4, 14 Jan 2025) [S1]. The method addresses the limitation of existing relightable NeRF methods that focus on small bounded indoor environments or require multiple lighting sources/LiDAR [S1].

## Sources
[S1] UrbanIR: Large-Scale Urban Scene Inverse Rendering from a Single Video, https://doi.org/10.1109/3dv66043.2025.00052, fetched 2026-10-30
