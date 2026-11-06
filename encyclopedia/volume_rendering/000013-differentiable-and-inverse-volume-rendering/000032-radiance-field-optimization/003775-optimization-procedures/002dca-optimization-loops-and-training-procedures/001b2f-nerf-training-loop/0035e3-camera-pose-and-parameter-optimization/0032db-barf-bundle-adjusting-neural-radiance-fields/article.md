# BARF (Bundle-Adjusting Neural Radiance Fields)
Also called: Bundle-Adjusting NeRF
## Overview
BARF reconstructs a Neural Radiance Field (NeRF) when the input camera poses contain errors and the corresponding multi-view images are provided [S1, unofficial]. It addresses NeRF's sensitivity to inaccurate camera poses by using a coarse-to-fine refinement strategy that gradually increases the high-frequency components of the positional encoding, allowing the model to first optimize low-frequency geometry and pose before refining high-frequency details [S1, unofficial]. This sets it apart from standard NeRF training, which assumes precise camera poses and typically uses full-frequency positional encoding from the start.
## How it works
**Inputs and Outputs:**
- Inputs: Multi-view images with erroneous camera poses (extrinsics and intrinsics).
- Outputs: A trained NeRF model (MLPs for density and color) and refined camera poses.

**Data Structures:**
- Positional Encoding (PE): Standard NeRF PE function $\gamma(p)$ mapping a 3D point $p$ to a vector of sines and cosines at multiple frequencies.
- NeRF MLP: Takes concatenated position $p$, view direction $v$, and PE of $p$ to output density $\sigma$ and color $c$.

**Steps:**
1. **Initialization:** Initialize the NeRF MLP and the erroneous camera poses.
2. **Coarse-to-Fine Schedule:** Define a schedule for the maximum frequency $\omega_{max}$ used in the positional encoding. Start with a low $\omega_{max}$ (e.g., only the base frequency or low harmonics) and gradually increase it over the course of training.
3. **Training Loop:**
   a. Sample rays from the input images using the current (erroneous) camera poses.
   b. Encode the 3D points along the rays using the positional encoding with the current $\omega_{max}$.
   c. Query the NeRF MLP to get density and color.
   d. Perform volume rendering to predict pixel colors.
   e. Compute the loss between predicted and ground truth pixel colors.
   f. Update the NeRF MLP parameters and the camera pose parameters using gradient descent.
4. **Frequency Increase:** At predefined intervals or based on convergence, increase $\omega_{max}$ to include higher frequencies. This allows the model to fit finer details only after the coarse geometry and pose have been roughly aligned.
5. **Final Refinement:** Once $\omega_{max}$ reaches the full set of frequencies, continue training to refine high-frequency details.

**Parameters:**
- Initial $\omega_{max}$: Not given in the sources.
- Final $\omega_{max}$: Typically the full set of frequencies used in standard NeRF.
- Schedule: Gradual increase over training iterations.

**Edge Cases and Pitfalls:**
- If the initial pose error is too large, the coarse-to-fine strategy may fail to converge.
- Precision issues may arise if the frequency increase is too abrupt.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize NeRF MLP and camera poses
nerf = initialize_nerf()
poses = initialize_erroneous_poses()

# Define PE schedule: list of max frequencies to use at different stages
pe_schedule = [low_freq, medium_freq, high_freq]

for stage, max_freq in enumerate(pe_schedule):
    # Train for a number of iterations with current max_freq
    for iteration in range(num_iterations_per_stage):
        # Sample rays using current poses
        rays = sample_rays(poses, images)
        
        # Encode 3D points with PE up to max_freq
        encoded_points = positional_encoding(rays.points, max_freq=max_freq)
        
        # Query NeRF
        density, color = nerf(encoded_points, rays.directions)
        
        # Render
        rendered_colors = volume_rendering(density, color, rays)
        
        # Compute loss
        loss = compute_loss(rendered_colors, images)
        
        # Update parameters
        update(nerf, poses, loss)
        
    # Increase max_freq for next stage
```
## Subtopics

### Iterative Alignment Strategy
An iterative process to generate better-aligned training and testing data by using optical flow to warp ground truth images to match the geometry estimated by NeRF, addressing misalignment from camera pose errors and non-rigid motion [S2]. It involves: 1) Training NeRF (e.g., mip-NeRF 360) with current data; 2) Calculating optical flow from rendered views to ground truth views using PWC-Net; 3) Warping ground truth images using the estimated optical flows to build a new, better-aligned training set; 4) Repeating steps 1-3 for 2-3 iterations to reach best quality, leveraging sharper images to improve flow accuracy in subsequent rounds [S2]. Pitfalls: PWC-Net may run out of memory on high-resolution images; RAFT was rejected for the same reason [S2]. NeRF rendered images are often blurry due to misalignment, violating the general optical flow assumption of sharp paired inputs, but PWC-Net produces reasonable results based on global structures [S2]. Usage: Used for analysis and data generation in experiments; does not represent the proposed algorithm in the AligNeRF paper but demonstrates the impact of misalignment [S2].

## Operating characteristics
Time and memory complexity: Not given in the sources. Measured performance: Not given in the sources. Scaling: Not given in the sources. Strengths and limitations: Improves reconstruction quality when camera poses are inaccurate [S1, unofficial]. When to use it: When camera poses are estimated with errors. When not: When precise camera poses are available.
## Background
Origin: BARF (Bundle-Adjusting Neural Radiance Fields) was introduced in ICCV 2021 [S1, unofficial]. It addresses the problem of NeRF's poor robustness to inaccurate camera poses [S1, unofficial].
## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
