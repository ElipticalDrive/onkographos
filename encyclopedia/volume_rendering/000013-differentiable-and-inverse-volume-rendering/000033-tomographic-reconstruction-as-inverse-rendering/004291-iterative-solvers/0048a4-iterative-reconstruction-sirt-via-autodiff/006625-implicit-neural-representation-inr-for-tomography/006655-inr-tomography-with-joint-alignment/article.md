# INR Tomography with Joint Alignment

Also called: INR Tomography Optimization with Pose Alignment, Implicit Neural Representation Tomography, Joint Alignment and Reconstruction, Self-supervised INR Tomography, Self-supervised INR reconstruction

## Overview
A fully self-supervised tomographic reconstruction algorithm that uses an Implicit Neural Representation (INR) to jointly optimize the 3D volume density field and the full 3D pose (orientation and in-plane translation) for each projection image [S1, unofficial]. Unlike conventional voxel-grid methods that require interpolation during rotation and struggle with multi-GPU parallelization, this method represents the volume as a continuous neural network, enabling arbitrarily fine sampling and projection at any orientation without interpolation [S1, unofficial]. It is particularly suited for heterogeneous materials where supervised training data is unavailable, solving for complete sample pose rather than relying on pre-estimated shifts or single tilt angles [S1, unofficial].

## How it works
**Inputs:** A tilt series of $M$ 2D projection images $I^{exp}_{i,j}$, where $i \in \{1, \dots, N\}$ is the pixel index and $j \in \{1, \dots, M\}$ is the tilt index. The nominal tilt angle $\theta_j$ for each image is assumed known.
**Outputs:** The weights of the INR volume representation and the optimized pose parameters for each tilt image.

**Data Structures:**
1.  **INR Model:** A fully-connected neural network (MLP) with periodic activation functions.
    *   Input: 3D coordinate vector $\mathbf{r} = [x, y, z]^T$.
    *   Output: Scalar intensity value $\phi$.
2.  **Pose Parameters:** For each tilt image $j$, a set of learnable parameters:
    *   $z1_j, z3_j$: Learnable Euler angles representing tilt axis oscillation and rotation around the beam axis.
    *   $\delta x_j, \delta y_j$: Learnable scalars representing in-plane translation shifts along the x and y axes.
3.  **Optimization State:**
    *   INR weights $W$.
    *   Pose parameters $\mathbf{P}_j = [z1_j, z3_j, \delta x_j, \delta y_j]^T$ for $j=1 \dots M$.

**Formulas:**
The total loss function is:
$$ L = L_{pixel} + \alpha L_{TV} $$
where $\alpha$ is the total variation regularization weight [S1, unofficial].

The pixel loss is defined as:
$$ L_{pixel} = \frac{1}{MN} \sum_{j=1}^{M} \sum_{i=1}^{N} \text{smoothL1}(I^{exp}_{i,j} - I^{pred}_{i,j}) $$

The smoothL1 loss function is:
$$ \text{smoothL1}(x) = \begin{cases} 0.5x^2/\beta & \text{if } |x| < \beta \\ |x| - 0.5\beta & \text{otherwise} \end{cases} $$
where $\beta = 0.07$ is the crossover point hyperparameter [S1, unofficial].

The predicted intensity $I^{pred}_{i,j}$ for pixel $i$ in image $j$ is computed by integrating the INR output along a ray:
1.  Define a base ray $\mathbf{r}_0(t) = [0, 0, t]^T$ (along the z-axis).
2.  Apply in-plane shifts $\delta x_j, \delta y_j$ to the ray origin in the image plane.
3.  Apply rotation $R(\mathbf{p}_j)$ to the ray direction and origin, where $R(\mathbf{p}_j) = R(z3_j) R(\theta_j) R(z1_j)$. Here $R(\theta_j)$ is the nominal rotation around the tilt axis, and $R(z1_j), R(z3_j)$ are the learned corrections [S1, unofficial].
4.  Sample discrete points $\mathbf{r}_k$ along the transformed ray.
5.  Evaluate $\phi(\mathbf{r}_k)$ using the INR.
6.  Integrate $\phi$ along the ray to obtain $I^{pred}_{i,j}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize INR weights $W$ randomly (or pre-trained on SIRT). Initialize pose parameters $\mathbf{P}_j$ to identity/zero (nominal pose).
2.  **Sampling:** For each training step, sample a batch of pixels from the tilt series. The batch size is 1024 pixels per GPU [S1, unofficial].
3.  **Forward Pass:**
    a. For each pixel $(i, j)$ in the batch, construct the ray based on current pose $\mathbf{P}_j$ and nominal angle $\theta_j$.
    b. Sample $K$ points along the ray. $K$ starts small and ramps up over the first few epochs [S1, unofficial].
    c. Evaluate INR at these points to get intensities.
    d. Integrate intensities to predict $I^{pred}_{i,j}$.
