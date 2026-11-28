# Trainable Camera Parameters

Also called: Camera Parameter Optimization, Camera Parameters Joint Learning, Joint Camera-NeRF Optimization, NeRF Initialization with Trainable Camera Parameters

## Overview
This method optimizes camera intrinsic and extrinsic parameters jointly with the neural radiance field (NeRF) during training to handle scenarios where Structure-from-Motion (SfM) pipelines like COLMAP fail, such as small camera motion videos or narrow-baseline structured light fields [S1][S2, unofficial]. Unlike methods that train camera parameters from scratch, this approach initializes residuals from a prior estimate (e.g., RCVD) and optimizes additive or multiplicative corrections [S1]. It builds on [Camera and Pose Optimization](21918-camera-and-pose-optimization.md) [G].

## How it works
**Inputs:**
- Initial camera extrinsics $P_{init}$ and focal lengths $f_{init}$ for each frame, obtained from a prior estimation method (e.g., RCVD [S1] or COLMAP [S2, unofficial]).
- Ground-truth images $I_{gt}$ for each frame [S2, unofficial].
- NeRF model parameters (MLP weights and biases) [S2, unofficial].

**Outputs:**
- Optimized camera extrinsics $P$ and focal lengths $f$ [S1].
- Optimized NeRF model parameters [S2, unofficial].

**Data Structures:**
- For each frame $i$: $P_{init}^{(i)}$ (initial $4 \times 4$ extrinsic matrix), $f_{init}^{(i)}$ (initial scalar focal length), $\Delta P^{(i)}$ (residual extrinsic, same size as $P_{init}^{(i)}$), $\Delta f^{(i)}$ (residual focal length, same size as $f_{init}^{(i)}$), $P^{(i)}$ (final extrinsic), $f^{(i)}$ (final focal length) [S1].

**Numbered Steps:**
1. **Initialization:** Obtain $P_{init}$ and $f_{init}$ for all frames using a prior method (e.g., RCVD [S1] or COLMAP [S2, unofficial]). Initialize residual variables $\Delta P^{(i)}$ and $\Delta f^{(i)}$ to zero for all frames $i$ [S1]. Initialize NeRF model parameters (weights $W$, biases $b$) [S2, unofficial].
2. **Ray Generation:** For each training image $i$, generate rays using the current camera parameters $P^{(i)}$ and $f^{(i)}$ [S1][S2, unofficial]. For structured light fields, rays may be defined using only focal length $f$ and a parallax offset $\Delta$ (proportional to angular distance) [S2, unofficial].
3. **Rendering:** Render the image $I'$ by querying the NeRF model along the generated rays [S2, unofficial].
4. **Loss Computation:** Compute the rendering loss, typically the mean squared error between the rendered image $I'$ and the ground-truth image $I_{gt}$ [S2, unofficial]:
   $$ \mathcal{L} = || I' - I_{gt} ||_2^2 $$
   Additional constraints (e.g., depth smoothing gradient loss) may be added [S1].
5. **Optimization:** Update all trainable parameters jointly using gradient descent: NeRF parameters ($W, b$) and camera residuals ($\Delta P^{(i)}, \Delta f^{(i)}$ for all frames $i$) [S1][S2, unofficial]. Repeat until convergence.

**Formulas:**
- Final focal length:
  $$ f = f_{init} + \Delta f \quad (1) $$
  where $f$ is the final focal length, $f_{init}$ is the initial focal length, and $\Delta f$ is the learned residual [S1].
- Final camera extrinsic:
  $$ P = P_{init} \cdot \Delta P \quad (2) $$
  where $P$ is the final camera extrinsic, $P_{init}$ is the initial extrinsic, and $\Delta P$ is the learned residual [S1].
- Optimization objective (for structured light fields):
  $$ \arg\min_{W, b, f, \Delta} || c' - c_{gt} ||_2^2 \quad (3) $$
  where $c'$ is the rendered pixel value and $c_{gt}$ is the ground-truth pixel value [S2, unofficial].

