# NeRF-Casting Optimization and Representation

Also called: NeRF-Casting, Improved View-Dependent Appearance with Consistent Reflections

## Overview
NeRF-Casting is a differentiable volume rendering method that extends NeRF with explicit reflection modeling to handle glossy surfaces and consistent view-dependent appearance [S1]. It solves the problem of inaccurate specular highlights and inconsistent reflections in standard neural radiance fields by introducing a reflection-dependent color MLP and a modified proposal sampling scheme [S1]. It builds on [Mip-NeRF 360 Proposal Sampling](5408-mip-nerf-360-proposal-sampling.md) and [Zip-NeRF](https://arxiv.org/abs/2302.11423) by adapting their adaptive sampling and hash grid structures to support reflected rays [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera poses and intrinsic parameters.
*   Training images (resolution $3504 \times 2336$, downsampled by 4).
*   Masked images (background removal).

**Outputs:**
*   Rendered RGB images for given camera poses.
*   Predicted normals $\tilde{\mathbf{n}}$.
*   Predicted roughness $\rho$.
*   Density field $\tau$.

### Data Structures

**Hash Grids (NGP):**
The method uses separate hash grids for different feature types.
*   **Appearance Model Grid:** Resolutions $2^0, \dots, 2^9$ with 2 dimensions each.
*   **Final Sampling Grids:** Three separate NGP features queried in the final round:
    1.  **Roughness:** 1 dimension.
    2.  **Normals:** 3 dimensions.
    3.  **Density/Bottleneck/Mixing:** 4 dimensions.
*   **Proposal Grids:** Used for adaptive sampling. Round $i$ uses a hash grid with resolutions $2^{a_1}, \dots, 2^{a_i}$, where $a_1=9, a_2=11, a_3=13$. Each grid has a maximum of $2^{21}$ elements [S1].

**MLPs:**
*   **View-dependent Color MLP ($M_c$):** Two-layer MLP, hidden dimension 128, output logistic sigmoid.
*   **Reflection-dependent Color MLP ($M_r$):** Two-layer MLP, hidden dimension 128, output logistic sigmoid.
*   **Feature Decoders:** Two-layer MLPs, hidden dimension 64, for roughness, normals, density, and mixing coefficients [S1].

### Numbered Steps

1.  **Initialization:** Initialize hash grids for proposal sampling and appearance modeling. Initialize MLP weights. Set optimization progress $t=0$.
2.  **Ray Casting:** For each training pixel, cast a primary camera ray. Also, compute reflected rays based on predicted normals and view directions.
3.  **Adaptive Proposal Sampling:**
    *   Perform 3 rounds of proposal sampling [S1].
    *   **Round 1:** Resolution $2^9$, 64 samples for camera rays, 64 for reflected rays.
    *   **Round 2:** Resolution $2^{11}$, 64 samples for camera rays, 64 for reflected rays.
    *   **Round 3:** Resolution $2^{13}$, 32 samples for camera rays. No reflected rays in this round.
    *   Use the proposal distribution to sample intervals along the ray [S1].
4.  **Feature Querying:**
    *   For each sample point $\mathbf{x}$, query the hash grids.
    *   Apply coarse-to-fine windowing to the features $\mathbf{f}$ using the windowing function $\mathbf{w}(\mathbf{v}, t)$ [S1]:
        $$ \mathbf{f}' = \mathbf{f} \odot \mathbf{w}(\mathbf{v}, t) $$
        where $\mathbf{v}$ is the resolution vector, $t \in [0,1]$ is optimization progress, and:
        $$ \mathbf{w}(\mathbf{v}, t) = \frac{1 - \cos(\pi \xi(\mathbf{v}, t))}{2} $$
        $$ \xi(\mathbf{v}, t) = \text{clip}(\log_2(m) - \log_2(\mathbf{v}) - 1 + st) $$
        with $m = 16\sqrt{2}$ and $s = 25$ [S1].
5.  **Feature Decoding:**
    *   Decode raw NGP features into physical properties:
        *   **Roughness:** $\rho = \text{softplus}(\text{MLP}(\tilde{\mathbf{f}}_\rho) - 1)$ [S1].
        *   **Normals:** $\tilde{\mathbf{n}} = \text{normalize}(\text{MLP}(\tilde{\mathbf{f}}_n))$ [S1].
        *   **Density:** $\tau = \exp(\text{MLP}(\hat{\mathbf{f}}_b) + 2)$ [S1].
        *   **Mixing Coefficients:** $\beta = \text{sigmoid}(\text{MLP}(\tilde{\mathbf{f}}_b))$ [S1].
6.  **Color Prediction:**
    *   **View-dependent Color:** Input to $M_c$: position $\mathbf{x}$, bottleneck vector $\mathbf{b}$, normal $\mathbf{n}$, view direction $\mathbf{d}$, and camera position $\mathbf{o}$ encoded as $(\cos(\mathbf{o}), \sin(\mathbf{o}), \cos(2\mathbf{o}), \sin(2\mathbf{o}))$ [S1].
    *   **Reflection-dependent Color:** Input to $M_r$: position $\mathbf{x}$, bottleneck $\mathbf{b}$, normal $\mathbf{n}$, dot product $\mathbf{n} \cdot \mathbf{d}$, reflected ray direction $\mathbf{d}'$, feature vector $\bar{\mathbf{f}}$, and camera position encoding [S1].
7.  **Volume Rendering:**
    *   Accumulate color and density along the ray using standard volume rendering equations.
    *   Apply stop-gradient operators when accumulating normal vectors and intersection points to prevent gradients from affecting weights [S1].
8.  **Loss Computation:**
    *   **Reconstruction Loss:** MSE between rendered and ground truth images.
    *   **Proposal Supervision:** Modify Zip-NeRF's spline-based loss. Blur each NeRF interval by a rectangular pulse whose radius is the weighted geometric mean of the proposal distribution intervals overlapping with the NeRF interval. Compute by taking log-widths of proposal intervals, interpolating into a step function, averaging log-widths, and exponentiating [S1].
    *   **Distortion Loss:** Apply distortion loss from mip-NeRF 360 to both camera rays and reflected rays to prevent 'floater' artifacts [S1].
    *   **Orientation Loss:** Apply orientation loss from Ref-NeRF to normal vectors with weight $10^{-3}$ [S1].
9.  **Optimization:**
    *   Update parameters using Adam optimizer [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def softplus(x):
    return np.log1p(np.exp(x))

def normalize(x):
    return x / np.linalg.norm(x)

def clip(x, min_val=0.0, max_val=1.0):
    return np.clip(x, min_val, max_val)

def windowing_function(v, t, m=16*np.sqrt(2), s=25):
    """
    v: resolution vector (scalar or array)
    t: optimization progress [0, 1]
    """
    xi = clip(np.log2(m) - np.log2(v) - 1 + s * t)
    return (1 - np.cos(np.pi * xi)) / 2

def decode_roughness(raw_f_rho):
    # Two-layer MLP with hidden dim 64, softplus, bias -1
    # Simplified as a linear projection for pseudocode clarity
    h = np.tanh(np.dot(raw_f_rho, W1_rho) + b1_rho)
    out = np.dot(h, W2_rho) + b2_rho
    return softplus(out - 1)

def decode_normals(raw_f_n):
    # Two-layer MLP with hidden dim 64, normalize
    h = np.tanh(np.dot(raw_f_n, W1_n) + b1_n)
    out = np.dot(h, W2_n) + b2_n
    return normalize(out)

def decode_density(raw_f_b):
    # Two-layer MLP with hidden dim 64, exp, bias 2
    h = np.tanh(np.dot(raw_f_b, W1_b) + b1_b)
    out = np.dot(h, W2_b) + b2_b
    return np.exp(out + 2)

def decode_mixing(raw_f_b):
    # Two-layer MLP with hidden dim 64, sigmoid
    h = np.tanh(np.dot(raw_f_b, W1_mix) + b1_mix)
    out = np.dot(h, W2_mix) + b2_mix
    return 1 / (1 + np.exp(-out))

def proposal_supervision_loss(neural_intervals, proposal_intervals):
    """
    Compute modified Zip-NeRF proposal supervision loss.
    Blur each NeRF interval by a rectangular pulse whose radius is the 
    weighted geometric mean of the proposal distribution intervals overlapping 
    with the NeRF interval.
    """
    # 1. Identify overlapping proposal intervals for each NeRF interval
    # 2. For each NeRF interval, collect log-widths of overlapping proposal intervals
    # 3. Interpolate these log-widths into a step function
    # 4. Average log-widths for each NeRF interval
    # 5. Exponentiate to get geometric mean radius
    # 6. Compute loss between blurred NeRF distribution and proposal distribution
    # Implementation details omitted as per source description complexity
    pass

def train_step(batch, params):
    # 1. Cast rays
    camera_rays = batch.camera_rays
    reflected_rays = compute_reflected_rays(batch, params)
    
    # 2. Proposal Sampling (3 rounds)
    # Round 1: Res 2^9, 64 samples
    # Round 2: Res 2^11, 64 samples
    # Round 3: Res 2^13, 32 samples (camera only)
    samples = adaptive_sample(camera_rays, reflected_rays, params.proposal_grids)
    
    # 3. Feature Query and Decoding
    features = query_hash_grids(samples.positions, params.appearance_grids)
    
    # Apply windowing
    windowed_features = features * windowing_function(params.grid_resolutions, params.t)
    
    rho = decode_roughness(windowed_features['rho'])
    normals = decode_normals(windowed_features['normals'])
    density = decode_density(windowed_features['density'])
    mixing = decode_mixing(windowed_features['mixing'])
    
    # 4. Color Prediction
    # View-dependent
    view_input = [samples.positions, samples.bottleneck, normals, samples.view_dirs, 
                  encode_camera_pos(batch.camera_pos)]
    color_view = M_c(view_input)
    
    # Reflection-dependent
    refl_input = [samples.positions, samples.bottleneck, normals, 
                  np.dot(normals, samples.view_dirs), samples.reflected_dirs, 
                  samples.features, encode_camera_pos(batch.camera_pos)]
    color_refl = M_r(refl_input)
    
    # 5. Volume Rendering
    # Apply stop-gradient to normals and intersection points during accumulation
    rendered_color = volume_render(samples, density, color_view, color_refl, mixing)
    
    # 6. Loss Computation
    loss_recon = mse(rendered_color, batch.gt_images)
    loss_proposal = proposal_supervision_loss(samples.neural_intervals, samples.proposal_intervals)
    loss_distortion = distortion_loss(density, samples) # Applied to both camera and reflected rays
    loss_orientation = orientation_loss(normals) # Weight 1e-3
    
    total_loss = loss_recon + loss_proposal + loss_distortion + loss_orientation
    
    return total_loss

def encode_camera_pos(camera_pos):
    """Low-degree positional encoding of degree 2"""
    return np.concatenate([
        np.cos(camera_pos),
        np.sin(camera_pos),
        np.cos(2 * camera_pos),
        np.sin(2 * camera_pos)
    ])
```

### Parameters and Typical Values
*   **Proposal Sampling Rounds:** 3 rounds [S1].
*   **Resolutions:** Round 1: $2^9$, Round 2: $2^{11}$, Round 3: $2^{13}$ [S1].
*   **Max Grid Elements:** $2^{21}$ per hash grid [S1].
*   **Samples per Ray:** Camera rays: 64 (R1), 64 (R2), 32 (R3). Reflected rays: 64 (R1), 64 (R2) [S1].
*   **MLP Hidden Dimension:** 128 for color MLPs, 64 for feature decoders [S1].
*   **Windowing Hyperparameters:** $s = 16\sqrt{2}$, $t = 25$ (initial scale) [S1].
*   **Batch Size:** $2^{15}$ (halved from $2^{16}$ due to memory) [S1].
*   **Iterations:** 50k total [S1].
*   **Learning Rate:** Logarithmic decay from $10^{-2}$ to $10^{-3}$ for first 25k iterations (after 5k cosine warmup), then constant $10^{-3}$ for next 25k [S1].
*   **Optimizer:** Adam ($\beta_1=0.9, \beta_2=0.99, \epsilon=10^{-15}$) [S1].

### Edge Cases and Pitfalls
*   **Memory Consumption:** The inclusion of reflected rays increases peak memory consumption, necessitating a halved batch size [S1].
*   **Floater Artifacts:** Without distortion loss on reflected rays, unobserved regions may develop floating artifacts [S1].
*   **Gradient Flow:** Stop-gradient operators are required when accumulating normals and intersection points to prevent gradients from affecting weights [S1].
*   **Manual Hyperparameter Tuning:** The modified proposal supervision avoids the need for manual blur radius $\alpha$ tuning, which is challenging for variable-length reflected ray intervals [S1].

## Operating characteristics

*   **Time Complexity:** Dependent on the number of proposal sampling rounds and samples per ray. 3 rounds with up to 64 samples per ray. Total iterations: 50k [S1].
*   **Memory Complexity:** High due to multiple hash grids and reflected ray processing. Batch size reduced to $2^{15}$ to manage memory [S1].
*   **Performance:** Uses 3504x2336 images downsampled by 4. Trained on 4 new scenes with 231-348 images each [S1].
*   **Strengths:** Handles glossy surfaces and consistent reflections. Adaptive proposal supervision reduces aliasing.
*   **Limitations:** Increased memory usage. Requires careful tuning of windowing parameters.

## Background

*   **Origin:** NeRF-Casting was introduced in "NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections" [S1].
*   **History:** Builds on Zip-NeRF [S1], mip-NeRF 360 [S1], and Ref-NeRF [S1].
*   **Usage:** Used for novel view synthesis of scenes with glossy objects and reflections.

## Sources

[S1] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
