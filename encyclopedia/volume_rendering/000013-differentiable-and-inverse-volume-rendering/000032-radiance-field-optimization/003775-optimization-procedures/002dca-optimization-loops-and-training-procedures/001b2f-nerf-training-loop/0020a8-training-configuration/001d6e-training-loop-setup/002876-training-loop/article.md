# Training Loop

Also called: NeRF Training Pipeline, Training Loop & Data Fetching

## Overview
The Training Loop is the iterative optimization procedure that updates a radiance field representation (such as a NeRF or 3D Gaussian Splatting model) by minimizing the difference between rendered images and ground-truth observations [S1, unofficial][S2, unofficial]. It differs from its sibling [Training Ray Batching](2252-training-ray-batching.md) by managing the high-level control flow, data loading, and optimizer state, whereas batching focuses specifically on the memory-efficient assembly of ray samples [S2, unofficial].

## How it works
The loop executes a sequence of data fetching, rendering, loss computation, and parameter updates for a specified number of steps or until convergence.

### Inputs and Outputs
**Inputs:**
*   Training dataset (images, camera poses, intrinsics, and optional masks) [S2, unofficial].
*   Model configuration (network architecture, sampling strategy) [S2, unofficial].
*   Optimization hyperparameters (learning rate, batch size `N_rand`, optimizer type) [S2, unofficial][S4, unofficial].
*   Rendering parameters (near/far clipping planes, background color) [S2, unofficial][S1, unofficial].

**Outputs:**
*   Updated model parameters (weights of the neural network or Gaussian attributes) [S2, unofficial].
*   Optimizer state (momentum, variance estimates) [S3, unofficial].

### Data Structures
The training state typically includes:
*   **Optimizer State:** Arrays for momentum ($m$) and variance ($v$) estimates for each parameter, updated per step [S3, unofficial].
*   **Render Kwargs:** A dictionary or struct containing `near`, `far`, `N_rand`, and model references [S2, unofficial].
*   **Ray Batch:** A flat array or tensor containing ray origins, directions, and corresponding ground-truth RGB values, reshaped for parallel processing [S2, unofficial].

### Numbered Steps
1.  **Initialize:** Load dataset-specific data (images, poses, bounds) and split into train/val/test sets [S2, unofficial]. Create the model and optimizer [S2, unofficial]. Set near/far clipping planes based on dataset type (e.g., LLFF: $near = \min(bounds) \times 0.9$, $far = \max(bounds) \times 1.0$; Blender: $near=2.0, far=6.0$) [S2, unofficial].
2.  **Fetch Batch:** Retrieve the next batch of rays from the training iterator. If the iterator is exhausted, reset it to `iter(train_loader)` and fetch again [S1, unofficial].
3.  **Device Transfer:** Move ground-truth images, camera parameters, and masks to the CUDA device [S1, unofficial].
4.  **Render:** Render the scene using the current model parameters and the ray batch. For Gaussian Splatting, use `bg_colour=(0.0, 0.0, 0.0)` and specified image size [S1, unofficial]. For NeRF, use the accumulated color formula:
    $$ \tilde{\mathbf{C}} = \sum_{i=1}^{N} \left(\alpha_i \boldsymbol{c}_i \prod_{j=1}^{i-1} \left(1 - \alpha_j\right)\right) $$
    where $\alpha_i$ is the transmittance-weighted opacity and $\boldsymbol{c}_i$ is the color at sample $i$ [S2, unofficial].
5.  **Compute Loss:** Calculate the loss (e.g., Mean Squared Error) between the rendered image and the ground-truth image [S1, unofficial].
6.  **Backpropagate:** Call `loss.backward()` to compute gradients [S1, unofficial].
7.  **Update:** Call `optimizer.step()` to update parameters and `optimizer.zero_grad()` to clear gradients [S1, unofficial].
8.  **Iterate:** Increment the global step counter and repeat from Step 2.

### Formulas
**Adam Optimizer Update [S3, unofficial]:**
For each parameter $p$ at step $t$:
$$ m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t $$
$$ v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2 $$
$$ \hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} $$
$$ p_t = p_{t-1} - \eta \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} $$
where $\beta_1=0.9$, $\beta_2=0.999$, $\epsilon=10^{-6}$, and $\eta$ is the learning rate [S3, unofficial].

**NeRF Volume Rendering [S2, unofficial]:**
$$ \tilde{\mathbf{C}}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \boldsymbol{c}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
where $\sigma_j$ is density, $\delta_j$ is distance, and $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S2, unofficial].

