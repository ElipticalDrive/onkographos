# Gaussian Splatting Training with SDS Loss

Also called: SDS-based Gaussian Splatting Training, Score Distillation Sampling for 3D Gaussian Splatting

## Overview
This method optimizes 3D Gaussian primitives using a Score Distillation Sampling (SDS) loss derived from a pretrained 2D diffusion model, enabling reconstruction from unstructured image collections or text prompts where ground-truth 3D geometry is unavailable. Unlike sibling methods that rely on photometric consistency across multiple views [S1, unofficial], this approach leverages the semantic prior of a diffusion model to guide the Gaussian parameters, making it suitable for single-view or sparse-view synthesis.

## How it works
**Inputs:**
- A set of 3D Gaussian primitives $\mathcal{G}$ initialized from a point cloud (isotropic covariance).
- A pretrained diffusion model (e.g., Stable Diffusion 2.1) comprising a VAE, text encoder, UNet, and DDIM scheduler.
- A text prompt $T$ describing the target scene.
- Camera poses and image dimensions from the dataset.

**Outputs:**
- Optimized Gaussian parameters: means $\mathbf{p}_i$, pre-activation scales $\log \mathbf{s}_i$, pre-activation opacities $\log \alpha_i$, and colors $\mathbf{c}_i$.

**Data Structures:**
- **Gaussians:** Each primitive $i$ stores:
  - $\mathbf{p}_i \in \mathbb{R}^3$: Mean position.
  - $\mathbf{q}_i \in \mathbb{R}^4$: Rotation quaternion (normalized).
  - $\mathbf{s}_i \in \mathbb{R}^3$: Scale vector (stored as $\log \mathbf{s}_i$ for optimization).
  - $\alpha_i \in [0, 1]$: Opacity (stored as $\log \alpha_i$ for optimization).
  - $\mathbf{c}_i$: Color coefficients (SH or RGB).
- **Scene:** Manages the list of Gaussians and handles rendering calls.

**Parameters:**
- Learning rates: $\eta_{\text{opacity}} = 0.01$, $\eta_{\text{scale}} = 0.01$, $\eta_{\text{color}} = 0.01$, $\eta_{\text{mean}} = 0.001$ [S1, unofficial].
- Optimizer: Adam with $\epsilon = 10^{-15}$ [S1, unofficial].
- Training iterations: 1,000 [S1, unofficial].
- SDS Guidance Scale ($s$): 100 [S1, unofficial].
- SDS Inference Steps: 50 [S1, unofficial].
- Timestep Range: $t_{\min} = 0.02 \times T$, $t_{\max} = 0.98 \times T$ where $T$ is the total number of training timesteps [S1, unofficial].
- Precision: `torch.float32` [S1, unofficial].

**Steps:**

1. **Initialization:**
   - Initialize Gaussians from a point cloud with isotropic covariance.
   - Enable gradient tracking for $\mathbf{p}_i$, $\mathbf{c}_i$, $\log \alpha_i$, and $\log \mathbf{s}_i$ [S1, unofficial].
   - Load the diffusion model components (VAE, text encoder, UNet, scheduler) [S1, unofficial].

2. **Rendering:**
   - Render the scene from a random camera pose using the Gaussian splatting rasterizer.
   - The rasterizer computes 2D covariance $\Sigma_{2D}$ via the Jacobian $J$ of the projection and view transformation $W$:
     $$ \Sigma_{2D} = J W \Sigma_{3D} W^T J^T $$
   - Add a regularization term of 0.3 to the diagonal elements of $\Sigma_{2D}$ to ensure each Gaussian covers at least ~1 pixel [S1, unofficial].
   - Compute 2D means $\mathbf{m}_{2D}$ by projecting 3D means $\mathbf{p}_i$ to screen space [S1, unofficial].
   - Calculate the exponent for each pixel $k$:
     $$ E_{i,k} = -\frac{1}{2} (\mathbf{x}_k - \mathbf{m}_{2D,i})^T \Sigma_{2D,i}^{-1} (\mathbf{x}_k - \mathbf{m}_{2D,i}) $$
   - Compute alpha values:
     $$ \alpha_{i,k} = \min(0.99, \max(0, \alpha_i \exp(E_{i,k}))) $$
     (Clamped to avoid numerical instability; values $< 1/255$ are set to 0) [S1, unofficial].
   - Compute transmittance $T_k$ for each pixel by accumulating $\alpha$ values in depth order:
     $$ T_k = \prod_{j \le k} (1 - \alpha_{j,k}) $$
   - Render the final image $\hat{I}$ with background color $(0,0,0)$ [S1, unofficial].