**Parameters:**
- Learning rate for camera residuals and NeRF parameters: Not given in the sources [S1][S2, unofficial].
- Initialization method for $P_{init}$ and $f_{init}$: RCVD [S1] or COLMAP [S2, unofficial].
- For structured light fields, only focal length $f$ and parallax offset $\Delta$ are required for ray tracing [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Small Camera Motion:** COLMAP may fail to predict camera parameters; this method addresses this by optimizing residuals [S1].
- **Narrow-Baseline Light Fields:** COLMAP may struggle; treating camera pose parameters as trainable variables helps [S2, unofficial].
- **Initialization:** Residuals are initialized to zero, ensuring the initial camera parameters are close to the prior estimate [S1].
- **Depth Noise:** Depth maps may display noise; gradient loss can be added to alleviate this [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Load initial camera parameters from prior estimation (e.g., RCVD or COLMAP)
P_init = load_prior_extrinsics()  # Not given in the sources
f_init = load_prior_focal_lengths()  # Not given in the sources
# Initialize residuals to zero
Delta_P = np.zeros_like(P_init)
Delta_f = np.zeros_like(f_init)
# Initialize NeRF MLP weights and biases
nerf_params = initialize_mlp_weights()  # Not given in the sources
optimizer = SGD(params=[nerf_params, Delta_P, Delta_f], lr=lr)  # Not given in the sources

for epoch in range(num_epochs):  # Not given in the sources
    for image, gt_image in training_data:  # Not given in the sources
        # Compute final camera parameters
        P = P_init @ Delta_P  # Matrix multiplication for extrinsics
        f = f_init + Delta_f  # Element-wise addition for focal lengths

        # Generate rays from camera center using P and f
        rays = generate_rays_from_camera(P, f)  # Not given in the sources

        # Render image by sampling along rays
        rendered_image = volume_render(nerf_params, rays)  # Not given in the sources

        # Compute mean squared error loss
        loss = torch.mean((rendered_image - gt_image) ** 2)  # Not given in the sources

        # Update parameters via gradient descent
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

## Subtopics

### [Joint Optimization of Scene and Calibration Parameters](21724-joint-optimization-of-scene-and-calibration-parameters.md)
Treats camera pose parameters as trainable variables, particularly for narrow-baseline light fields. Choose when dealing with structured light fields where COLMAP struggles.

## Operating characteristics
- **Time Complexity:** Depends on the number of training iterations and the size of the NeRF model [G]. Joint optimization may increase training time compared to fixed-camera NeRF [G].
- **Memory Complexity:** Additional memory for storing residual variables $\Delta P$ and $\Delta f$ for each frame [G].
- **Strengths:** Handles cases where SfM fails (small motion, narrow baseline) [S1][S2, unofficial]. Improves reconstruction quality in challenging scenarios [G].
- **Limitations:** Requires a reasonable initial estimate (e.g., from RCVD or COLMAP) [S1][S2, unofficial]. May be sensitive to initialization if the prior is poor [G].
- **When to Use:** Small camera motion videos, narrow-baseline structured light fields [S1][S2, unofficial].
- **When Not to Use:** Well-calibrated datasets with sufficient camera motion where COLMAP performs reliably [G].

## Background
- **Origin:** Proposed in "Dynamic View Synthesis From Small Camera Motion Videos" [S1] to address COLMAP failures in small motion videos. Also adopted in "Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression" [S2, unofficial] for narrow-baseline light fields.
- **History:** Builds on the idea of treating camera parameters as trainable variables, as seen in earlier works [20] cited in [S2, unofficial].
- **Usage:** Used in dynamic NeRF methods and light field compression pipelines [S1][S2, unofficial].

## Sources
[S1] Dynamic View Synthesis From Small Camera Motion Videos, https://doi.org/10.1109/tvcg.2025.3565642, fetched 2026-10-27
[S2] Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression, https://core.ac.uk/works/391353499, fetched 2026-10-30 (unofficial)