4.  **Loss Calculation:** Compute $L_{pixel}$ using the smoothL1 formula. Compute $L_{TV}$ (Total Variation) on the INR outputs if regularization is active. Compute total loss $L$.
5.  **Backpropagation:** Compute gradients $\frac{\partial L}{\partial W}$ and $\frac{\partial L}{\partial \mathbf{P}_j}$ using automatic differentiation.
6.  **Update:** Update $W$ and $\mathbf{P}_j$ using the Adam optimizer [S1, unofficial].
7.  **Validation (Optional):** If SNR is high, split data into 80% training / 20% validation. Monitor $L_{pixel}$ on validation set. Stop if loss increases [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class PeriodicActivation(nn.Module):
    """Periodic activation function for INR."""
    def __init__(self, omega_0):
        super().__init__()
        self.omega_0 = omega_0

    def forward(self, x):
        return torch.sin(self.omega_0 * x)

class INRVolume(nn.Module):
    """Fully connected neural network representing the 3D volume."""
    def __init__(self, input_dim=3, output_dim=1, hidden_dim=256, num_layers=8, omega_0=30.0):
        super().__init__()
        self.layers = nn.ModuleList()
        self.omega_0 = omega_0
        
        # Input layer
        self.layers.append(nn.Linear(input_dim, hidden_dim))
        self.layers.append(PeriodicActivation(omega_0))
        
        # Hidden layers
        for _ in range(num_layers - 2):
            self.layers.append(nn.Linear(hidden_dim, hidden_dim))
            self.layers.append(PeriodicActivation(omega_0))
            
        # Output layer
        self.layers.append(nn.Linear(hidden_dim, output_dim))

    def forward(self, x):
        for layer in self.layers:
            x = layer(x)
        return x

def compute_rotation_matrix(z1, theta, z3):
    """
    Compute rotation matrix R(z3) * R(theta) * R(z1).
    Assumes standard Euler angle conventions compatible with the source.
    R(theta) is rotation around the tilt axis (typically Y or X depending on setup).
    For this implementation, we assume a generic rotation composition.
    Note: Exact axis definitions for z1, theta, z3 must match the experimental setup.
    Typically: z1 is tilt axis wobble, theta is nominal tilt, z3 is in-plane rotation.
    """
    # Placeholder for specific rotation matrix construction based on axis definitions.
    # The source specifies R(z3_j)R(theta_j)R(z1_j).
    # We assume rotations are around specific axes (e.g., Z, Y, Z or similar).
    # For a generic implementation, one must define the axes.
    # Let's assume standard Z-Y-Z or similar. The source does not specify axes explicitly,
    # but implies a composition of three rotations.
    # We will use a generic rotation composition function.
    pass

def sample_ray(ray_origin, ray_direction, num_samples):
    """Sample points along a ray."""
    t = torch.linspace(0, 1, num_samples)
    points = ray_origin.unsqueeze(1) + t.unsqueeze(0).unsqueeze(2) * ray_direction.unsqueeze(1)
    return points

def integrate_ray(points, inr_model):
    """Evaluate INR and integrate along ray."""
    intensities = inr_model(points) # Shape: [num_samples, 1]
    # Simple trapezoidal or midpoint integration
    # Assuming points are uniformly spaced
    dy = 1.0 / (points.shape[0] - 1) # Normalized step size
    integral = torch.trapz(intensities.squeeze(-1), dx=dy)
    return integral

def smooth_l1_loss(input, target, beta=0.07):
    diff = input - target
    abs_diff = torch.abs(diff)
    loss = torch.where(abs_diff < beta, 
                       0.5 * diff**2 / beta, 
                       abs_diff - 0.5 * beta)
    return torch.mean(loss)

def total_variation_loss(inr_model, coordinates):
    """Compute Total Variation regularization on INR outputs."""
    # Evaluate INR at coordinates
    outputs = inr_model(coordinates)
    # Compute gradient of outputs w.r.t coordinates
    # This is computationally expensive. Often approximated or computed via autodiff.
    # For simplicity, we assume a helper function exists or use torch.autograd.grad
    grad_outputs = torch.autograd.grad(outputs.sum(), coordinates, create_graph=True)[0]
    tv = torch.mean(torch.norm(grad_outputs, dim=1))
    return tv

def train_inr_tomography(tilt_images, nominal_angles, num_epochs=150, num_gpus=16):
    """
    Main training loop for INR Tomography with Joint Alignment.
    """
    # Hyperparameters
    lr_inr = 8e-6
    lr_pose = 5e-2
    beta = 0.07
    alpha_tv = 1e-4
    batch_size_per_gpu = 1024
    warmup_epochs = 10
    
    # Initialize INR
    inr = INRVolume()
    
    # Initialize Pose Parameters
    # M tilt images
    M = len(tilt_images)
    N = tilt_images[0].shape[0] # Assuming square images NxN
    
    # Learnable pose parameters: [z1, z3, dx, dy] for each tilt
    pose_params = nn.Parameter(torch.zeros(M, 4))
    
    # Optimizers
    optimizer_inr = optim.Adam(inr.parameters(), lr=lr_inr)
    optimizer_pose = optim.Adam([pose_params], lr=lr_pose)
    
    # Learning rate schedulers
    scheduler_inr = optim.lr_scheduler.CosineAnnealingLR(optimizer_inr, T_max=num_epochs)
    # Warmup for inr
    # Scheduler for pose can be constant or scaled
    
    # Distributed Data Parallel setup would go here
    # inr = DDP(inr)
    
    for epoch in range(num_epochs):
        # Warmup logic
        if epoch < warmup_epochs:
            current_lr = lr_inr * (epoch + 1) / warmup_epochs
            for param_group in optimizer_inr.param_groups:
                param_group['lr'] = current_lr
        
        # Sample batch of pixels
        # Randomly select batch_size_per_gpu pixels from the tilt series
        # Flatten images to 1D array of pixels
        # Indices: (image_idx, pixel_idx)
        
        batch_indices = sample_batch_indices(M, N, batch_size_per_gpu)
        
        optimizer_inr.zero_grad()
        optimizer_pose.zero_grad()
        
        total_loss = 0.0
        num_samples = 0
        
        for idx in batch_indices:
            img_idx, pix_idx = idx
            # Get experimental intensity
            exp_intensity = tilt_images[img_idx][pix_idx]
            
            # Get nominal angle
            theta = nominal_angles[img_idx]
            
            # Get current pose parameters
            z1, z3, dx, dy = pose_params[img_idx]
            
            # Construct ray
            # 1. Base ray along Z
            ray_origin = torch.tensor([0.0, 0.0, 0.0])
            ray_direction = torch.tensor([0.0, 0.0, 1.0])
            
            # 2. Apply shifts dx, dy to origin (in-plane)
            # Assuming x,y are in-plane, z is beam
            ray_origin[0] += dx
            ray_origin[1] += dy
            
            # 3. Apply rotations R(z3) R(theta) R(z1)
            # Note: Exact rotation matrix construction depends on coordinate system
            # We assume a helper function that applies these rotations to the ray
            ray_origin, ray_direction = apply_pose_transform(ray_origin, ray_direction, z1, theta, z3)
            
            # 4. Sample points along ray
            # Number of samples ramps up with epoch
            num_samples_ray = ramp_up_samples(epoch)
            points = sample_ray(ray_origin, ray_direction, num_samples_ray)
            
            # 5. Evaluate INR
            intensities = inr(points)
            
            # 6. Integrate
            pred_intensity = integrate_ray(points, inr)
            
            # 7. Compute Loss
            pixel_loss = smooth_l1_loss(pred_intensity, exp_intensity, beta)
            total_loss += pixel_loss
            num_samples += 1
            
        # Average loss
        total_loss /= num_samples
        
        # Add Total Variation regularization
        # Sample a subset of coordinates for TV calculation to save memory
        # Or compute on the batch coordinates
        # For simplicity, we assume TV is computed on the batch
        # tv_loss = total_variation_loss(inr, batch_coordinates)
        # total_loss += alpha_tv * tv_loss
        
        # Backpropagate
        total_loss.backward()
        
        # Update
        optimizer_inr.step()
        optimizer_pose.step()
        
        # Scheduler step
        scheduler_inr.step()
        
        # Validation check if needed
        # if epoch % 10 == 0:
        #     val_loss = evaluate_validation(inr, pose_params, val_data)
        #     if val_loss > best_val_loss:
        #         break

    return inr, pose_params
```

**Parameters:**
*   **Network Architecture:** Fully-connected neural network with periodic activation functions. Hidden dimension and number of layers are hyperparameters; the source notes that this architecture required the least hyperparameter tuning to converge to high spatial frequencies [S1, unofficial].
*   **Loss Function:** smoothL1 loss with $\beta = 0.07$ [S1, unofficial].
*   **Regularization:** Total Variation (TV) loss with weight $\alpha$. For phantom data, $\alpha = 1 \times 10^{-4}$ [S1, unofficial].
*   **Learning Rate (INR Weights):** $8 \times 10^{-6}$ [S1, unofficial].
*   **Learning Rate (Pose Parameters):** $5 \times 10^{-2}$ [S1, unofficial].
*   **Batch Size:** 1024 pixels per GPU. Global batch size 16384 with 16 GPUs [S1, unofficial].
*   **Optimizer:** Adam [S1, unofficial].
*   **LR Schedule:** Warmup for 10 epochs, then cosine-annealing [S1, unofficial].
*   **Sampling:** Number of sampling points per ray starts small and ramps up over the first few epochs [S1, unofficial].
*   **Initialization:** Random initialization is sufficient for self-supervision. Pre-training on a converged SIRT reconstruction can speed up convergence [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Interpolation:** Conventional voxel-grid methods require interpolation during rotation, which is difficult to parallelize across multiple GPUs because the volume must be instantiated across each GPU [S1, unofficial]. The INR method avoids this by using precisely rotated projection vectors [S1, unofficial].
*   **Overfitting:** In high SNR scenarios, the model may overfit to noise. Mitigation involves splitting data into 80% training and 20% validation, and stopping training when validation loss increases [S1, unofficial].
*   **Parallelization:** The method is parallelizable via Distributed Data Parallel (DDP). Gradients are averaged across GPUs, and the learning rate is scaled by the square root of the number of GPUs [S1, unofficial].
*   **Coordinate System:** The exact definition of the rotation matrices $R(z1_j), R(\theta_j), R(z3_j)$ depends on the experimental setup (e.g., tilt axis orientation). The source specifies the composition order but not the specific axes for each angle, which must be derived from the hardware configuration [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Reconstruction speed scales nearly linearly with the number of GPUs. For 150 epochs on a full dataset using 16 GPUs, reconstruction takes approximately 2 hours [S1, unofficial]. Without multi-GPU systems, speed depends heavily on volume size and single-GPU performance.
*   **Memory Complexity:** Memory usage is dominated by the INR model size and the batch of sampled coordinates. Since the volume is not stored as a grid, memory usage is independent of the voxel grid resolution, allowing for high-resolution representations without proportional memory increase [S1, unofficial].
*   **Strengths:**
    *   Self-supervised: No training data from similar samples required [S1, unofficial].
    *   Joint Alignment: Optimizes full 3D pose (orientation and in-plane translation) for each projection [S1, unofficial].
    *   Missing Wedge Inpainting: Acts as a regularizer to fill in missing information [S1, unofficial].
    *   Denoising: Regularization helps denoise low-dose datasets [S1, unofficial].
    *   No Interpolation: Allows arbitrarily fine sampling and projection at any orientation [S1, unofficial].
*   **Limitations:**
    *   Computationally intensive per epoch compared to analytical methods like SIRT, though parallelization mitigates this.
    *   Requires careful tuning of regularization parameter $\alpha$ [S1, unofficial].
    *   Convergence may be slow without pre-training or good initial alignment [S1, unofficial].
*   **When to Use:** Suitable for heterogeneous materials, beam-sensitive samples (low dose), and datasets with significant misalignment or missing wedges where supervised methods are impractical [S1, unofficial].
*   **When Not to Use:** When computational resources are severely limited (single CPU, no GPU) and real-time reconstruction is required, or when high-quality training data from similar samples is available and supervised methods are preferred.

## Background
The method was proposed in the paper "Missing Wedge Inpainting and Joint Alignment in Electron Tomography through Implicit Neural Representations" [S1, unofficial]. It addresses limitations of conventional voxel-grid tomographic reconstruction, which requires interpolation and struggles with parallelization, and supervised deep learning methods that require training data. The approach leverages the properties of Implicit Neural Representations (INRs) to provide a continuous, differentiable volume representation that can be optimized jointly with projection poses. It is implemented in the open-source Python package quantEM [S1, unofficial]. The method is applicable to HAADF STEM data, where image intensity depends linearly on the projected potential [S1, unofficial].

## Sources
[S1] Missing Wedge Inpainting and Joint Alignment in Electron Tomography through Implicit Neural Representations, https://arxiv.org/abs/2512.08113, fetched 2026-10-27 (unofficial)
