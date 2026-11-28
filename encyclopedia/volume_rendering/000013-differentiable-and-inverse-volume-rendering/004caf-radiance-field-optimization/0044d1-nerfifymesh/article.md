# NeRFifyMesh


## Overview
NeRFifyMesh is a pipeline for converting textured 3D mesh models into Neural Radiance Fields (NeRFs) by generating a ground-truth point-based radiance field from mesh geometry and texture [S1, unofficial]. It eliminates the need for multi-view image rendering or camera pose supervision during training, making it suitable for robotics scene building where mesh assets are abundant [S1, unofficial]. Unlike [Neural Radiance Fields](4972-neural-radiance-fields.md) or [Instant-NGP](1414-hierarchical-grid-optimization.md) which optimize photometric loss from images, NeRFifyMesh decouples color, normals, and opacity into separate networks trained directly on mesh-derived samples [S1, unofficial]. This results in lower PSNR but superior perceptual similarity (LPIPS) and explicit surface normal prediction [S1, unofficial].

## How it works
**Inputs:** A textured mesh model $\Omega$ with surface $S$, texture images, and vertex normals [S1, unofficial].
**Outputs:** Three neural networks: $f_\theta$ (color), $f_\phi$ (normals), and $f_\zeta \circ f_\psi$ (opacity), along with saved voxel indices $I$ [S1, unofficial].

**Data Structures:**
1.  **Surface Voxel Grids:** $N$ grids $G_1, \dots, G_N$ [S1, unofficial].
2.  **Hash Tables:** $H_l$ for each grid level $l$ [S1, unofficial].
3.  **Default Latent Vector:** $v_{l, \text{default}}$ for points outside the occupied voxel region of $G_l$ [S1, unofficial].
4.  **Voxel Indices:** $I = [i,j,k]$ for occupied voxels in each $G_l$, saved with the model to allow reconstruction without the original mesh [S1, unofficial].

**Parameters:**
*   $\tau$: Threshold parameter for importance weighting. Typical value: 0.1 [S1, unofficial].
*   $N$: Number of grid levels.
*   Training iterations: 300,000 [S1, unofficial].
*   Batch size: 4,096 samples [S1, unofficial].
*   Samples per mesh: 2 million points (inside, outside, on surface) [S1, unofficial].

**Assumptions:**
1.  Mesh appearance is view-independent (view-direction parameters $\phi, \theta$ are omitted) [S1, unofficial].
2.  Meshes are opaque (opacity is binary) [S1, unofficial].

**Steps:**

