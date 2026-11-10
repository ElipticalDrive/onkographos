# Neural Surfel Radiance Fields

Also called: SurfelNeRF

## Overview
Neural Surfel Radiance Fields (SurfelNeRF) is an online photorealistic reconstruction method that represents a scene as a dynamic collection of neural surfels, combining explicit geometric attributes with neural appearance features [S1]. Unlike standard Neural Radiance Fields (NeRF) which rely on offline optimization and volumetric sampling, SurfelNeRF progressively integrates incoming image streams into a global model using a differentiable rasterization-based rendering scheme, achieving significant speedups in both training and inference [S1]. It solves the problem of real-time, large-scale indoor scene reconstruction by using a differentiable rasterization scheme that evaluates density and radiance only at ray-surfel intersection points [S1].

## How it works
SurfelNeRF operates in three main phases: local surfel construction, online integration (association and fusion), and rasterization-guided rendering [S1].

### Inputs and Outputs
**Inputs:**
- Input image frame $I_t$ at time $t$ [S1].
- Camera parameters $c_t$ (extrinsics and intrinsics) [S1].
- Estimated depth map $D_t$ (from sensors or MVS) [S1].

**Outputs:**
- Global neural surfel radiance field $S_g$ (updated set of surfels) [S1].
- Rendered color image $\hat{C}$ for novel views [S1].

### Data Structures
The scene is represented by a set of neural surfels $S_g$ [S1]. Each surfel $s_i$ stores the following attributes [S1]:
- Position $\mathbf{p}^i \in \mathbb{R}^3$
- Normal $\mathbf{n}^i \in \mathbb{R}^3$
- Radius $r^i \in \mathbb{R}$
- Weight $w^i \in \mathbb{R}$ (confidence based on depth measurement distance)
- Neural feature vector $\mathbf{f}^i \in \mathbb{R}^N$ (extracted from input images)

### Step 1: Local Neural Surfel Radiance Field Construction
For each incoming frame $I_t$, a local set of surfels $S_t$ is constructed [S1]:
1. **Depth Refinement:** If $D_t$ is noisy or incomplete, refine it using a U-Net structure [S1].
2. **Unprojection:** Unproject pixels from $I_t$ to 3D space using $c_t$ and $D_t$ to obtain 3D positions [S1].
3. **Attribute Estimation:** Estimate normal, weight, and radius for each point [S1]. The weight $w^i$ measures confidence based on the distance of the depth measurement from the camera center [S1]. The radius $r^i$ represents the local support area [S1].
4. **Feature Extraction:** Extract multi-scale view-dependent features from $I_t$ using a MasNet [S1].
5. **Feature Projection:** Project the 2D image features to 3D surfel features $\mathbf{f}^i$ using an MLP [S1].

### Step 2: Neural Surfel Association and Fusion
Local surfels $S_t$ are integrated into the global field $S_g$ using a neural network $N_f$ parameterizing the fusion process [S1].

**Association:**
1. Project each global surfel in $S_g$ as an ellipse onto the image plane of $I_t$ [S1].
2. For each pixel, gather the top-$K$ global surfels overlapping the pixel after rasterization [S1]. Discard irrelevant surfels using a normal filter [S1].
3. Compare the depth distance between the local surfel and the top-$M$ global surfel candidates (sorted near to far) [S1].
4. If the minimum depth distance between a local surfel and its global candidates is less than a threshold $\delta_{depth}$, mark it for **Merge**. Otherwise, mark it for **Add** [S1].
   - Typical values: $K=8$, $\delta_{depth}=0.1\text{m}$, $M=80$ [S1].