3. **SDS Loss Computation:**
   - **Encode:** Normalize $\hat{I}$ from $[0,1]$ to $[-1,1]$. Pass through the VAE encoder to get latent $z$. Scale by $s_{\text{vae}}$ (from `vae.config.scaling_factor`) [S1, unofficial].
   - **Text Embedding:** Tokenize prompt $T$ with padding to `max_length`. Pass through text encoder to get $\mathbf{t}$ [S1, unofficial].
   - **Noise Prediction:** Sample a random timestep $t \in [t_{\min}, t_{\max}]$. Add noise $\epsilon \sim \mathcal{N}(0,1)$ to $z$. Pass noisy latent and $\mathbf{t}$ through the UNet to predict noise $\hat{\epsilon}_\theta(z_t, \mathbf{t})$ [S1, unofficial].
   - **Compute SDS:**
     $$ \mathcal{L}_{SDS} = \mathbb{E}_{t, \epsilon} \left[ w(t) (\hat{\epsilon}_\theta(z_t, \mathbf{t}) - \epsilon) \right]^2 $$
     where $w(t)$ is a weighting function based on the guidance scale $s$ and timestep $t$ (standard SDS weighting) [S1, unofficial].

4. **Optimization:**
   - Backpropagate $\mathcal{L}_{SDS}$ through the rendering pipeline to update Gaussian parameters.
   - Perform an Adam optimizer step.
   - Zero gradients.
   - Repeat for 1,000 iterations [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
from diffusers import StableDiffusionPipeline

def train_gaussian_splatting_sds(gaussians, dataset, prompt, num_iterations=1000):
    # 1. Initialize Optimizer
    params = list(gaussians.parameters())
    optimizer = torch.optim.Adam(params, lr=0.001) # Means use 0.001, others 0.01 in practice
    
    # 2. Load Diffusion Model
    pipe = StableDiffusionPipeline.from_pretrained("stabilityai/stable-diffusion-2-1")
    pipe = pipe.to("cuda")
    pipe.enable_xformers_memory_efficient_attention() # Optional for memory
    
    # 3. Training Loop
    for i in range(num_iterations):
        # Random camera from dataset
        camera, gt_image = dataset.get_random_view()
        
        # Render Scene
        rendered_image = render_scene(gaussians, camera) # Returns RGB [0,1]
        
        # Encode to Latent Space
        rendered_image = (rendered_image * 2 - 1) # Normalize to [-1, 1]
        with torch.no_grad():
            latents = pipe.vae.encode(rendered_image).latent_dist.sample()
            latents = latents * pipe.vae.config.scaling_factor
            
        # Text Embedding
        text_input = pipe.tokenizer(prompt, padding="max_length", max_length=pipe.tokenizer.model_max_length, return_tensors="pt")
        text_embeddings = pipe.text_encoder(text_input.input_ids.to("cuda"))[0]
        
        # Sample Timestep
        t = torch.randint(0, pipe.config.num_train_timesteps, (1,)).item()
        noise = torch.randn_like(latents)
        latents_noisy = pipe.scheduler.add_noise(latents, noise, t)
        
        # Predict Noise
        noise_pred = pipe.unet(latents_noisy, t, encoder_hidden_states=text_embeddings).sample
        
        # Compute SDS Loss
        # Standard SDS weighting: w(t) = (1 - t/T) or similar depending on scheduler
        # For DDIM, often use simple weighting or variance preserving
        weights = get_sds_weights(t, pipe.config.num_train_timesteps)
        loss = (weights * (noise_pred - noise)).pow(2).mean()
        
        # Backpropagate
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # Update Gaussian Parameters (apply activations if stored as pre-act)
        gaussians.apply_activations()
        
        if i % 20 == 0:
            visualize(gaussians, camera)

def get_sds_weights(t, num_timesteps):
    # Simplified weighting; exact formula depends on specific SDS implementation
    # Often proportional to (1 - alpha_t) / sqrt(1 - alpha_t^2) or similar
    # For Stable Diffusion, a common choice is:
    alpha_t = pipe.scheduler.alphas_cumprod[t]
    sigma_t = (1 - alpha_t) / alpha_t
    return sigma_t
```

## Subtopics

### 3D Gaussian Splatting Training Loop
The base optimization loop using photometric loss [15139-3d-gaussian-splatting-training-loop.md].

### TurboGS
Accelerates training by reducing redundant computations [15697-turbogs.md].

### Error-Guided Sparse Pixel Sampling
Optimizes training by focusing on high-error pixels [15708-error-guided-sparse-pixel-sampling.md].

### Frequency Guided Resolution Scheduler
Manages resolution during training [15751-frequency-guided-resolution-scheduler.md].

### Multi-View Regulated Gaussian Splatting
Uses multi-view consistency constraints [15783-multi-view-regulated-gaussian-splatting.md].

### Mip-Splatting Training Loop & Evaluation Pipeline
Handles mip-mapping for anti-aliasing [17975-mip-splatting-training-loop-evaluation-pipeline.md].

### Distributed 3DGS Training Loop with Adaptive Workload Division
Parallelizes training [18650-distributed-3dgs-training-loop-with-adaptive-workload-divisi.md].

### Sparse SH Order Increment
Gradually increases spherical harmonics order [18689-sparse-sh-order-increment.md].

### Regression-based FisherRF
Uses Fisher information for regularization [18747-regression-based-fisherrf.md].

### Gaussian Primitive Training Schedule
Defines the schedule for primitive addition/removal [18977-gaussian-primitive-training-schedule.md].

### Loss Function Design
Details on loss components [19337-loss-function-design.md].

### Local 3DGS Optimization
Optimizes subsets of Gaussians [20073-local-3dgs-optimization.md].

### Immediate 3D Gaussian Splat Reconstruction
Fast reconstruction methods [20247-immediate-3d-gaussian-splat-reconstruction.md].

### Active View Selection and Residual Learning for 3DGS
Selects views for training [20899-active-view-selection-and-residual-learning-for-3dgs.md].

### Uncertainty-Guided Residual Learning for 3DGS
Uses uncertainty for training [20901-uncertainty-guided-residual-learning-for-3dgs.md].

### Submap-based Gaussian SLAM Mapping
SLAM integration [21005-submap-based-gaussian-slam-mapping.md].

### Neural Parametric Gaussians Optimization
Neural network-based Gaussian parameters [21236-neural-parametric-gaussians-optimization.md].

### 3D Gaussian Splatting Optimization
General optimization techniques [22286-3d-gaussian-splatting-optimization.md].

### Gradient Modification
Techniques to modify gradients [22461-gradient-modification.md].

## Operating characteristics
- **Time Complexity:** Dominated by the diffusion model inference (UNet forward pass) per iteration. Rendering is $O(N)$ where $N$ is the number of Gaussians.
- **Memory Complexity:** High due to storing the diffusion model in memory and gradients for all Gaussian parameters.
- **Performance:** Training is significantly slower than photometric methods due to the heavy diffusion model overhead. However, it produces semantically consistent results from few or single views.
- **Strengths:** Does not require multi-view input; leverages powerful 2D priors.
- **Limitations:** Slow training; potential for "smearing" artifacts if guidance scale is too high; sensitive to prompt quality.
- **When to use:** Single-view reconstruction, text-to-3D, or sparse-view scenarios where photometric loss is insufficient.
- **When not to use:** Dense multi-view capture where photometric loss is more efficient and accurate.

## Background
Originated from the application of Score Distillation Sampling (SDS) to 3D Gaussian Splatting, allowing the optimization of Gaussian primitives using pretrained diffusion models. This approach bridges the gap between 2D generative models and 3D representation learning. The method was popularized by works such as "Diffusion-guided 3D Representations" [S1, unofficial], which demonstrated the feasibility of optimizing Gaussians directly with SDS loss.

## Sources
[S1] ShashwatChawla/Diffusion-guided-3D-Representations, https://github.com/ShashwatChawla/Diffusion-guided-3D-Representations, fetched 2026-10-27 (unofficial)