1.  **Generate Ground Truth Data:**
    Sample points $x=(x,y,z)$ within the bounding box of $\Omega$ [S1, unofficial]. Calculate Signed Distance Function (SDF) for each point [S1, unofficial]. Let $S$ be the mesh surface and $(x', y', z')$ be the nearest point on $S$ [S1, unofficial].
    *   **Color/Normal GT:**
        $$
        c_{gt}(x) = \begin{cases} c_S(x', y', z') & \text{if } SDF < 0 \\ c_S(x, y, z) & \text{if } SDF = 0 \\ 0 & \text{if } SDF > 0 \end{cases}
        $$
        $$
        \vec{n}_{gt}(x) = \begin{cases} \vec{n}_S(x', y', z') & \text{if } SDF < 0 \\ \vec{n}_S(x, y, z) & \text{if } SDF = 0 \\ 0 & \text{if } SDF > 0 \end{cases}
        $$
        where $c_S$ is texture color and $\vec{n}_S$ is barycentrically interpolated vertex normal [S1, unofficial].
    *   **Opacity GT:**
        $$
        \alpha_{gt}(x) = \begin{cases} 1 & \text{if } SDF \le 0 \\ 0 & \text{if } SDF > 0 \end{cases}
        $$
    *   **Importance Weights:**
        $$
        w_c(x) = w_{\vec{n}}(x) = \begin{cases} \exp(-|SDF|/\tau) & \text{if } SDF \le 0 \\ 0 & \text{if } SDF > 0 \end{cases}
        $$
        $$
        w_\alpha(x) = \begin{cases} \exp(-|SDF|/\tau) & \text{if } SDF \le 0 \\ 1 & \text{if } SDF > 0 \end{cases}
        $$
        These weights prioritize points near the surface and suppress external points [S1, unofficial].

2.  **Initialize Opacity Network:**
    Initialize $N$ surface voxel grids $G_1, \dots, G_N$ from mesh surface points [S1, unofficial]. Assign trainable latent vectors $v_l[i,j,k]$ (dim 2) to grid vertices corresponding to occupied voxels [S1, unofficial]. Save indices $I$ [S1, unofficial].

3.  **Train Networks:**
    *   **Color/Normal Networks:** Apply positional encoding (sine/cosine) to input $x$ [S1, unofficial]. Feed to $f_\theta$ (color) and $f_\phi$ (normals), composed of SIREN layers [S1, unofficial].
    *   **Opacity Network:**
        1.  Query each grid $G_l$: Multiply $x$ by resolution $R_l$ [S1, unofficial]. If $x \in G_l$, use corner indices as hash keys to retrieve $v_l$ from $H_l$ [S1, unofficial]. Else, use $v_{l, \text{default}}$ [S1, unofficial].
        2.  Trilinearly interpolate to get $v'_l(x)$ [S1, unofficial].
        3.  Concatenate: $V(x) = [v'_1(x), \dots, v'_N(x)]$ [S1, unofficial].
        4.  Decode: $f_\psi(V(x))$ [S1, unofficial].
        5.  Predict Opacity: $\alpha(x) = f_\zeta([f_\psi(V(x)), x])$ [S1, unofficial].

4.  **Loss Functions:**
    *   **Color Loss:** $L_c = \frac{1}{n} \sum_{i=1}^n w_{c,i} (c_i - c_{gt,i})^2$ [S1, unofficial].
    *   **Normal Loss:** $L_{\vec{n}} = \frac{1}{n} \sum_{i=1}^n w_{\vec{n},i} (1 - \cos \delta_i)^2$, where $\cos \delta_i = \vec{n}_{gt,i} \cdot \vec{n}_i$ [S1, unofficial].
    *   **Opacity Loss:** $L_\alpha = \lambda_1 L_{BCE} + \lambda_2 L_{reg}$ [S1, unofficial].
        $$
        L_{BCE} = -\frac{1}{n} \sum_{i=1}^n w_{\alpha,i} [\alpha_{gt,i} \log \alpha_i + (1-\alpha_{gt,i}) \log(1-\alpha_i)]
        $$
        $$
        L_{reg} = \sum_{l=1}^N \|v_l\|^2
        $$
        $\lambda_1, \lambda_2$ are weighting terms [S1, unofficial].

5.  **Rendering:**
    Sample points $\{t_i\}_{i=1}^M$ along camera rays with intervals $\delta_i = t_{i+1} - t_i$ [S1, unofficial].
    *   **Transmittance:** $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S1, unofficial].
    *   **Color:** $\hat{C}(r) \approx \sum_{i=1}^M T_i \alpha_i c_i$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(x, num_frequencies=10):
    """Standard NeRF positional encoding."""
    scales = [2**i for i in range(num_frequencies)]
    scaled_x = np.outer(x, scales)
    encoded = np.concatenate([np.sin(scaled_x), np.cos(scaled_x)], axis=-1)
    return np.concatenate([x, encoded], axis=-1)

def trilinear_interpolate(v000, v100, v010, v110, v001, v101, v011, v111, x_frac):
    """Trilinear interpolation of 8 corner vectors."""
    x, y, z = x_frac
    v00 = v000 * (1-x) + v100 * x
    v01 = v010 * (1-x) + v110 * x
    v10 = v001 * (1-x) + v101 * x
    v11 = v011 * (1-x) + v111 * x
    v0 = v00 * (1-y) + v01 * y
    v1 = v10 * (1-y) + v11 * y
    return v0 * (1-z) + v1 * z

def get_opacity_features(x, grids, hash_tables, default_vectors):
    """Get concatenated features for opacity network."""
    features = []
    for l, resolution in enumerate(grids):
        indices = np.floor(x * resolution).astype(int)
        if np.all((indices >= 0) & (indices < resolution)):
            corners = []
            for dx in [0, 1]:
                for dy in [0, 1]:
                    for dz in [0, 1]:
                        idx = indices + np.array([dx, dy, dz])
                        key = tuple(idx)
                        corners.append(hash_tables[l].get(key, default_vectors[l]))
            frac = x * resolution - indices
            v_interp = trilinear_interpolate(*corners, frac)
            features.append(v_interp)
        else:
            features.append(default_vectors[l])
    return np.concatenate(features, axis=-1)

def compute_sdf_weights(sdf, tau=0.1):
    """Compute importance weights based on SDF."""
    abs_sdf = np.abs(sdf)
    w_cn = np.where(sdf <= 0, np.exp(-abs_sdf / tau), 0.0)
    w_alpha = np.where(sdf <= 0, np.exp(-abs_sdf / tau), 1.0)
    return w_cn, w_alpha

def train_step(samples, sdf, gt_color, gt_normal, gt_opacity, 
               grids, hash_tables, default_vectors, 
               color_net, normal_net, opacity_net, tau, lambda1=1.0, lambda2=1.0):
    """Single training step."""
    w_cn, w_alpha = compute_sdf_weights(sdf, tau)
    
    enc_x = positional_encoding(samples)
    pred_color = color_net(enc_x)
    pred_normal = normal_net(enc_x)
    
    features = get_opacity_features(samples, grids, hash_tables, default_vectors)
    opacity_input = np.concatenate([features, samples], axis=-1)
    pred_opacity = opacity_net(opacity_input).squeeze(-1)
    
    color_loss = np.mean(w_cn * (pred_color - gt_color)**2)
    
    cos_sim = np.sum(pred_normal * gt_normal, axis=-1)
    cos_sim = np.clip(cos_sim, -1.0, 1.0)
    normal_loss = np.mean(w_cn * (1 - cos_sim)**2)
    
    eps = 1e-7
    pred_opacity_clamped = np.clip(pred_opacity, eps, 1-eps)
    bce = -np.mean(w_alpha * (gt_opacity * np.log(pred_opacity_clamped) + 
                              (1 - gt_opacity) * np.log(1 - pred_opacity_clamped)))
    reg_loss = 0.0
    for table in hash_tables.values():
        for vec in table.values():
            reg_loss += np.sum(vec**2)
    
    total_opacity_loss = lambda1 * bce + lambda2 * reg_loss
    return color_loss, normal_loss, total_opacity_loss

def render_ray(ray_origin, ray_direction, max_dist, grids, hash_tables, default_vectors,
               color_net, normal_net, opacity_net, num_samples=128):
    """Render a single ray."""
    t_vals = np.linspace(0.0, max_dist, num_samples)
    points = ray_origin + ray_direction * t_vals[:, np.newaxis]
    
    enc_x = positional_encoding(points)
    pred_color = color_net(enc_x)
    
    features = get_opacity_features(points, grids, hash_tables, default_vectors)
    opacity_input = np.concatenate([features, points], axis=-1)
    pred_opacity = opacity_net(opacity_input).squeeze(-1)
    
    transmittance = np.ones(num_samples)
    for i in range(1, num_samples):
        transmittance[i] = transmittance[i-1] * (1 - pred_opacity[i-1])
        
    weights = transmittance * pred_opacity
    final_color = np.sum(weights[:, np.newaxis] * pred_color, axis=0)
    return final_color
```

## Subtopics

### [Scene Composition](11397-scene-composition.md)
A method for assembling multiple independent radiance-field models (specifically NeRFifyMesh or iVR-GS representations) into a single unified global scene without retraining. It solves the problem of scalable scene building by allowing pre-converted mesh-based models to be placed in a global coordinate system or integrated with existing NeRF models. It differs from monolithic training approaches by treating each model as an independent, transformable primitive that is composited at render time via depth-based blending.

### [Vosh Hybrid Rendering](15204-vosh-hybrid-rendering.md)
A hybrid representation method that combines a textured mesh with a sparse voxel grid to achieve real-time view synthesis while preserving comprehensive scene information. It solves the trade-off between the geometric precision of meshes and the volumetric completeness of neural radiance fields by using the mesh for surface details and the voxels for thin structures and background volume. Unlike Adaptive Shells, which constrain volume rendering to regions near the surface, Vosh allows pure volume rendering where necessary, making it suitable for outdoor scenes. It differs from VMesh by reconstructing a volumetric density field, enabling the mesh to represent thin structures like fishing nets that are difficult to capture with Signed Distance Fields (SDFs).

### [MobileNeRF](15554-mobilenerf.md)
A neural radiance field representation that synthesizes novel images efficiently using standard polygon rasterization pipelines, avoiding specialized volumetric ray marching. It represents the NeRF as a set of textured polygons with binary opacities and feature vectors, interpreted by a small view-dependent MLP in a fragment shader to produce final pixel color. Unlike classic NeRF which uses volumetric density compositing, MobileNeRF uses opacity compositing on a mesh. Unlike semi-transparent mesh rendering which requires per-frame sorting, MobileNeRF uses binary opacity and z-buffering, which is order-independent. It is 10x faster than SNeRG with the same output quality and consumes less memory by storing surface textures instead of volumetric textures.

### [Coordinate Rectification for Point Cloud Rendering](15912-coordinate-rectification-for-point-cloud-rendering.md)
A surface estimation technique used in point cloud rendering to determine the precise 3D position of a ray-surface intersection from discrete point data. It solves the problem of inaccurate depth sampling in sparse or noisy point clouds by correcting the raw depth value based on the angle between the viewing ray and the scene's primary axis. Unlike standard depth buffering which assumes a planar intersection perpendicular to the view ray, this method accounts for surface orientation, enabling photorealistic rendering from discrete points by providing accurate input coordinates for subsequent neural radiance mapping. It serves as the foundational geometry estimation step in the End-to-End Rendering Pipeline, preceding the frequency-modulated radiance mapping stage.

### NeuMesh Texture Filling
A method to fill a user-selected area on a neural implicit field with a texture template (e.g., furry hair or golden metal) from a pre-captured object model [S2]. First, the target UV-map of the selected area is obtained by unwrapping in Blender [S2]. A texture template from a pre-trained NeuMesh model (e.g., a small squared patch with ~10 vertices) is repeatedly filled into the target UV-map in a sliding-window manner [S2]. Texture codes in target vertices are assigned with interpolated codes from the template, and the radiance decoder is bound to the one from the pretrained model because target and template texture codes do not share the same latent radiance space [S2]. For smooth transitions near the area boundary, color contributions from different decoders are fused with inverse distance weighting for query points having texture codes/decoders from different sources [S2].

### NeRF Surface Structure Attachment to Mesh
A method to attach realistic surface structures modeled by NeRFs onto a 3D mesh by deforming a voxel grid and using a deformation field [S3]. First, a scene is synthesized on a plane with desired lighting using a specific shading map guider [S3]. The corresponding voxel grid is deformed to match the 3D mesh using a deformation field $\Psi : x \rightarrow \Delta x$, which maps points in the deformed space to the canonical space [S3]. The deformation field is bent to synthesize a large-scale $V(d)$ onto the 3D mesh, and a ray-tracing-based method computes the shading map guider on the surface [S3]. The shading map guider synthesizes a new scene with desired lighting and deforms it to match the target mesh surface [S3]. The ARAP parametric method establishes the mapping between the 3D mesh and the undersurface of the scene, defining corresponding directions for each vertex along the normal direction to collect discrete corresponding points [S3]. A neural network fits the deformation field using MSE loss on the coordinates of corresponding points as supervision [S3].

## Operating characteristics
**Time and Memory Complexity:**
*   Training time: ~2 hours per model on an NVIDIA P100 GPU with 50 GB RAM [S1, unofficial].
*   Model Size: 4.7 ± 0.2 MB on the Objaverse dataset, achieving a 14.4x compression ratio compared to the original mesh size of 67.3 ± 60.6 MB [S1, unofficial].
*   Memory usage depends on the number of hash table entries and grid levels [S1, unofficial].

**Measured Performance (Objaverse dataset):**
*   PSNR: 31.70 (lower than Instant-NGP 35.62 and NeRF 32.20) [S1, unofficial].
*   LPIPS: 0.037 (better than Instant-NGP 0.043 and NeRF 0.083) [S1, unofficial].
*   SSIM: 0.958 (better than NeRF 0.927, lower than Instant-NGP 0.965) [S1, unofficial].
*   Surface Normal Accuracy: RMSAE of 4.59 ± 1.84 degrees [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** No need for multi-view images or camera poses. Explicit surface normal prediction enables relighting [S1, unofficial]. High perceptual quality (LPIPS). Significant model compression.
*   **Limitations:** Lower PSNR compared to image-optimized methods. Assumes view-independent appearance and opaque meshes [S1, unofficial]. Requires watertight meshes for SDF calculation [S1, unofficial].

**When to use:**
*   When converting existing mesh assets (e.g., from Objaverse, Poly Haven) to NeRFs for robotics or simulation [S1, unofficial].
*   When explicit normals are needed for shading/relighting [S1, unofficial].
*   When model size is a constraint.

**When not to use:**
*   When photometric accuracy (PSNR) is the primary metric [S1, unofficial].
*   For scenes with translucent objects or view-dependent effects (e.g., glossy reflections) [S1, unofficial].

## Background
NeRFifyMesh was introduced to address the need for tools to convert existing 3D mesh models into NeRF representations for rapid algorithm development and testing, particularly in robotics [S1, unofficial]. It builds on the principles of [Radiance-field optimization](44-radiance-field-optimization.md) but diverges by using mesh-derived ground truth instead of image-based optimization [S1, unofficial]. The method leverages hash encoding inspired by [Instant-NGP](1414-hierarchical-grid-optimization.md) and SIREN networks for color/normal prediction [S1, unofficial].

## Sources
[S1] NeRFifyMesh: Optimizing Neural Radiance Fields from Textured Meshes for Robotics Scene Building, https://doi.org/10.48550/arxiv.2610.10387, fetched 2026-10-26 (unofficial)
[S2] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
[S3] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