**Fusion:**
- **Add:** Directly add the new local surfel to $S_g$ [S1].
- **Merge:** Update the corresponding global surfel $s_j$ using the local surfel $s_i$ [S1]:
  1. **Geometric Update:** Compute weighted sums for position, normal, and radius [S1]:
     $$
     \mathbf{p}_t^j = \frac{w_t^i \mathbf{p}_t^i + w_{t-1}^j \mathbf{p}_{t-1}^j}{w_t^i + w_{t-1}^j}
     $$
     $$
     \mathbf{n}_t^j = \frac{w_t^i \mathbf{n}_t^i + w_{t-1}^j \mathbf{n}_{t-1}^j}{w_t^i + w_{t-1}^j}
     $$
     $$
     r_t^j = \frac{w_t^i r_t^i + w_{t-1}^j r_{t-1}^j}{w_t^i + w_{t-1}^j}
     $$
     $$
     w_t^j = w_t^i + w_{t-1}^j
     $$
  2. **Feature Update:** Update the neural feature $\mathbf{f}_t^{corrs}$ using a Gated Recurrent Unit (GRU) [S1]. The GRU takes the merged input feature $\mathbf{f}_t^{merge}$ and the previous global feature $\mathbf{f}_{t-1}^{corrs}$ as inputs [S1]:
     $$
     \mathbf{f}_t^{corrs} = \text{GRU}(\mathbf{f}_t^{merge}, \mathbf{f}_{t-1}^{corrs})
     $$
     The GRU implementation uses three one-layer MLPs ($M_z, M_r, M_t$) followed by sigmoid, sigmoid, and tanh activations respectively [S1]:
     $$
     \mathbf{z}_t = M_z([\mathbf{f}_t^{merge}, \mathbf{f}_{t-1}^{corrs}])
     $$
     $$
     \mathbf{r}_t = M_r([\mathbf{f}_t^{merge}, \mathbf{f}_{t-1}^{corrs}])
     $$
     $$
     \tilde{\mathbf{f}}_t^{corrs} = M_t([\mathbf{r}_t * \mathbf{f}_{t-1}^{corrs}, \mathbf{f}_t^{merge}])
     $$
     $$
     \mathbf{f}_t^{corrs} = (1 - \mathbf{z}_t) * \mathbf{f}_{t-1}^{corrs} + \mathbf{z}_t * \tilde{\mathbf{f}}_t^{corrs}
     $$
     where $[\cdot, \cdot]$ denotes concatenation and $*$ denotes element-wise multiplication [G].

### Step 3: Rendering
Rendering uses a differentiable rasterization scheme to project surfels onto the target image plane [S1]. For a pixel $p_x$, gather the $M$ surfels $\{s_i\}$ that cover it [S1]. Render the color using a volume rendering formula evaluated only at the intersection points (shading points) $\{x_i\}$ between the ray and the surfels [S1]:

$$
\hat{C}(p_x) = \sum_{i=1}^M \tau_i (1 - \exp(-\sigma_i \Delta_i)) c_i
$$

where $\tau_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \Delta_j\right)$ is the accumulated transmittance, $\sigma_i$ and $c_i$ are the volume density and radiance at shading point $x_i$, and $\Delta_j$ is the distance between adjacent shading points [S1].

The density $\sigma_i$ and radiance $c_i$ are predicted by an MLP-like module $\text{Render}$ [S1]:
$$
\sigma_i, c_i = \text{Render}(x_i, \mathbf{f}^i(x_i), \mathbf{d})
$$
where $\mathbf{d}$ is the view direction and $\mathbf{f}^i(x_i)$ is the interpolated surfel feature [S1].

The interpolated feature $\mathbf{f}^i(x_i)$ is computed as [S1]:
$$
\mathbf{f}^i(x_i) = \frac{r^i - \|x^i - p^i\|}{r^i} \text{F}(\mathbf{f}^i, \mathbf{d}, \mathbf{n}^i, w^i)
$$
where $\text{F}$ is a two-layer MLP with ReLU activation and hidden channel size 256 [S1]. The input to $\text{F}$ is the concatenation of the surfel feature $\mathbf{f}^i$, positional embeddings $\gamma(\cdot)$ (max frequency 5) of the view direction $\mathbf{d}$, weight $w^i$, normal $\mathbf{n}^i$, and the difference $\mathbf{d}-\mathbf{n}$ [S1]:
$$
\text{F}(\mathbf{f}^i, \mathbf{d}, \mathbf{n}^i, w^i) = F_f([\mathbf{f}^i, \gamma(\mathbf{d}), \gamma(w^i), \gamma(\mathbf{n}^i), \gamma(\mathbf{d}-\mathbf{n})])
$$

### Training and Optimization
The model is trained end-to-end using the following loss functions [S1]:
1. **Rendering Loss:** L2 loss between ground truth color $C_{gt}$ and rendered color $C_r$ [S1]:
   $$
   \mathcal{L}_{render} = \| C_{gt} - C_r \|_2^2
   $$
2. **Depth Loss:** L1 loss between sensor depth $D_i^{sen}$ and refined depth $D^i$, masked by valid depth mask $M_i$ [S1]:
   $$
   \mathcal{L}_d = \| D_i^{sen} \odot M_i - D^i \odot M_i \|_1
   $$
3. **Total Loss:**
   $$
   \mathcal{L} = \mathcal{L}_{render} + \lambda \cdot \mathbbm{1}(\mathcal{L}_d)
   $$
   where $\mathbbm{1}$ is an indicator function (1 if sensor depth is available) and $\lambda=0.1$ [S1].

