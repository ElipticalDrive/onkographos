# BARF (Bundle-Adjusting Neural Radiance Fields)
Also called: Bundle-Adjusting NeRF

## Overview
BARF reconstructs a Neural Radiance Field (NeRF) from multi-view images when the input camera poses contain errors [S1, unofficial]. It addresses the sensitivity of standard NeRF to inaccurate pose initialization by employing a coarse-to-fine refinement strategy that gradually increases the high-frequency components of the positional encoding [S1, unofficial]. This approach stabilizes the joint optimization of network weights and camera parameters, preventing convergence to local minima caused by high-frequency details that cannot be resolved with poor initial poses [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N$ images $\{I_i\}_{i=1}^N$ and their corresponding initial camera poses $\{\pi_i\}_{i=1}^N$, where $\pi_i$ includes rotation $R_i$, translation $t_i$, and potentially focal length $f$. The initial poses may contain errors.
- **Outputs:** Optimized NeRF volume representation (MLP weights $\Theta$) and refined camera poses $\{\hat{\pi}_i\}_{i=1}^N$.

**Data Structures:**
- **Positional Encoding (PE):** Standard NeRF PE $\gamma(p)$ maps a 3D coordinate $p$ to a vector of sinusoidal functions. BARF modifies this to control frequency progression.
- **Frequency Schedule:** A scalar parameter $\lambda(t)$ or a step-wise schedule controlling the maximum frequency of the PE used during training iteration $t$.

**Method Steps:**
1.  **Initialization:** Initialize the NeRF MLP weights $\Theta$ and camera poses $\pi_i$ with the provided (potentially erroneous) estimates.
2.  **Frequency Scheduling:** Define a coarse-to-fine schedule for the positional encoding. Initially, only low-frequency basis functions are active. As training progresses, higher-frequency basis functions are gradually introduced.
3.  **Ray Generation:** For each pixel $p$ in image $I_i$, generate the ray origin $\hat{o}_i$ and direction $\hat{d}_{i,p}$ using the current pose estimates $\pi_i$ [7405-nerf-with-unknown-camera-parameters.md].
4.  **Volume Rendering:** Sample points along the ray. Query the NeRF for density $\sigma$ and color $c$. Compute the rendered color $C_i(p)$ using the standard volume rendering integral [7405-nerf-with-unknown-camera-parameters.md].
5.  **Loss Computation:** Calculate the photometric loss $\mathcal{L}_{photo} = \| \hat{C}_i(p) - C_i(p) \|^2$.
6.  **Backpropagation:** Backpropagate the loss through the rendering equation and ray generation to compute gradients for both the NeRF weights $\Theta$ and the camera parameters $\pi_i$.
7.  **Optimization:** Update $\Theta$ and $\pi_i$ using an optimizer (e.g., Adam).
8.  **Frequency Update:** Increment the training step and update the positional encoding frequency schedule to include higher frequencies.
9.  **Termination:** Repeat steps 3-8 until convergence or a maximum number of iterations is reached.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barf_training(images, initial_poses, max_iterations, frequency_schedule):
    """
    BARF training loop.
    
    Args:
        images: List of input images.
        initial_poses: List of initial camera poses (may contain errors).
        max_iterations: Total number of training steps.
        frequency_schedule: Function or list defining the max frequency index 
                            for positional encoding at each iteration.
    
    Returns:
        optimized_nerf: Trained NeRF model.
        refined_poses: Optimized camera poses.
    """
    # Initialize NeRF model with random weights
    nerf = initialize_nerf()
    
    # Initialize camera poses with provided initial estimates
    poses = copy(initial_poses)
    
    for t in range(max_iterations):
        # Determine current max frequency for positional encoding
        max_freq_idx = frequency_schedule(t)
        
        # Sample rays from all images
        rays = []
        for i, image in enumerate(images):
            # Generate rays using current pose estimates
            # Ray origin o_i and direction d_i,p derived from poses[i]
            # See [7405-nerf-with-unknown-camera-parameters.md] for ray generation details
            rays.extend(sample_rays(image, poses[i]))
        
        # Forward pass: Render images
        rendered_colors = []
        for ray in rays:
            # Sample points along ray
            points = sample_points(ray, nerf)
            
            # Query NeRF for density and color
            # Use positional encoding with limited frequency
            encoded_points = positional_encoding(points, max_freq_idx)
            sigma, color = nerf(encoded_points)
            
            # Volume rendering
            rendered_color = volume_render(ray, sigma, color)
            rendered_colors.append(rendered_color)
        
        # Compute photometric loss
        # Match rendered colors to ground truth image pixels
        loss = compute_photometric_loss(rendered_colors, images)
        
        # Backpropagation
        # Gradients flow to both nerf weights and camera poses
        gradients = compute_gradients(loss, nerf, poses)
        
        # Update parameters
        nerf.update(gradients['nerf'])
        poses.update(gradients['poses'])
        
    return nerf, poses
```

**Parameters:**
- **Frequency Schedule:** The rate at which high frequencies are introduced. A common approach is linear or exponential increase. Not given in the sources.
- **Learning Rate:** Standard Adam learning rate. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Poor Initialization:** If initial poses are too far off, even coarse-to-fine may fail to converge. [G] Ensure initial poses are roughly aligned (e.g., via structure-from-motion).
- **High-Frequency Noise:** Introducing high frequencies too early can lead to overfitting to noise or local minima. [S1, unofficial]

## Subtopics

### Iterative Alignment Strategy
An iterative process to generate better-aligned training and testing data by using optical flow to warp ground truth images to match the geometry estimated by NeRF, addressing misalignment from camera pose errors and non-rigid motion [S2]. It is used for analysis and data generation to demonstrate the impact of misalignment, not as the core BARF algorithm [S2].
- **Steps:**
    1. Train NeRF (e.g., mip-NeRF 360) using current ground truth images [S2].
    2. Calculate optical flow from rendered views to ground truth views using PWC-Net [S2].
    3. Warp the ground truth images using the estimated optical flows to build a new, better-aligned training set [S2].
    4. Repeat steps 1-3 for 2-3 iterations to reach best quality, leveraging sharper images to improve flow accuracy in subsequent rounds [S2].
- **Pitfalls:** PWC-Net may run out of memory on high-resolution images; RAFT was considered but rejected for the same reason [S2]. NeRF rendered images are often blurry due to misalignment, violating the general optical flow assumption of sharp paired inputs, though PWC-Net produces reasonable results based on global structures [S2].
- **Usage:** Used to generate a better-aligned test set to avoid biased conclusions from misaligned data [S2].

## Operating characteristics
- **Time Complexity:** Training time is comparable to standard NeRF but may require more iterations due to the joint optimization of poses and network weights. The iterative alignment strategy adds significant overhead due to repeated training and optical flow computation [S2].
- **Memory Complexity:** Similar to standard NeRF. The iterative alignment strategy requires storing multiple versions of warped images and intermediate NeRF models if not careful, but typically only the latest model is used [S2].
- **Performance:** BARF significantly improves reconstruction quality when initial poses are inaccurate [S1, unofficial]. The iterative alignment strategy can further improve PSNR (e.g., +0.62dB for large models) by correcting misalignment [S2].
- **Strengths:** Robust to initial pose errors; does not require precise initial poses [S1, unofficial].
- **Limitations:** Sensitive to the frequency schedule; may converge to local minima if initialization is extremely poor [S1, unofficial]. The iterative alignment strategy is computationally expensive and memory-intensive [S2].
- **When to use:** When camera poses are estimated from SfM or other methods with known errors.
- **When not to use:** When precise ground truth poses are available; when computational resources are limited for iterative alignment.

## Background
BARF (Bundle-Adjusting Neural Radiance Fields) was introduced in ICCV 2021 [S1, unofficial]. It addresses the lack of robustness in NeRF when faced with inaccurate camera poses, a common issue in real-world scenarios where poses are estimated from SfM [S1, unofficial]. The method builds on the principle of differentiable rendering for pose optimization, as shared by its siblings [7405-nerf-with-unknown-camera-parameters.md].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
