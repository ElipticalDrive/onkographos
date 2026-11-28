# Implicit Neural Representation (INR) for Tomography

Also called: INR reconstruction, Implicit Neural Representation (INR) for Electron Tomography, NeRF for tomography, Neural Radiance Field (NeRF) for tomography

## Overview
This method reconstructs 3D volumes from 2D projection images by representing the volume density as a continuous Implicit Neural Network (INR) rather than a discrete voxel grid [S1, unofficial]. It solves the tomographic reconstruction problem by optimizing the network parameters to minimize the difference between simulated projections and measured data, while simultaneously optimizing the 3D pose (orientation) for each projection [S1, unofficial]. Unlike discrete voxel-based methods such as [Iterative Reconstruction (SIRT via Autodiff)](12326-iterative-reconstruction-sirt-via-autodiff.md), the INR approach enables arbitrarily fine sampling and projection at any orientation without interpolation, making it particularly robust to missing wedges, large tilt steps, and low electron fluence noise [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 2D projection images $P_i$ (where $i=1 \dots N$), their corresponding initial estimated poses (rotation matrices $R_i$ and translation vectors $t_i$), and the camera geometry (ray origins and directions).
*   **Outputs:** The optimized network weights $\theta$ defining the density field $\rho(x)$, and the refined 3D poses for each projection.

**Data Structures:**
*   **Neural Network:** A Multi-Layer Perceptron (MLP) $f_\theta: \mathbb{R}^3 \to \mathbb{R}$.
    *   **Input:** 3D spatial coordinates $(x, y, z)$.
    *   **Output:** Scalar density value $\rho$.
    *   **Activation:** Typically ReLU or Sine (Sin) activations are used in INR literature, though specific activation functions are not explicitly detailed in the provided notes [S1, unofficial].
*   **Pose Parameters:** For each projection $i$, a 6-degree-of-freedom pose vector $p_i = [R_i, t_i]$ is stored as a trainable parameter tensor.

**Numbered Steps:**

1.  **Coarse Preprocessing:** Perform a coarse alignment or initial pose estimation to reduce large degrees of misalignment. This step is critical because large initial misalignments significantly increase convergence time for the INR pose optimization [S1, unofficial]. [G] This typically involves standard algebraic reconstruction techniques (like SIRT) or feature-based alignment to provide a starting point within the basin of attraction of the global optimum.

2.  **Forward Projection (Differentiable Rendering):**
    For each projection $i$:
    a. Define a set of 2D pixel coordinates $(u, v)$ for the image plane.
    b. Map these 2D coordinates to 3D ray origins and directions based on the current pose $p_i$.
    c. Sample $K$ points along each ray within the volume bounds.
    d. Query the neural network $f_\theta$ at each sampled point to obtain density values $\rho_k$.
    e. Compute the line integral (projection) using numerical quadrature (e.g., trapezoidal rule or simple summation) to simulate the projection $P_i^{sim}$:
       $$ P_i^{sim}(u, v) = \sum_{k=1}^{K} \rho(f_\theta(x_k, y_k, z_k)) \cdot \Delta s_k $$
       where $\Delta s_k$ is the step size along the ray.

3.  **Loss Calculation:**
    Compute the Mean Squared Error (MSE) between the simulated projection $P_i^{sim}$ and the measured projection $P_i^{meas}$:
    $$ L = \frac{1}{N} \sum_{i=1}^{N} \| P_i^{sim} - P_i^{meas} \|^2_2 $$

4.  **Joint Optimization:**
    Use an optimizer (e.g., Adam) to update both the network weights $\theta$ and the pose parameters $p_i$ simultaneously to minimize $L$.
    *   The network weights $\theta$ are updated via backpropagation through the differentiable forward projection.
    *   The poses $p_i$ are updated via backpropagation through the ray casting operation.

5.  **Holdout Validation and Early Stopping:**
    Split the projection data into a training set and a holdout (validation) set. Monitor the loss on the holdout set. Stop training when the holdout loss reaches its minimum to prevent overfitting to noise, which is a significant risk at low electron fluence [S1, unofficial].

6.  **Volume Extraction:**
    Once optimized, the 3D volume is represented by the neural network $f_\theta$. To visualize or save the volume, evaluate $f_\theta$ on a regular 3D grid of coordinates $(x, y, z)$ and store the resulting density values.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class DensityNetwork(nn.Module):
    def __init__(self, input_dim=3, hidden_dim=256, output_dim=1, num_layers=8):
        super(DensityNetwork, self).__init__()
        layers = []
        for _ in range(num_layers):
            layers.append(nn.Linear(input_dim, hidden_dim))
            layers.append(nn.ReLU()) # [G] Standard activation; Sin is also common in INRs
            input_dim = hidden_dim
        layers.append(nn.Linear(hidden_dim, output_dim))
        self.network = nn.Sequential(*layers)

    def forward(self, x):
        return self.network(x)

def ray_casting(network, poses, image_shape, ray_steps=100):
    """
    Simulates projections for a batch of poses.
    poses: Tensor of shape (N, 6) representing rotation and translation.
    image_shape: Tuple (H, W).
    """
    N = poses.shape[0]
    H, W = image_shape
    projections = []

    for i in range(N):
        # Extract pose parameters for projection i
        # [G] Assuming poses are stored as [R11, R12, R13, R21, R22, R23, R31, R32, R33, tx, ty, tz]
        # or similar. Implementation depends on specific pose representation.
        # For simplicity, assume a function `get_ray_directions` exists that uses the pose
        # to map 2D pixel coordinates to 3D rays.
        
        # Generate 2D grid coordinates
        u, v = torch.meshgrid(torch.arange(W), torch.arange(H), indexing='ij')
        u = u.float().to(poses.device)
        v = v.float().to(poses.device)
        
        # Map to 3D rays based on pose i
        # ray_origins: (H, W, 3), ray_directions: (H, W, 3)
        ray_origins, ray_directions = get_ray_directions(u, v, poses[i])
        
        # Sample points along rays
        t = torch.linspace(0, 1, ray_steps).to(poses.device)
        # Expand dimensions for broadcasting
        t = t.view(-1, 1, 1, 1)
        ray_origins = ray_origins.unsqueeze(0) # (1, H, W, 3)
        ray_directions = ray_directions.unsqueeze(0) # (1, H, W, 3)
        
        # Sampled points: (1, H, W, ray_steps, 3)
        points = ray_origins + t * ray_directions
        points = points.view(-1, ray_steps, 3) # (H*W, ray_steps, 3)
        
        # Query network
        densities = network(points) # (H*W, ray_steps, 1)
        
        # Integrate along ray (simple sum for now, weighted by step size)
        # [G] More accurate integration uses quadrature weights
        projection = densities.sum(dim=1) # (H*W, 1)
        projection = projection.view(H, W)
        projections.append(projection)
        
    return torch.stack(projections) # (N, H, W)

def train_inr_tomography(projections, initial_poses, network, num_epochs=1000, holdout_ratio=0.1):
    """
    Main training loop for INR Tomography with joint alignment.
    """
    N = projections.shape[0]
    H, W = projections.shape[1], projections.shape[2]
    
    # Split data into train and holdout
    indices = torch.randperm(N)
    split = int(N * (1 - holdout_ratio))
    train_indices = indices[:split]
    holdout_indices = indices[split:]
    
    train_projections = projections[train_indices]
    holdout_projections = projections[holdout_indices]
    
    # Initialize trainable poses
    # [G] Initial poses should be provided by coarse preprocessing
    train_poses = nn.Parameter(initial_poses[train_indices].clone())
    holdout_poses = nn.Parameter(initial_poses[holdout_indices].clone())
    
    optimizer = optim.Adam(list(network.parameters()) + [train_poses, holdout_poses], lr=1e-3)
    
    best_holdout_loss = float('inf')
    best_network_state = None
    best_train_poses_state = None
    best_holdout_poses_state = None
    
    for epoch in range(num_epochs):
        optimizer.zero_grad()
        
        # Forward pass on training data
        train_sim = ray_casting(network, train_poses, (H, W))
        train_loss = nn.MSELoss()(train_sim, train_projections)
        
        # Forward pass on holdout data
        holdout_sim = ray_casting(network, holdout_poses, (H, W))
        holdout_loss = nn.MSELoss()(holdout_sim, holdout_projections)
        
        # Backward pass
        (train_loss + holdout_loss).backward()
        optimizer.step()
        
        # Early stopping check
        if holdout_loss.item() < best_holdout_loss:
            best_holdout_loss = holdout_loss.item()
            best_network_state = {k: v.clone() for k, v in network.state_dict().items()}
            best_train_poses_state = train_poses.data.clone()
            best_holdout_poses_state = holdout_poses.data.clone()
            
        if epoch % 100 == 0:
            print(f"Epoch {epoch}: Train Loss {train_loss.item():.4f}, Holdout Loss {holdout_loss.item():.4f}")
            
    # Restore best model
    network.load_state_dict(best_network_state)
    train_poses.data = best_train_poses_state
    holdout_poses.data = best_holdout_poses_state
    
    return network, torch.cat([train_poses, holdout_poses])

# Helper function placeholder for ray direction calculation
def get_ray_directions(u, v, pose):
    # [G] This function depends on the specific camera model and pose representation.
    # It maps 2D pixel coordinates to 3D rays given the current rotation and translation.
    raise NotImplementedError("Implementation depends on specific camera geometry.")
```

**Parameters:**
*   **Tilt Step:** The angular increment between projections. Typical values tested include $1^\circ$, $2^\circ$, $5^\circ$, $10^\circ$, $12^\circ$, $14^\circ$, $20^\circ$, $28^\circ$, and $70^\circ$ [S1, unofficial].
*   **Missing Wedge Angle:** The angular range not covered by projections. Tested ranges include $(-80^\circ, 80^\circ)$, $(-70^\circ, 70^\circ)$, $(-50^\circ, 50^\circ)$, $(-40^\circ, 40^\circ)$, $(-20^\circ, 20^\circ)$, and $(-10^\circ, 10^\circ)$ [S1, unofficial].
*   **Electron Fluence:** Measured in counts/px. Tested values include $10^4$, $10^3$, $10^2$, $10^1$, and $1$ count/px [S1, unofficial].
*   **Holdout Ratio:** Not explicitly given, but a split is required for early stopping [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overfitting to Noise:** At very low electron fluence (e.g., 1 count/px), the INR overfits to noise. Holdout validation is required for early stopping to output the volume when loss is minimal on the holdout set [S1, unofficial].
*   **Convergence Time:** Large degrees of rotational misalignment correspond to longer convergence times for the INR pose optimization. A coarse preprocessing routine is recommended to speed up convergence [S1, unofficial].
*   **Initialization:** The method requires initial pose estimates. Poor initialization can lead to local minima or slow convergence [S1, unofficial].

## Subtopics

### [INR Tomography with Joint Alignment](19595-inr-tomography-with-joint-alignment.md)
A fully self-supervised tomographic reconstruction algorithm that uses an Implicit Neural Representation (INR) to jointly optimize the 3D volume density field and the full 3D pose (orientation and in-plane translation) for each projection image. Unlike conventional voxel-grid methods that require interpolation during rotation and struggle with multi-GPU parallelization, this method represents the volume as a continuous neural network, enabling arbitrarily fine sampling and projection at any orientation without interpolation. It is particularly suited for heterogeneous materials where supervised training data is unavailable, solving for complete sample pose rather than relying on pre-estimated shifts or single tilt angles.

### IntraTomo
A three-step CT reconstruction approach leveraging neural fields to enhance final reconstruction quality [S2].
*   **Step 1:** Optimize a neural field $F_\Phi$ by minimizing the data-fidelity term on sparse measurements. Given sparse measurements $b \in \mathbb{R}^m$, forward operator $A \in \mathbb{R}^{m \times n}$, and voxel coordinates $X_{grid} \in \mathbb{R}^{n \times 3}$, solve: $\hat{\Phi} = \arg \min_\Phi \frac{1}{2} \|A F_\Phi(X_{grid}) - b\|^2_2$ [S2].
*   **Step 2:** Generate dense synthetic measurements $b_{new} \in \mathbb{R}^{m_{full}}$ by applying the full-range operator $A_{full} \in \mathbb{R}^{m_{full} \times n}$ to the learned field: $b_{new} = A_{full} F_{\hat{\Phi}}(X_{grid})$, where $m_{full} \gg m$ and $m_{full} \approx n$ [S2].
*   **Step 3:** Perform regularized reconstruction on the pixel basis using a weighted data-fidelity term combining synthetic and real fidelity, plus a regularization term $R$ (linear combination of total variation and nonlocal means). Solve: $\hat{f} = \arg \min_{f \in \mathbb{R}^n} L(f_e) + \lambda R(f)$, where $L(f_e) = \frac{1}{2}(\alpha \|A_{full} f - b_{new}\|^2_2 + (1-\alpha) \|Af - b\|^2_2)$ [S2].
*   **Parameter:** $\alpha > 0$ is a weighting term that balances the synthetic fidelity (from the neural field) with the real fidelity (from sparse measurements) [S2].
*   **Pitfall:** The pixel-basis reconstruction in Step 3 acts as a memory bottleneck, forcing reconstruction on the CPU for large-scale problems [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The memory footprint is dominated by the neural network weights and the pose parameters, which is significantly lower than storing a high-resolution 3D voxel grid. However, the forward projection step requires sampling many points along rays, which can be memory-intensive if not batched efficiently [S1, unofficial].
*   **Time:** Convergence time is sensitive to the initial pose alignment. Large misalignments lead to longer convergence times [S1, unofficial]. The method is generally slower per iteration than closed-form methods like SIRT but may require fewer iterations to achieve high fidelity in challenging scenarios (e.g., missing wedges) [S1, unofficial].

**Measured Performance:**
*   **Missing Wedge Robustness:** INR is more robust to missing wedges than SIRT. It reconstructs internal features accurately up to a $100^\circ$ missing wedge, whereas SIRT degrades significantly and shows streaking artifacts [S1, unofficial].
    *   SSIM of 0.985 for INR vs 0.972 for SIRT with a $20^\circ$ missing wedge (tilt range $-80^\circ$ to $80^\circ$) and $1^\circ$ tilt increment [S1, unofficial].
*   **Tilt Step Robustness:** INR is more robust to large tilt steps than SIRT. It generates accurate volumes up to a $20^\circ$ step size, while SIRT significantly suffers with tilt steps greater than $5^\circ$ [S1, unofficial].
    *   SSIM of 0.957 for INR vs 0.427 for SIRT using only 9 projections ($20^\circ$ tilt step) with high-angle tilts included [S1, unofficial].
*   **Low Fluence/Denoising:** INR denoises projections better than SIRT at low electron fluence. At 100 counts/px, SIRT is visibly noisy while INR is almost noise-free [S1, unofficial].
    *   SSIM of 0.857 for INR vs 0.293 for SIRT at a dose of 10 count/px [S1, unofficial].
*   **Pose Optimization:** INR with pose optimization achieves an average SSIM of 0.974 regardless of the degree of rotational misalignment [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** High robustness to missing wedges, large tilt steps, and low signal-to-noise ratios. Continuous representation allows for arbitrary resolution and interpolation-free rotation. Self-supervised alignment improves accuracy in the presence of pose errors [S1, unofficial].
*   **Limitations:** Slower convergence compared to algebraic methods. Sensitive to initialization. Risk of overfitting to noise at very low fluence without early stopping. [S1, unofficial].

**When to Use:**
*   Use when dealing with sparse views, large missing wedges, or low-dose data where traditional methods like SIRT fail or produce significant artifacts [S1, unofficial].
*   Use when precise pose estimation is unknown or inaccurate, and joint alignment is beneficial [S1, unofficial].

**When Not to Use:**
*   Use when computational resources are extremely limited and fast, approximate reconstructions are sufficient [S1, unofficial].
*   Use when high-quality, dense tilt series with minimal missing wedges are available, and speed is the primary concern [S1, unofficial].

## Background
**Origin:**
The method was introduced in the paper "Missing Wedge Inpainting and Joint Alignment in Electron Tomography through Implicit Neural Representations" [S1, unofficial]. It applies the concept of Implicit Neural Representations, popularized in computer graphics for novel view synthesis (e.g., NeRF), to the domain of electron tomography [S1, unofficial].

**History and Development:**
The approach leverages the differentiability of neural networks to solve the inverse problem of tomography. By treating the volume as a continuous function parameterized by a neural network, it avoids the discretization errors associated with voxel-based methods. The joint optimization of pose and density allows for self-supervised alignment, addressing a common challenge in electron tomography where precise alignment is difficult to achieve [S1, unofficial].

**Usage:**
The method has been experimentally validated using STEM (Scanning Transmission Electron Microscopy) instruments in high annular dark field (HAADF) imaging mode [S1, unofficial]. It has been applied to reconstruct catalytic nanoparticles on a support, demonstrating its ability to resolve fine features and mitigate missing wedge artifacts [S1, unofficial].

## Sources
[S1] Missing Wedge Inpainting and Joint Alignment in Electron Tomography through Implicit Neural Representations, https://arxiv.org/abs/2512.08113, fetched 2026-10-27 (unofficial)
[S2] Neural field regularization by denoising for 3D sparse-view X-ray computed tomography, https://doi.org/10.1109/3DV62453.2024.00094, fetched 2026-10-26
