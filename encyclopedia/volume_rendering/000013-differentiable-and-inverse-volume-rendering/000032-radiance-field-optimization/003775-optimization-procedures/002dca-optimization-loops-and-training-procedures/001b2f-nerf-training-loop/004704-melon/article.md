# MELON

Also called: Modulo-Equivalent Latent Optimization of NeRF, NeRF with Unposed Images in SO(3)

## Overview
MELON is a method for 3D inverse rendering that simultaneously reconstructs a neural radiance field and estimates camera poses from a set of unposed images [S1]. It solves the problem of local minima in pose estimation by using an encoder to amortize inference and a "modulo loss" that maps the latent pose space to a quotient set, allowing gradient descent to converge despite rotational symmetries [S1]. Unlike adversarial methods such as GNeRF, MELON does not require a discriminator, enabling ab-initio reconstruction with few images [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ unposed images $\{I_i\}_{i=1}^n$. The images are assumed to be renderings of an unknown density $\sigma^*$ and radiance $c^*$ from unknown cameras $\{R_i^*\}$ [S1].
*   **Output:** A neural radiance field parameterized by $\psi$ (predicting $f_\psi = (c_\psi, \sigma_\psi)$) and a set of estimated camera poses $\{R_i\}_{i=1}^n$ [S1].

**Data Structures:**
*   **Neural Radiance Field:** A coordinate-based neural network parameterized by $\psi$. It takes a 3D position and viewing direction as input and outputs color and density [S1].
*   **Encoder:** A CNN encoder $h_\xi$ parameterized by weights $\xi$. It maps an input image $I_i$ to a latent pose variable $z_i$ in the latent space $\Omega$ [S1].
*   **Latent Space:** The space of camera orientations $\Omega$, defined as $SO(3)$ or $SO(2)$ [S1].
*   **Camera Parameterization:** Each camera $R_i$ is defined by three parameters: azimuth $\theta \in SO(2)$, elevation $\phi \in [-\pi/2, \pi/2]$, and roll $\alpha \in SO(2)$ [S1]. The camera-to-world rotation is $R_i \in SO(3)$, and the origin location in the camera frame is $T_i \in \mathbb{R}^3$. In the ideal object-centered setup, $T_i$ is constant at $(0, 0, r)$ [S1].

**Equivalence Relations and Operations:**
MELON defines additive and multiplicative operations on $\Omega$ for $R \sim (\theta, \phi, \alpha)$, $\theta_0 \in SO(2)$, and $\lambda \in \mathbb{R}$ [S1]:
$$ \theta_0 + R \sim (\theta_0 + \theta, \phi, \alpha) $$
$$ \lambda R \sim (\lambda\theta, \phi, \alpha) $$
[ S1 ]

An equivalence relation $R_N$ is defined by a replication order $N \in \mathbb{N}^*$ [S1]:
$$ z R_N z' \iff \exists k \in \mathbb{Z}, z = z' + \frac{2k\pi}{N} $$
[ S1 ]

This relation creates an equivalence class $[z]_{R_N} = \{ z' \mid z' R_N z \}$, which contains $N$ distinct elements [S1]. The quotient space $\Omega/R_N$ is $N$ times smaller than $\Omega$ [S1].

**Algorithm Steps:**
1.  **Initialize:** Initialize the neural radiance field parameters $\psi$ and the encoder weights $\xi$.
2.  **Encode:** For each input image $I_i$, compute the latent pose estimate $z_i = h_\xi(I_i)$ [S1].
3.  **Generate Equivalence Class:** Compute the equivalence class $[z_i]_{R_N}$ for each latent pose [S1].
4.  **Render Synthetic Views:** For each $z \in [z_i]_{R_N}$, render a synthetic view $C(\cdot; f_\psi, z)$ using the current neural radiance field [S1]. Collect these into a set of synthetic observations $C_i^R(\psi, \xi) = \{ C(\cdot; f_\psi, z) \mid z \in [z_i]_{R_N} \}$ [S1].
5.  **Compute Modulo Loss:** Calculate the loss $L_R(\psi, \xi)$ by taking the minimum L2 distance between the input image $I_i$ and all synthetic views in $C_i^R(\psi, \xi)$ [S1]:
    $$ L_R(\psi, \xi) = \sum_{i=1}^n \min \{ \| C - I_i \|_2^2 \mid C \in C_i^R(\psi, \xi) \} $$
    [ S1 ]
    This loss is a lower bound of the standard L2 loss due to the reflexivity of $R_N$ [S1].
6.  **Optimize:** Update $\psi$ and $\xi$ using gradient descent to minimize $L_R(\psi, \xi)$ [S1].
7.  **Final Pose Estimation:** After training, for each image $I_i$, compute the final pose estimate $z_i$ by finding the element in the equivalence class that minimizes the reconstruction error [S1]:
    $$ z_i = \arg \min \{ \| C(\cdot; f_{\psi^*}, z) - I_i(\cdot) \|_2^2 \mid z \in [h_{\xi^*}(I_i)]_{R_N} \} $$
    [ S1 ]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Assume NeRF model and Encoder are defined classes
# NeRF: f_psi(x, d) -> (color, density)
# Encoder: h_xi(I) -> latent_vector (azimuth, elevation, roll)

def render_crop(nerf, latent_pose, intrinsics, image_shape):
    """
    Renders a crop of the scene given a latent pose.
    latent_pose: (azimuth, elevation, roll)
    Returns: rendered_image_tensor
    """
    # Convert latent pose to camera-to-world rotation matrix R
    # and translation T (fixed at (0,0,r) for object-centered)
    R = pose_to_rotation_matrix(latent_pose)
    T = torch.tensor([0, 0, r]) 
    
    # Generate ray origins and directions for the image grid
    # based on intrinsics, R, and T
    origins, directions = generate_rays(intrinsics, R, T, image_shape)
    
    # Query NeRF along rays
    # This is a simplified representation of volumetric rendering
    colors, densities = nerf_query(nerf, origins, directions)
    rendered_image = volumetric_render(colors, densities)
    
    return rendered_image

def modulo_loss_step(nerf, encoder, images, intrinsics, N, optimizer):
    """
    One step of MELON training.
    """
    optimizer.zero_grad()
    
    total_loss = 0.0
    
    for I_i in images:
        # 1. Encode image to latent pose
        z_i = encoder(I_i)
        
        # 2. Generate equivalence class
        # z_i is (theta, phi, alpha)
        # Equivalence relation: theta' = theta + 2*k*pi/N
        # We generate N samples for the azimuth dimension
        synthetic_views = []
        theta, phi, alpha = z_i
        
        for k in range(N):
            # Shift azimuth by 2*k*pi/N
            theta_shifted = theta + (2 * k * torch.pi / N)
            z_equiv = torch.stack([theta_shifted, phi, alpha])
            
            # 3. Render synthetic view
            # Note: In practice, this might be batched for efficiency
            C_equiv = render_crop(nerf, z_equiv, intrinsics, I_i.shape)
            synthetic_views.append(C_equiv)
            
        # 4. Compute minimum L2 distance
        # Compare I_i against all synthetic views in the class
        losses = []
        for C_equiv in synthetic_views:
            loss = torch.mean((C_equiv - I_i) ** 2)
            losses.append(loss)
            
        # The modulo loss is the minimum of these losses
        min_loss = torch.min(torch.stack(losses))
        total_loss += min_loss
        
    # 5. Backpropagate
    total_loss.backward()
    optimizer.step()
    
    return total_loss.item()

def final_pose_estimation(nerf, encoder, images, intrinsics, N):
    """
    Estimate final poses after training.
    """
    poses = []
    for I_i in images:
        z_i = encoder(I_i)
        theta, phi, alpha = z_i
        
        best_loss = float('inf')
        best_pose = None
        
        for k in range(N):
            theta_shifted = theta + (2 * k * torch.pi / N)
            z_equiv = torch.stack([theta_shifted, phi, alpha])
            
            C_equiv = render_crop(nerf, z_equiv, intrinsics, I_i.shape)
            loss = torch.mean((C_equiv - I_i) ** 2)
            
            if loss < best_loss:
                best_loss = loss
                best_pose = z_equiv
                
        poses.append(best_pose)
        
    return poses
```

**Parameters:**
*   **Replication Order ($N$):** An integer $N \in \mathbb{N}^*$ determining the number of replicas in the equivalence class [S1]. It controls the size of the quotient space [S1]. Typical values tested are $N \in \{1, 2, 3, 4\}$ [S1]. The method performs best when $N$ is a multiple of the dataset's symmetry order $K$ [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $10^{-4}$ [S1].
*   **Batch Size:** 256 crops [S1].
*   **Encoder Architecture (1D case):** Positional encoding and four fully-connected layers of size 64 with ReLU activation [S1].

**Edge Cases and Pitfalls:**
*   **Local Minima:** The low dimensionality of the $SO(3)$ latent space induces local minima, making pose estimation difficult for gradient descent [S1]. The modulo loss mitigates this by making the loss landscape more convex [S1].
*   **Unknown Translations:** MELON fails to jointly infer rotation $R_i$ and translation $T_i$ when both object-to-camera distances and in-plane camera translations are unknown (latent space $SE(3)$) [S1].
*   **Symmetry Matching:** If the replication order $N$ does not match the symmetry order $K$ of the object, the fractional covering of the region of attraction decreases, potentially leading to convergence issues [S1].

## Operating characteristics
*   **Time Complexity:** The compute time per gradient step increases linearly with the replication order $N$ [S1]. For $N=1$, it takes 23 ms/it; $N=2$, 34 ms/it; $N=3$, 43 ms/it; $N=4$, 52 ms/it [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On the 'lego' scene with 6 views, MELON achieves a PSNR of 35.4 and a mean angular error of 0.901°, outperforming GNeRF (PSNR 35.3, error 1.21°) and failing COLMAP [S1].
*   **Strengths:** Does not require CNN pre-training or pose initialization (ab-initio) [S1]. Does not require a discriminator, unlike adversarial approaches [S1]. Robust to small datasets (e.g., 6 views) [S1].
*   **Limitations:** Fails when translations are unknown [S1]. Performance depends on matching $N$ to object symmetry [S1].

## Background
MELON was introduced in the paper "MELON: NeRF with Unposed Images in SO(3)" [S1]. It addresses the challenges of inverse rendering with unposed images, specifically the issue of local minima in the latent pose space [S1]. It is compared against methods like GNeRF and SAMURAI [S1].

## Sources
[S1] MELON: NeRF with Unposed Images in SO(3), https://doi.org/10.1109/3dv62453.2024.00084, fetched 2026-10-30
