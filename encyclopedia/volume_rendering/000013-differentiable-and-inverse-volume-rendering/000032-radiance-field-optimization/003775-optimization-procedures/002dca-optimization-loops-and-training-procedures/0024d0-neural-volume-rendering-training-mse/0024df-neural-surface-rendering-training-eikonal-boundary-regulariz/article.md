# Neural Surface Rendering Training (Eikonal & Boundary Regularization)

Also called: Eikonal and Boundary Regularization for Neural Surfaces, Neural SDF Training with Image Reconstruction and Eikonal Regularization

## Overview
This method trains a neural Signed Distance Function (SDF) to render images that match ground truth RGB while enforcing geometric priors via Eikonal and boundary regularization losses [S1, S2, unofficial]. It differs from standard neural volume rendering training (MSE) by explicitly optimizing the gradient magnitude of the SDF to be unity (Eikonal equation) and penalizing non-zero SDF values in the volume (boundary loss), which stabilizes surface extraction and improves geometric accuracy [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ground truth images, camera parameters, a neural SDF model (typically an MLP with positional encoding), and a 3D bounding box for sampling.
*   **Outputs:** Optimized SDF model parameters that minimize the combined loss.

**Data Structures:**
*   **Neural SDF Model:** An MLP that takes 3D coordinates $\mathbf{x}$ and outputs a scalar SDF value $d(\mathbf{x})$ and optionally color features. The model often uses Harmonic Embedding for input coordinates [S1, unofficial].
*   **Ray Bundle:** Contains origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel, generated from pixel coordinates and camera intrinsics/extrinsics [S1, S2, unofficial].
*   **Training Batch:** Contains image tensor, camera object, and iteration index [S1, unofficial].

**Numbered Steps:**

1.  **Pretraining (Optional but recommended):** Pretrain the SDF using a sphere loss or similar geometric prior to initialize the SDF shape before image-based training begins [S1, unofficial].
2.  **Random Point Sampling for Regularization:** Sample $N$ random points $\mathbf{x}_{rand}$ uniformly within the defined 3D bounding box `bounds` [S2, unofficial].
3.  **SDF Evaluation at Random Points:** Compute the SDF value $d(\mathbf{x}_{rand})$ and its gradient $\nabla d(\mathbf{x}_{rand})$ at the sampled random points using the neural network [S2, unofficial].
4.  **Compute Regularization Losses:**
    *   **Eikonal Loss:** Enforce $|\nabla d(\mathbf{x})| = 1$. Compute the mean squared error of the gradient magnitude from 1 [S2, unofficial].
    *   **Boundary Loss:** Penalize non-zero SDF values in the volume. Compute $\mathbb{E}[\exp(-100 \cdot |d(\mathbf{x})|)]$ [S2, unofficial].
5.  **Ray Sampling for Rendering:**
    *   Sample random pixel coordinates $(u, v)$ from the training image dimensions [S1, unofficial].
    *   Generate a ray bundle $\{\mathbf{o}_i, \mathbf{d}_i\}$ for these pixels using the camera parameters [S1, unofficial].
    *   Sample ground truth RGB values $\mathbf{c}_{gt}$ at these pixel coordinates [S1, unofficial].
6.  **Volume/Surface Rendering:**
    *   Render the scene along the rays using the current SDF model. This typically involves sphere tracing or volumetric integration to compute the predicted RGB color $\mathbf{c}_{pred}$ [S1, S2, unofficial].
7.  **Compute Image Reconstruction Loss:** Calculate the Mean Squared Error (MSE) between predicted and ground truth colors: $\mathcal{L}_{image} = \frac{1}{M} \sum_{i=1}^{M} ||\mathbf{c}_{pred}^{(i)} - \mathbf{c}_{gt}^{(i)}||^2$ [S1, S2, unofficial].
8.  **Total Loss Calculation:** Combine the losses:
    $$ \mathcal{L}_{total} = \mathcal{L}_{image} + \lambda_{eik} \mathcal{L}_{eikonal} + \lambda_{inter} \mathcal{L}_{boundary} $$
    where $\lambda_{eik}$ and $\lambda_{inter}$ are weighting factors [S2, unofficial].
9.  **Optimization Step:** Perform backpropagation and update model parameters using an optimizer (e.g., Adam) [S1, unofficial].
10. **Scheduler Update:** Update the learning rate according to the schedule at the end of each epoch [S1, unofficial].

**Formulas:**

*   **Eikonal Loss:**
    $$ \mathcal{L}_{eikonal} = \frac{1}{N} \sum_{j=1}^{N} (|\nabla d(\mathbf{x}_{rand}^{(j)})| - 1)^2 $$
    [S2, unofficial]

*   **Boundary Loss:**
    $$ \mathcal{L}_{boundary} = \lambda_{inter} \cdot \mathbb{E}[\exp(-100 \cdot |d(\mathbf{x})|)] $$
    [S2, unofficial]

*   **Image Loss:**
    $$ \mathcal{L}_{image} = \text{mean}((\mathbf{c}_{gt} - \mathbf{c}_{pred})^2) $$
    [S2, unofficial]

*   **Learning Rate Schedule:**
    $$ \eta_{epoch} = \eta_{initial} \cdot \gamma^{\frac{epoch}{step\_size}} $$
    where $\gamma$ is the decay factor and $step\_size$ is the divisor for the epoch index [S1, unofficial].

**Parameters:**
*   `lr`: Initial learning rate for the optimizer [S1, unofficial].
*   `eikonal_weight` ($\lambda_{eik}$): Weight for the Eikonal loss [S2, unofficial].
*   `inter_weight` ($\lambda_{inter}$): Weight for the boundary loss [S2, unofficial].
*   `bounds`: 3D bounding box defining the region for random point sampling [S2, unofficial].
*   `lr_scheduler_gamma` ($\gamma$): Decay factor for the learning rate schedule [S1, unofficial].
*   `lr_scheduler_step_size`: Divisor for the epoch index in the learning rate schedule exponent [S1, unofficial].
*   `checkpoint_interval`: Number of epochs between model checkpoints [S1, unofficial].
*   `render_interval`: Number of epochs between visualization renders [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_neural_surface_sdf(model, dataloader, optimizer, scheduler, cfg):
    """
    Train a neural SDF using image reconstruction, Eikonal, and boundary regularization.
    
    Args:
        model: Neural SDF model (outputs SDF and optionally color)
        dataloader: Iterator yielding batches of (image, camera)
        optimizer: Optimizer (e.g., Adam)
        scheduler: Learning rate scheduler
        cfg: Configuration dictionary containing hyperparameters
    """
    device = cfg.device
    bounds = cfg.training.bounds
    batch_size = cfg.training.batch_size
    eikonal_weight = cfg.training.eikonal_weight
    inter_weight = cfg.training.inter_weight
    
    # Optional: Pretrain SDF with sphere loss if configured
    if cfg.training.pretrain_sphere:
        pretrain_sdf_with_sphere_loss(model, cfg)

    for epoch in range(cfg.training.epochs):
        for iteration, batch in enumerate(dataloader):
            image = batch['image'].to(device).unsqueeze(0)
            camera = batch['camera'].to(device)
            
            # 1. Sample random points for regularization
            eikonal_points = get_random_points(batch_size, bounds, device)
            
            # 2. Compute SDF and gradients at random points
            eikonal_distances = model.get_distance(eikonal_points)
            eikonal_gradients = model.get_gradient(eikonal_points)
            
            # 3. Compute Eikonal Loss: ||grad(d)||^2 - 1
            grad_magnitude = torch.linalg.norm(eikonal_gradients, dim=-1, keepdim=True)
            eikonal_loss_val = torch.mean((grad_magnitude - 1.0) ** 2) * eikonal_weight
            
            # 4. Compute Boundary Loss: exp(-100 * |d|)
            boundary_loss_val = torch.exp(-1e2 * torch.abs(eikonal_distances)).mean() * inter_weight
            
            # 5. Sample random pixels for rendering
            xy_grid = get_random_pixels_from_image(batch_size, cfg.data.image_size, camera)
            
            # 6. Generate rays from pixels
            ray_bundle = get_rays_from_pixels(xy_grid, cfg.data.image_size, camera)
            
            # 7. Sample ground truth RGB
            rgb_gt = sample_images_at_xy(image, xy_grid)
            
            # 8. Render predicted RGB
            out = model(ray_bundle)
            rgb_hat = out['color']
            
            # 9. Compute Image Loss (MSE)
            image_loss = torch.mean(torch.square(rgb_gt - rgb_hat))
            
            # 10. Total Loss
            loss = image_loss + eikonal_loss_val + boundary_loss_val
            
            # 11. Optimization step
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        # Update learning rate at end of epoch
        scheduler.step()
        
        # Optional: Save checkpoints or render visualizations
        if epoch % cfg.training.checkpoint_interval == 0:
            save_checkpoint(model, optimizer, epoch, cfg)
        if epoch % cfg.training.render_interval == 0:
            render_visualization(model, camera, cfg)

def get_random_points(batch_size, bounds, device):
    """Sample uniform random points within a 3D bounding box."""
    min_bound = torch.tensor(bounds.min, device=device)
    max_bound = torch.tensor(bounds.max, device=device)
    points = torch.rand(batch_size, 3, device=device)
    points = points * (max_bound - min_bound) + min_bound
    return points

def get_random_pixels_from_image(batch_size, image_size, camera):
    """Sample random pixel coordinates (u, v)."""
    h, w = image_size
    u = torch.randint(0, w, (batch_size, 1), dtype=torch.float32)
    v = torch.randint(0, h, (batch_size, 1), dtype=torch.float32)
    return torch.cat([u, v], dim=-1)

def get_rays_from_pixels(xy_grid, image_size, camera):
    """Generate ray origins and directions from pixel coordinates and camera."""
    h, w = image_size
    u, v = xy_grid[:, 0], xy_grid[:, 1]
    
    # Convert pixel coordinates to normalized device coordinates or camera space
    # This depends on the specific camera model implementation
    origins, directions = camera.generate_rays(u, v)
    
    return {'origins': origins, 'directions': directions}

def sample_images_at_xy(image, xy_grid):
    """Sample RGB values from the image tensor at given pixel coordinates."""
    # Implementation depends on image tensor layout (e.g., NHWC or NCHW)
    # Assuming image is [1, H, W, 3]
    h, w = image.shape[1], image.shape[2]
    u = xy_grid[:, 0].long().clamp(0, w-1)
    v = xy_grid[:, 1].long().clamp(0, h-1)
    
    # Gather RGB values
    rgb_gt = image[0, v, u, :]
    return rgb_gt
```

**Edge Cases and Pitfalls:**
*   **Precision:** The boundary loss uses an exponential term $\exp(-100 \cdot |d|)$. If $|d|$ is large, this term underflows to zero. If $|d|$ is very small, it approaches 1. Ensure numerical stability [S2, unofficial].
*   **Boundary Definition:** The `bounds` for random point sampling must encompass the object of interest. Sampling too far from the surface may lead to inefficient gradient updates for the SDF [S2, unofficial].
*   **Gradient Computation:** Computing gradients of the SDF with respect to input coordinates requires `torch.autograd.grad` with `create_graph=True` if higher-order derivatives are needed, or standard backward pass if only first-order gradients are used for the Eikonal loss [S2, unofficial].
*   **Loss Balancing:** The weights `eikonal_weight` and `inter_weight` are critical. Imbalanced weights can lead to poor geometric reconstruction or blurry surfaces [S2, unofficial].

## Subtopics

### [Surf²f Training and Rendering Pipeline](9947-surf-f-training-and-rendering-pipeline.md)
Surf²f is a hybrid neural representation that simultaneously learns an implicit Signed Distance Function (SDF) and an explicit surrogate surface mesh, unified by a shared neural shader for both volume and surface rendering. It solves the synchronization and efficiency issues inherent in separate implicit and explicit surface representations by maintaining a parallel dual-stream architecture. Unlike methods that rely solely on SDF volume rendering or static mesh extraction, Surf²f enables real-time interactive rendering of the extracted mesh while leveraging volume rendering for training stability, outperforming NDS, UNISURF, and NeuS in both geometric accuracy and rendering speed.

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of epochs, batch size, and the cost of rendering (ray marching/sphere tracing) and gradient computation. The Eikonal and boundary losses add overhead for random point sampling and gradient evaluation [S1, S2, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the neural network parameters and the intermediate activations during backpropagation. Random point sampling requires storing $N$ points and their gradients [S1, S2, unofficial].
*   **Measured Performance:** Surf²f, which uses this training approach, outperforms NDS, UNISURF, and NeuS in geometric accuracy and rendering speed [S1, unofficial].
*   **Scaling:** The method scales with the resolution of the input images and the complexity of the neural network. Larger bounding boxes require more random points for effective regularization [S1, S2, unofficial].
*   **Strengths:** Enforces geometric priors (Eikonal equation) leading to cleaner surfaces and better generalization. Boundary regularization helps in defining the surface location more precisely [S1, S2, unofficial].
*   **Limitations:** Requires careful tuning of loss weights. Pretraining may be necessary for complex shapes [S1, unofficial].
*   **When to Use:** When high-quality geometric reconstruction is required, and the input data allows for volume rendering or sphere tracing [S1, S2, unofficial].
*   **When Not to Use:** When real-time inference is the primary constraint and the overhead of training with regularization is prohibitive, or when the input data is sparse and volume rendering is not feasible [S1, unofficial].

## Background
This method originates from the field of neural implicit surface reconstruction and differentiable rendering. It builds upon the principles of Neural Radiance Fields (NeRF) but focuses on explicit surface representation via SDFs. The use of Eikonal regularization for SDFs was popularized by works such as "NeuS" and "UNISURF", which demonstrated that enforcing the Eikonal equation improves surface quality. The specific implementation details provided here are from unofficial repositories [S1, S2, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
