# MELON

Also called: Modulo-Equivalent Latent Optimization of NeRF, NeRF with Unposed Images in SO(3)

## Overview
MELON is a differentiable inverse rendering method that jointly optimizes a Neural Radiance Field (NeRF) and camera poses from a set of unposed images [S1]. It solves the problem of local minima in the low-dimensional $SO(3)$ latent space by introducing a "modulo loss" that maps the encoder's output to a quotient space, effectively replicating the latent variables to match the object's rotational symmetries [S1]. Unlike adversarial approaches such as GNeRF, MELON uses an encoder-decoder architecture with amortized inference, allowing it to perform ab-initio reconstruction without pose initialization or pre-trained CNNs [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ unposed images $\{I_i\}_{i=1}^n$. The method assumes known camera intrinsics and that cameras point towards the scene origin from a known distance $r$ [S1].
*   **Output:** A NeRF parameterized by $\psi$ (predicting density $\sigma_\psi$ and color $c_\psi$) and a set of camera poses $\{R_i\}_{i=1}^n$ in $SO(3)$ [S1].

**Data Structures:**
*   **NeRF ($f_\psi$):** A coordinate-based neural network predicting $(c_\psi, \sigma_\psi)$ for a given 3D point and viewing direction [S1].
*   **Encoder ($h_\xi$):** A neural network with weights $\xi$ that maps an input image $I_i$ to a latent pose variable $z_i$ in the quotient space [S1].
*   **Latent Space ($\Omega$):** Parameterized as $SO(3)$ using azimuth $\theta \in SO(2)$, elevation $\phi \in [-\pi/2, \pi/2]$, and roll $\alpha \in SO(2)$ [S1].
*   **Replication Order ($N$):** An integer $N \in \mathbb{N}^*$ defining the equivalence relation [S1].

**Mathematical Formulation:**
The image formation model is:
$$ I_i(p) = C(p ; f^*, R_i^*) $$
where $C$ is the rendering function, $f^* = (c^*, \sigma^*)$ is the unknown ground truth field, and $R_i^*$ is the unknown ground truth pose [S1].

Operations on $\Omega$ are defined for $R \sim (\theta, \phi, \alpha)$:
$$ \theta_0 + R \sim (\theta_0 + \theta, \phi, \alpha) $$
$$ \lambda R \sim (\lambda\theta, \phi, \alpha) $$
for $\theta_0 \in SO(2), \lambda \in \mathbb{R}$ [S1].

The equivalence relation $R_N$ for $N \in \mathbb{N}^*$ is:
$$ z R_N z' \iff \exists k \in \mathbb{Z}, z = z' + \frac{2k\pi}{N} $$
This creates a quotient space $\Omega/R_N$ which is $N$ times smaller than $\Omega$, replicating cameras along the azimuthal dimension [S1].

**Algorithm Steps:**
1.  **Initialize:** Initialize NeRF parameters $\psi$ and Encoder parameters $\xi$. Set replication order $N$ [S1].
2.  **Encode:** For each input image $I_i$, compute the latent estimate $z_i = h_\xi(I_i)$ [S1].
3.  **Generate Equivalence Class:** Compute the equivalence class $[z_i]_{R_N} = \{ z \mid z R_N z_i \}$ [S1].
4.  **Render Synthetic Views:** For each $z \in [z_i]_{R_N}$, render a synthetic image $C(\cdot ; f_\psi, z)$ using the current NeRF [S1]. Let $\mathcal{C}_i^R(\psi, \xi)$ be the set of these rendered images [S1].
5.  **Compute Modulo Loss:** Calculate the loss as the minimum L2 distance between the input image and any synthetic view in the equivalence class:
    $$ \mathcal{L}_R(\psi, \xi) = \sum_{i=1}^n \min \{ \| C - I_i \|_2^2 \mid C \in \mathcal{C}_i^R(\psi, \xi) \} $$
    This loss is a lower bound of the standard L2 loss due to the reflexivity of $R_N$ [S1].
6.  **Optimize:** Update $\psi$ and $\xi$ using gradient descent (e.g., Adam) to minimize $\mathcal{L}_R$ [S1].
7.  **Final Pose Estimation:** After convergence, for each $I_i$, the final pose estimate $z_i^*$ is:
    $$ z_i^* = \arg \min \{ \| C(\cdot ; f_\psi, z) - I_i(\cdot) \|_2^2 \mid z \in [h_{\xi^*}(I_i)]_{R_N} \} $$
    where $\xi^*$ is the optimal encoder weight [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Assume NeRF class and Encoder class are defined elsewhere
# NeRF: forward(ray_o, ray_d) -> (color, density)
# Encoder: forward(image) -> latent_vector (azimuth, elevation, roll)

def render_image(nerf, latent_pose, intrinsics, image_shape):
    """
    Renders an image given a latent pose and NeRF.
    latent_pose: (azimuth, elevation, roll)
    Returns: rendered_image tensor
    """
    # 1. Construct camera matrix from latent pose
    # R is rotation matrix derived from (azimuth, elevation, roll)
    # T is translation vector (0, 0, r) assuming known distance r
    R = pose_to_rotation(latent_pose)
    T = torch.tensor([0, 0, r], device=latent_pose.device)
    
    # 2. Generate rays for each pixel
    # u, v are pixel coordinates
    # ray_d = R^T * (u - W/2, -(v - H/2), -f) / f
    # ray_o = T
    rays_o, rays_d = generate_rays(intrinsics, R, T, image_shape)
    
    # 3. Volume rendering
    # Sample points along rays, query NeRF, composite
    color = volume_render(nerf, rays_o, rays_d)
    return color

def modulo_loss_step(nerf, encoder, images, N, optimizer):
    """
    One step of MELON optimization.
    images: batch of input images
    N: replication order
    """
    optimizer.zero_grad()
    
    total_loss = 0
    
    for i, I_i in enumerate(images):
        # Step 1: Encode image to latent
        z_i = encoder(I_i)
        
        # Step 2: Generate equivalence class
        # Replicate azimuth by 2*pi/N * k for k in 0..N-1
        # Keep elevation and roll fixed
        azimuth = z_i.azimuth
        elevation = z_i.elevation
        roll = z_i.roll
        
        synthetic_views = []
        for k in range(N):
            # Modulo arithmetic for azimuth
            new_azimuth = (azimuth + 2 * torch.pi * k / N) % (2 * torch.pi)
            latent_pose = torch.stack([new_azimuth, elevation, roll])
            
            # Step 3: Render
            rendered_view = render_image(nerf, latent_pose, intrinsics, I_i.shape)
            synthetic_views.append(rendered_view)
            
        # Step 4: Compute min L2 distance
        # L2 distance between I_i and each synthetic view
        dists = [torch.sum((I_i - view)**2) for view in synthetic_views]
        min_dist = min(dists)
        
        total_loss += min_dist
        
    # Step 5: Backpropagate
    total_loss.backward()
    optimizer.step()
    
    return total_loss.item()

def pose_estimation(nerf, encoder, images, N):
    """
    Final pose estimation after training.
    """
    poses = []
    for I_i in images:
        z_i = encoder(I_i)
        azimuth = z_i.azimuth
        elevation = z_i.elevation
        roll = z_i.roll
        
        best_pose = None
        best_dist = float('inf')
        
        for k in range(N):
            new_azimuth = (azimuth + 2 * torch.pi * k / N) % (2 * torch.pi)
            latent_pose = torch.stack([new_azimuth, elevation, roll])
            rendered_view = render_image(nerf, latent_pose, intrinsics, I_i.shape)
            
            dist = torch.sum((I_i - rendered_view)**2)
            if dist < best_dist:
                best_dist = dist
                best_pose = latent_pose
                
        poses.append(best_pose)
        
    return poses
```

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $10^{-4}$ [S1].
*   **Batch Size:** 256 crops (in 1D experiments) or full images [S1].
*   **Replication Order ($N$):** Integer $N \in \mathbb{N}^*$. Typical values tested are $N \in \{1, 2, 3, 4\}$ [S1]. The method performs best when $N$ is a multiple of the dataset's symmetry order $K$ [S1].
*   **Neural Network Architecture (1D):** Positional encoding and four fully-connected layers of size 64 with ReLU activation [S1].

**Edge Cases and Pitfalls:**
*   **Local Minima:** The low dimensionality of $SO(3)$ induces local minima. The modulo loss mitigates this by making the loss landscape more convex [S1].
*   **Unknown Translation:** If both object-to-camera distances and in-plane translations are unknown, the latent space becomes $SE(3)$. MELON fails to jointly infer rotation $R_i$ and translation $T_i$ in this setup [S1].
*   **Symmetry Matching:** If $N$ does not match the object's symmetry, convergence may be poor. The fractional covering $D_R(f^*)$ indicates the likelihood of convergence [S1].

## Operating characteristics
*   **Time Complexity:** Compute time per gradient step increases linearly with the replication order $N$. For $N=1$, it takes 23 ms/it; $N=2$ takes 34 ms/it; $N=3$ takes 43 ms/it; $N=4$ takes 52 ms/it [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On the 'lego' scene with 6 views, MELON achieves a PSNR of 35.4 and a mean angular error of 0.901°, outperforming GNeRF (PSNR 35.3, error 1.21°) and failing COLMAP [S1].
*   **Strengths:** Does not require CNN pre-training or pose initialization (ab-initio). Does not require a discriminator (unlike GNeRF). Robust to small datasets (works with as few as 6 views) [S1].
*   **Limitations:** Fails when translation is unknown ($SE(3)$). Requires known intrinsics and object-to-camera distance [S1].

## Background
MELON was introduced in the paper "MELON: NeRF with Unposed Images in SO(3)" [S1]. It addresses the limitations of adversarial methods like GNeRF, which require large datasets, complex training schedules, and discriminators [S1]. MELON uses amortized inference via an encoder to map images to poses, combined with a modulo loss to handle rotational symmetries [S1]. It has been evaluated on the 'NeRF-Synthetic' dataset, RGB-MELON (spheres with quasi-symmetric textures), and real-world datasets (Gold Cape, Ethiopian Head, Toytruck) [S1].

## Sources
[S1] MELON: NeRF with Unposed Images in SO(3), https://doi.org/10.1109/3dv62453.2024.00084, fetched 2026-10-30