### Parameters
*   `N_rand`: Number of random rays sampled per batch (e.g., 1024, 2048) [S2, unofficial].
*   `use_batching`: Boolean flag enabling/disabling ray batching [S2, unofficial].
*   `valCamerasCount`: Number of validation cameras (default 100) [S3, unofficial].
*   `testCamerasCount`: Number of test cameras (default 200) [S3, unofficial].
*   `grayLized`: Grayscale rendering mode flag (default true) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Iterator Exhaustion:** The loop must handle `StopIteration` by resetting the data loader to avoid crashing [S1, unofficial].
*   **Trivial Solutions in Importance Prediction:** If the importance predictor's input features are not detached, gradients may push the proposer to generate bad samples and predict them as unimportant to minimize loss. A 'stop gradient' operation is required at the input of the importance predictor [S4, unofficial].
*   **Dataset-Specific Bounds:** Incorrect near/far plane settings (e.g., using NDC bounds for non-NDC datasets) cause rendering artifacts or empty volumes [S2, unofficial].
*   **Precision:** Floating-point precision in transmittance calculation can lead to underflow/overflow for deep scenes; careful normalization or log-space computation may be needed [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def training_loop(dataset_config, model_config, optimizer_config, num_steps):
    # 1. Initialize
    dataset = load_dataset(dataset_config) # Returns images, poses, bounds, etc.
    near, far = compute_bounds(dataset, dataset_config.dataset_type)
    model = create_model(model_config)
    optimizer = create_optimizer(model.parameters(), optimizer_config)
    
    # 2. Main Loop
    train_iterator = iter(dataset.train_loader)
    
    for step in range(num_steps):
        try:
            # Fetch batch
            ray_batch = next(train_iterator)
        except StopIteration:
            train_iterator = iter(dataset.train_loader)
            ray_batch = next(train_iterator)
            
        # Move to device
        rays = ray_batch.rays.to(device)
        gt_colors = ray_batch.colors.to(device)
        
        # Render
        # For NeRF:
        # colors = render_nerf(model, rays, near, far)
        # For 3DGS:
        # colors = render_gaussian_splatting(model, rays, bg_colour=(0.0, 0.0, 0.0))
        
        # Compute Loss
        loss = mse_loss(colors, gt_colors)
        
        # Backpropagate and Update
        optimizer.zero_grad()
        loss.backward()
        
        # Handle importance predictor stop gradient if applicable
        if model.has_importance_predictor:
            detach_importance_predictor_inputs(model)
            
        optimizer.step()
        
        # Log metrics
        if step % log_interval == 0:
            log_loss(loss.item())
```

## Subtopics

### [VCCNF Training Loop](4736-vccnf-training-loop.md)
Supervised optimization for VCCNF using MSE on latent codes, differing from diffusion-based siblings by direct regression.

### [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md)
Uses multi-view diffusion priors for gradient guidance, ensuring multi-view consistency unlike single-view methods.

### [Diffusion-Based Guidance for Radiance Field Optimization](8117-diffusion-based-guidance-for-radiance-field-optimization.md)
Uses single-view diffusion priors with classifier-free guidance, differing from MVDream by not enforcing multi-view consistency explicitly.

### [Two-Stage Training for Learnable Proposer](8685-two-stage-training-for-learnable-proposer.md)
Decouples proposer initialization from fine-tuning to avoid chicken-and-egg problems in sampling optimization.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot B \cdot S)$ per step, where $N$ is the number of rays, $B$ is the batch size, and $S$ is the number of samples per ray [S2, unofficial].
*   **Memory Complexity:** $O(N \cdot B \cdot S)$ for storing ray gradients and intermediate rendering buffers [S2, unofficial].
*   **Performance:** Depends heavily on hardware (GPU memory bandwidth) and resolution. Higher `N_rand` improves quality but increases memory usage [S2, unofficial].
*   **Strengths:** Flexible, supports various radiance field representations (NeRF, 3DGS) [S1, unofficial][S2, unofficial].
*   **Limitations:** Slow convergence for complex scenes; sensitive to hyperparameters like learning rate and batch size [S4, unofficial].
*   **When to Use:** Standard training for differentiable volume rendering.
*   **When Not to Use:** When real-time inference is required without pre-training, or when using non-differentiable representations.

## Background
The training loop concept originates from the NeRF paper (Mildenhall et al., 2020) [S2, unofficial]. It has been adapted for 3D Gaussian Splatting (Kerbl et al., 2023) [S1, unofficial] and other radiance field variants. The use of Adam optimizer and random ray sampling is standard in this domain [S2, unofficial][S3, unofficial].

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S3] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S4] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