**Parameters:**
- Optimizer: Adam [S1].
- Initial Learning Rate: 0.001 (training), 0.0002 (per-scene fine-tuning) [S1].
- Image Resolution: 640x480 [S1].
- Keyframe Sampling: 5% of the sequence [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class SurfelNeRF:
    def __init__(self, feature_dim=32, hidden_dim=256):
        self.gru = GRUUnit(input_dim=feature_dim, hidden_dim=feature_dim)
        # Feature projection MLP [S1]
        self.feature_proj = nn.Linear(83, feature_dim)
        # Rendering MLP [S1]
        self.render_mlp = RenderMLP(feature_dim=feature_dim, hidden_dim=hidden_dim)
        self.global_surfels = []
        self.K = 8
        self.M = 80
        self.delta_depth = 0.1
        self.lambda_depth = 0.1

    def construct_local_surfels(self, image, depth, camera_params):
        # Depth refinement via U-Net omitted [G]
        refined_depth = depth
        # Unproject pixels to 3D using pinhole camera model [G]
        positions, normals, weights, radii = unproject_pixels(refined_depth, camera_params)
        # Feature extraction via MasNet omitted [G]
        image_features = torch.randn(positions.shape[0], 83)
        surfel_features = self.feature_proj(image_features)
        local_surfels = []
        for i in range(len(positions)):
            local_surfels.append({
                'p': positions[i], 'n': normals[i], 'r': radii[i], 
                'w': weights[i], 'f': surfel_features[i]
            })
        return local_surfels

    def associate_and_fuse(self, local_surfels):
        new_global_surfels = []
        updated_global_surfels = []
        for local_s in local_surfels:
            candidates = find_overlapping_global_surfels(local_s, self.global_surfels, self.K)
            if not candidates:
                new_global_surfels.append(local_s)
            else:
                candidates.sort(key=lambda x: x[1])
                min_dist = candidates[0][1]
                if min_dist < self.delta_depth:
                    global_s, _ = candidates[0]
                    w_sum = local_s['w'] + global_s['w']
                    new_p = (local_s['w'] * local_s['p'] + global_s['w'] * global_s['p']) / w_sum
                    new_n = (local_s['w'] * local_s['n'] + global_s['w'] * global_s['n']) / w_sum
                    new_r = (local_s['w'] * local_s['r'] + global_s['w'] * global_s['r']) / w_sum
                    new_w = w_sum
                    new_f = self.gru(local_s['f'], global_s['f'])
                    updated_global_surfels.append({
                        'p': new_p, 'n': new_n, 'r': new_r, 'w': new_w, 'f': new_f
                    })
                else:
                    new_global_surfels.append(local_s)
        self.global_surfels = updated_global_surfels + new_global_surfels
        return self.global_surfels

    def render(self, camera_params, target_image_shape):
        shading_points_data = rasterize_surfels(self.global_surfels, camera_params, target_image_shape)
        rendered_image = torch.zeros(target_image_shape)
        for px, points in shading_points_data.items():
            if not points: continue
            points.sort(key=lambda x: x[2])
            points = points[:self.M]
            transmittance = 1.0
            color = torch.zeros(3)
            for point, surfel_idx, _ in points:
                surfel = self.global_surfels[surfel_idx]
                x_i = point
                d = get_view_direction(camera_params, x_i)
                dist = torch.norm(x_i - surfel['p'])
                scale = (surfel['r'] - dist) / surfel['r']
                f_interp = scale * self.render_mlp.get_feature_part(surfel['f'], d, surfel['n'], surfel['w'])
                sigma, c = self.render_mlp(x_i, f_interp, d)
                delta_i = 0.01 # Placeholder for distance between shading points along ray [G]
                alpha = 1.0 - torch.exp(-sigma * delta_i)
                weight = transmittance * alpha
                color += weight * c
                transmittance *= (1.0 - alpha)
                if transmittance < 1e-4: break
            rendered_image[px] = color
        return rendered_image

class GRUUnit(nn.Module):
    def __init__(self, input_dim, hidden_dim):
        super().__init__()
        self.M_z = nn.Sequential(nn.Linear(2 * hidden_dim, hidden_dim), nn.Sigmoid())
        self.M_r = nn.Sequential(nn.Linear(2 * hidden_dim, hidden_dim), nn.Sigmoid())
        self.M_t = nn.Sequential(nn.Linear(2 * hidden_dim, hidden_dim), nn.Tanh())
        
    def forward(self, x, h):
        concat = torch.cat([x, h], dim=-1)
        z = self.M_z(concat)
        r = self.M_r(concat)
        concat2 = torch.cat([r * h, x], dim=-1)
        tilde_h = self.M_t(concat2)
        new_h = (1 - z) * h + z * tilde_h
        return new_h

class RenderMLP(nn.Module):
    def __init__(self, feature_dim, hidden_dim):
        super().__init__()
        # Input: f(32) + gamma(d)(11) + gamma(w)(11) + gamma(n)(11) + gamma(d-n)(11) = 76
        self.F_f = nn.Sequential(
            nn.Linear(76, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 4) # sigma(1) + c(3)
        )
        
    def forward(self, x, f_surfel, d):
        pass

    def get_feature_part(self, f, d, n, w):
        # Compute interpolated feature f^i(x_i)
        # Requires x_i, which is passed in render loop
        pass

def unproject_pixels(depth, camera_params):
    # Standard pinhole unprojection [G]
    K, R, t = camera_params['K'], camera_params['R'], camera_params['t']
    H, W = depth.shape
    y, x = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    ones = torch.ones_like(depth)
    pix = torch.stack([x, y, ones], dim=-1).float()
    pix_cam = torch.inverse(K) @ pix.view(-1, 3).T
    pts_cam = pix_cam * depth.view(-1)
    pts_world = (R.T @ (pts_cam - t.view(3, -1))).T
    normals = torch.zeros_like(pts_world) # Placeholder for normal estimation [G]
    weights = torch.ones_like(pts_world[:, :1]) # Placeholder for weight estimation [G]
    radii = torch.ones_like(pts_world[:, :1]) # Placeholder for radius estimation [G]
    return pts_world, normals, weights, radii

def rasterize_surfels(surfels, camera_params, img_shape):
    # Project surfels as ellipses, gather overlaps per pixel [G]
    shading_points = {}
    for i, s in enumerate(surfels):
        # Simplified projection logic [G]
        pass
    return shading_points

def find_overlapping_global_surfels(local_s, global_surfels, K):
    # Project global surfels, check overlap, sort by depth [G]
    candidates = []
    for g in global_surfels:
        dist = torch.norm(local_s['p'] - g['p'])
        candidates.append((g, dist))
    candidates.sort(key=lambda x: x[1])
    return candidates[:K]

def get_view_direction(camera_params, point):
    # d = camera_center - point [G]
    cam_center = -camera_params['R'].T @ camera_params['t']
    return (cam_center - point).view(-1)

def positional_embedding(x, max_freq=5):
    # Standard NeRF positional embedding [G]
    scales = 2 ** torch.arange(max_freq)
    emb = torch.cat([x * s for s in scales], dim=-1)
    return torch.cat([torch.sin(emb), torch.cos(emb), x], dim=-1)
```

### Edge Cases and Pitfalls
- **Depth Quality:** Inaccurate or noisy estimated depth, as well as depth discontinuity, can cause geometry discontinuities or missing parts in the surfel representation [S1]. A depth refinement network is required to mitigate this [S1].
- **Incomplete Depth:** Methods relying solely on captured depths fail when depth is incomplete [S1].
- **Precision:** The method uses float32 for positions and features. The radius and weight are scalars.

## Operating characteristics
- **Time Complexity:** The major computation cost comes from rasterization. The number of shading points per pixel is small (average 2.92, max 10 on ScanNet) compared to volume rendering (hundreds of points) [S1]. This results in a 10× speedup in training and inference time compared to conventional NeRF methods [S1].
- **Memory Complexity:** The memory footprint grows linearly with the number of neural surfels. The number of surfels has sub-linear growth over time due to surfel integration [S1].
- **Performance:** On the ScanNet dataset, SurfelNeRF achieves 23.82 PSNR in feedforward inference and 29.58 PSNR in per-scene optimization settings, with a rendering time of 0.2s per image, compared to 30s for volume rendering [S1].
- **Strengths:** Efficient online reconstruction, fast rendering via rasterization, scalable fusion, and high-quality photorealistic results [S1].
- **Limitations:** Relies on high-quality geometry; inaccurate depth leads to artifacts. Requires depth refinement [S1].

## Background
SurfelNeRF was introduced in 2023 by researchers investigating the combination of classical 3D reconstruction and NeRF for online photorealistic reconstruction of large-scale indoor scenes [S1]. It builds on the concept of neural surfels, extending classical surfel-based fusion schemes to the neural setting. The method was evaluated on the ScanNet dataset, demonstrating state-of-the-art performance in both feedforward and per-scene optimization settings [S1].

## Sources
[S1] SurfelNeRF: Neural Surfel Radiance Fields for Online Photorealistic Reconstruction of Indoor Scenes, https://doi.org/10.1109/cvpr52729.2023.00019, fetched 2026-10-26
