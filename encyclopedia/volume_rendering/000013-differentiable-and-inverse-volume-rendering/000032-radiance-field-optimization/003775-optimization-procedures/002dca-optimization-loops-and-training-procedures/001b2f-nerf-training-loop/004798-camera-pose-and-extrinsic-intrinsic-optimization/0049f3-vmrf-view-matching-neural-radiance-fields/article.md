# VMRF (View Matching Neural Radiance Fields)

Also called: View Matching NeRF

## Overview
VMRF is a method for novel view synthesis that learns Neural Radiance Field (NeRF) representations without prior knowledge of camera poses or hand-crafted camera pose distributions [S1]. It addresses the failure of classical Structure-from-Motion (SfM) in low-texture scenes and the instability of absolute pose estimation methods (like GNeRF) when out-of-dataset views are present [S1]. Unlike GNeRF, which estimates absolute poses directly from images and requires accurate pose distributions for training its inversion network, VMRF estimates **relative pose transformations** between a rendered image (with a random pose) and a real image using a view matching scheme based on unbalanced optimal transport (UOT) [S1]. This approach makes VMRF robust to out-of-dataset views and removes the dependency on prior pose distribution knowledge [S1].

## How it works
VMRF operates in an iterative loop, building on the differentiable rendering principles of [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md). It initializes a coarse NeRF with random poses, estimates real poses via view matching and pose calibration, and jointly optimizes the NeRF and poses.

### Inputs and Outputs
*   **Inputs:** A set of $n$ RGB images $I = \{I_1, I_2, ..., I_n\}$ of a static scene. No camera poses are provided.
*   **Outputs:** Optimized NeRF parameters $\Theta$ and estimated camera poses $\hat{\Phi} = \{\hat{\phi}_1, ..., \hat{\phi}_n\}$.

### Data Structures
1.  **NeRF Model ($F_\Theta$):** A multi-layer perceptron (MLP) parameterizing the 5D scene function $F_\Theta: (x, d) \to (c, \sigma)$, where $x \in \mathbb{R}^3$ is a 3D point, $d \in S^2$ is a viewing direction, $c \in \mathbb{R}^3$ is RGB color, and $\sigma \in \mathbb{R}$ is volume density [S1].
2.  **Camera Pose ($\phi$):** Represented as a $4 \times 4$ transformation matrix containing rotation and translation. Rotation is encoded as a continuous 6D embedding, and position as a 3D Euclidean embedding [S1].
3.  **Feature Sets:**
    *   Source feature set $\mathbf{f}_s \in \mathbb{R}^{l \times d}$ extracted from the rendered image.
    *   Target feature set $\mathbf{f}_t \in \mathbb{R}^{l \times d}$ extracted from the real image.
    *   Parameters: $l = 64$ (number of feature vectors), $d = 256$ (dimension of feature vectors) [S1].
4.  **Transport Plan ($T$):** A matrix $T \in \mathbb{R}^{l \times l}$ where $T_{ij}$ represents the amount of mass moved from source feature $i$ to target feature $j$ [S1].
5.  **Relative Transformation Predictor ($RTP$):** A Vision Transformer (ViT) modified to output 6 values: $[t_x, t_y, t_z, \theta_x, \theta_y, \theta_z]$ representing relative translation and rotation [S1].

### Numbered Steps

1.  **Initialization:**
    Initialize a set of random camera poses $\Phi' = \{\phi'_i, i \in [1, n]\}$ for the $n$ input images [S1]. Initialize the NeRF parameters $\Theta$ and a discriminator $D$ [S1].

2.  **Coarse NeRF Training (Adversarial):**
    Train the coarse NeRF $F$ and discriminator $D$ using adversarial loss to ensure generated images have a similar distribution to real images. This stabilizes the initial random poses [S1].
    $$ \mathcal{L}_{adv}(F, D) = \mathbb{E}_{I \sim P_{data}} [\log(D(I))] + \mathbb{E}_{F(\Phi') \sim P_g} [\log(1 - D(F(\Phi')))] $$
    where $P_{data}$ is the distribution of real images and $P_g$ is the distribution of generated images [S1].

3.  **Rendering:**
    For each real image $I_i$, render a corresponding image using the coarse NeRF $F$ and the current random pose $\phi'_i$ [S1].

4.  **Feature Extraction:**
    Extract feature sets from the rendered image (source) and the real image (target) using two pre-trained VGG-19 networks [S1].
    *   Source: $\mathbf{f}_s \in \mathbb{R}^{l \times d}$
    *   Target: $\mathbf{f}_t \in \mathbb{R}^{l \times d}$

5.  **Cost Matrix Computation:**
    Compute the cost matrix $M \in \mathbb{R}^{l \times l}$ based on feature similarities. Each element $M_{ij}$ denotes the moving cost from source feature $i$ to target feature $j$ [S1].
    $$ M = 1 - \frac{\mathbf{f}_s \cdot \mathbf{f}_t^T}{\|\mathbf{f}_s\| \|\mathbf{f}_t\|} $$
    where $\cdot$ denotes matrix multiplication, and $\|\cdot\|$ denotes the Euclidean norm [S1].

6.  **Unbalanced Optimal Transport (UOT):**
    Compute the optimal transport plan $T^*$ by solving the entropic regularized UOT problem. This handles non-surjective matching caused by occlusions (inconsistent mass between views) [S1].
    $$ T^* = \arg \min_T \left[ \sum_{i,j=1}^{l} T_{ij} M_{ij} - \eta H(T) + \epsilon KL(T \mathbf{1}_l \| \mu_s) + \epsilon KL(T^T \mathbf{1}_l \| \mu_t) \right] $$
    Where:
    *   $\mu_s = \sum_{i=1}^l p_i^s \delta(f_i^s)$ and $\mu_t = \sum_{j=1}^l p_j^t \delta(f_j^t)$ are discrete source and target distributions with probability masses $p_i^s, p_j^t$ [S1].
    *   $H(T) = -\sum_{i,j=1}^l T_{ij} \log(T_{ij} - 1)$ is the entropic regularization term [S1].
    *   $\eta > 0$ is the entropy regularization parameter (default 0.005) [S1].
    *   $\epsilon > 0$ is the KL divergence regularization parameter [S1].
    *   $KL(\cdot \| \cdot)$ is the Kullback-Leibler divergence [S1].
    *   The solution is computed using the Sinkhorn algorithm on the Fenchel-Legendre dual form [S1].

7.  **Relative Pose Prediction:**
    Predict the relative transformation $[t_x, t_y, t_z, \theta_x, \theta_y, \theta_z]$ from the transport plan $T^*$ using the Relative Transformation Predictor $RTP$ [S1].
    $$ [t_x, t_y, t_z, \theta_x, \theta_y, \theta_z] = RTP(T^*) $$

8.  **Pose Calibration:**
    Construct the relative rotation matrix $\Delta R$ and translation vector $\Delta t$ from the predicted values [S1].
    $$ \Delta R = R_x(\theta_x) R_y(\theta_y) R_z(\theta_z) $$
    $$ \Delta R = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos\theta_x & \sin\theta_x \\ 0 & -\sin\theta_x & \cos\theta_x \end{bmatrix} \begin{bmatrix} \cos\theta_y & 0 & -\sin\theta_y \\ 0 & 1 & 0 \\ \sin\theta_y & 0 & \cos\theta_y \end{bmatrix} \begin{bmatrix} \cos\theta_z & \sin\theta_z & 0 \\ -\sin\theta_z & \cos\theta_z & 0 \\ 0 & 0 & 1 \end{bmatrix} $$
    $$ \Delta t = [t_x, t_y, t_z]^T $$
    Calibrate the random pose $\phi'_i$ to estimate the real pose $\hat{\phi}_i$ [S1]:
    $$ \hat{\phi}_i = \begin{bmatrix} \Delta R & \Delta t \\ 0 & 1 \end{bmatrix} \phi'_i $$

9.  **NeRF and Pose Optimization:**
    Optimize NeRF parameters $\Theta$ and estimated poses $\hat{\Phi}$ using photometric loss and calibration loss [S1].
    *   **Photometric Loss:**
        $$ \mathcal{L}_P(F, \hat{\Phi}) = \frac{1}{n} \sum_{i=1}^n \| I_i - F(\hat{\phi}_i) \|^2 $$
    *   **Calibration Loss:** (Used to train $RTP$ unsupervised by comparing predicted relative poses between random pose pairs) [S1]:
        $$ \mathcal{L}_{ca}(RTP) = \mathbb{E} \| F_{ca}(RTP(T_{ab}^*), \phi'_a) - \phi'_b \|^2 $$
        where $(\phi'_a, \phi'_b)$ is a pair of randomly initialized poses and $T_{ab}^*$ is the transport plan between their renderings [S1].

10. **Volume Rendering Formula:**
    The rendered color $c_r$ for a ray $r$ is computed as [S1]:
    $$ c_r = \sum_{i=1}^{N} T_{ri} \alpha_{ri} c_{ri} $$
    $$ T_{ri} = \prod_{j=1}^{i-1} (1 - \alpha_{rj}) $$
    $$ \alpha_{ri} = 1 - \exp(-\sigma_{ri} \delta_{ri}) $$
    Where:
    *   $N$ is the number of random samples along ray $r$ [S1].
    *   $c_{ri}, \sigma_{ri}$ are the color and volume density of the $i$-th sample [S1].
    *   $\delta_{ri} = x_{ri+1} - x_{ri}$ is the distance between adjacent samples [S1].
    *   $T_{ri}$ is the transmittance and $\alpha_{ri}$ is the alpha value [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim
from scipy.optimize import linear_sum_assignment # For Sinkhorn approximation or initialization if needed, 
# but VMRF uses differentiable Sinkhorn. Here we assume a differentiable Sinkhorn layer.

def sinkhorn_unbalanced(cost_matrix, mu_s, mu_t, epsilon, eta, iterations=10):
    """
    Computes the optimal transport plan T using entropic regularized UOT.
    cost_matrix: (l, l)
    mu_s: (l,) source masses
    mu_t: (l,) target masses
    epsilon: KL regularization strength
    eta: Entropic regularization strength
    """
    # Initialize potentials
    u = torch.zeros_like(mu_s)
    v = torch.zeros_like(mu_t)
    
    # Kernel K
    # Note: The exact dual form implementation depends on the specific library.
    # Standard Sinkhorn for UOT with KL penalties involves iterative scaling.
    # For the purpose of this article, we assume a differentiable Sinkhorn function exists.
    # The formula minimizes: sum(T*M) - eta*H(T) + epsilon*KL(T*1 || mu_s) + epsilon*KL(T^T*1 || mu_t)
    
    # Approximation using standard Sinkhorn with modified costs or potentials
    # This is a placeholder for the specific differentiable UOT implementation described in [S1]
    # which uses the Fenchel-Legendre dual.
    
    # Simplified iterative update for demonstration of the logic:
    # In practice, this is implemented as a custom autograd function.
    
    # K = exp((cost_matrix - eta * log(T)) / epsilon) ... 
    # The paper states: T* = arg min [ sum T_ij M_ij - eta H(T) + epsilon KL(...) ]
    # This is solved via Sinkhorn on the dual variables.
    
    # Returning a dummy plan for structure completeness.
    # Real implementation requires the specific UOT Sinkhorn kernel from the authors' code.
    return torch.ones_like(cost_matrix) / cost_matrix.numel()

def extract_features(image, vgg_model, l=64, d=256):
    """
    Extracts l feature vectors of dimension d from an image using VGG-19.
    """
    # Implementation details of VGG-19 feature extraction are assumed standard.
    # Returns tensor of shape (l, d)
    pass

def compute_cost_matrix(f_s, f_t):
    """
    Computes cost matrix M = 1 - (f_s . f_t^T) / (||f_s|| ||f_t||)
    """
    # f_s: (l, d), f_t: (l, d)
    # Normalize features
    f_s_norm = f_s / torch.norm(f_s, dim=1, keepdim=True)
    f_t_norm = f_t / torch.norm(f_t, dim=1, keepdim=True)
    
    # Cosine similarity matrix
    sim = torch.matmul(f_s_norm, f_t_norm.T)
    
    # Cost matrix
    M = 1.0 - sim
    return M

def predict_relative_pose(transport_plan, rtp_model):
    """
    Predicts relative translation and rotation from transport plan.
    """
    # RTP is a Vision Transformer
    pred = rtp_model(transport_plan)
    # pred shape: (6,) -> [tx, ty, tz, theta_x, theta_y, theta_z]
    return pred

def construct_relative_transform(tx, ty, tz, theta_x, theta_y, theta_z):
    """
    Constructs Delta R and Delta t from predicted values.
    """
    cos_x = torch.cos(theta_x)
    sin_x = torch.sin(theta_x)
    cos_y = torch.cos(theta_y)
    sin_y = torch.sin(theta_y)
    cos_z = torch.cos(theta_z)
    sin_z = torch.sin(theta_z)
    
    Rx = torch.tensor([[1, 0, 0],
                       [0, cos_x, sin_x],
                       [0, -sin_x, cos_x]])
    Ry = torch.tensor([[cos_y, 0, -sin_y],
                       [0, 1, 0],
                       [sin_y, 0, cos_y]])
    Rz = torch.tensor([[cos_z, sin_z, 0],
                       [-sin_z, cos_z, 0],
                       [0, 0, 1]])
    
    Delta_R = torch.matmul(torch.matmul(Rx, Ry), Rz)
    Delta_t = torch.tensor([tx, ty, tz])
    
    return Delta_R, Delta_t

def render_nerf(nerf_model, ray_origins, ray_directions, N=128):
    """
    Renders an image from NeRF using volume rendering.
    """
    # Sample points along rays
    # t_vals = ...
    # xyz = ray_origins + ray_directions * t_vals.unsqueeze(-1)
    # colors, densities = nerf_model(xyz, ray_directions)
    # alpha = 1 - torch.exp(-densities * deltas)
    # T = torch.cumprod(torch.ones_like(alpha) - alpha, dim=-1)
    # T = torch.cat([torch.ones_like(T[..., :1]), T[..., :-1]], dim=-1)
    # weights = T * alpha
    # color = torch.sum(weights.unsqueeze(-1) * colors, dim=-2)
    # return color
    pass

def train_vmrf(images, nerf_model, discriminator, rtp_model, optimizer_nerf, optimizer_pose, optimizer_rtp, 
               l=64, d=256, eta=0.005, epsilon=1e-3, num_epochs=1000):
    """
    Main training loop for VMRF.
    """
    n = len(images)
    # Initialize random poses
    poses = [init_random_pose() for _ in range(n)]
    
    for epoch in range(num_epochs):
        total_loss = 0
        
        for i in range(n):
            I_real = images[i]
            phi_prime = poses[i]
            
            # 1. Render image with random pose
            I_render = render_nerf(nerf_model, phi_prime)
            
            # 2. Extract features
            f_s = extract_features(I_render, vgg_model, l, d)
            f_t = extract_features(I_real, vgg_model, l, d)
            
            # 3. Compute cost matrix
            M = compute_cost_matrix(f_s, f_t)
            
            # 4. Compute Transport Plan (UOT)
            # Assuming mu_s and mu_t are uniform or computed from features
            mu_s = torch.ones(l) / l
            mu_t = torch.ones(l) / l
            T_star = sinkhorn_unbalanced(M, mu_s, mu_t, epsilon, eta)
            
            # 5. Predict relative pose
            pred_rel = predict_relative_pose(T_star, rtp_model)
            tx, ty, tz, th_x, th_y, th_z = pred_rel
            
            # 6. Construct relative transform and calibrate pose
            Delta_R, Delta_t = construct_relative_transform(tx, ty, tz, th_x, th_y, th_z)
            
            # Apply relative transform to random pose to get estimated pose
            # phi_hat = [Delta_R, Delta_t; 0, 1] @ phi_prime
            phi_hat = apply_transform(phi_prime, Delta_R, Delta_t)
            
            # 7. Compute Photometric Loss
            I_render_calibrated = render_nerf(nerf_model, phi_hat)
            loss_p = torch.mean((I_real - I_render_calibrated) ** 2)
            
            # 8. Compute Calibration Loss (for RTP)
            # Sample another random pose phi_b_prime
            phi_b_prime = init_random_pose()
            I_b_render = render_nerf(nerf_model, phi_b_prime)
            f_s_b = extract_features(I_b_render, vgg_model, l, d)
            T_star_b = sinkhorn_unbalanced(compute_cost_matrix(f_s_b, f_t), mu_s, mu_t, epsilon, eta)
            pred_rel_b = predict_relative_pose(T_star_b, rtp_model)
            
            # Calibration loss compares predicted relative pose to actual relative pose between random poses
            # L_ca = || F_ca(RTP(T*), phi_a') - phi_b' ||^2
            # Note: The exact implementation of F_ca in the paper uses the predicted relative pose 
            # to transform phi_a' and compares it to phi_b'.
            phi_a_pred = apply_transform(phi_prime, *pred_rel)
            loss_ca = torch.mean((phi_a_pred - phi_b_prime) ** 2)
            
            # 9. Adversarial Loss (for coarse NeRF/Discriminator)
            # L_adv = E[log(D(I))] + E[log(1 - D(F(phi')))]
            # This is typically computed over a batch of images/poses
            loss_adv = compute_adversarial_loss(nerf_model, discriminator, poses, images)
            
            # 10. Update
            loss = loss_p + loss_ca + loss_adv
            loss.backward()
            optimizer_nerf.step()
            optimizer_pose.step() # Update poses if optimized directly, or via phi_hat
            optimizer_rtp.step()
            
            total_loss += loss.item()
            
        print(f"Epoch {epoch}, Loss: {total_loss}")
```

### Parameters
*   $N$: Number of random samples along a camera ray for volume rendering. Not given in the sources.
*   $\Phi'$: Set of initially randomized camera poses.
*   $\Theta$: Network parameters of the MLP.
*   $l$: Number of feature vectors in each feature set. Value: 64 [S1].
*   $d$: Dimension of feature vectors. Value: 256 [S1].
*   $\eta$: Regularization parameter controlling the smoothness of UOT (entropy). Value: 0.005 [S1].
*   $\epsilon$: Regularization parameter for KL divergence in UOT. Not given in the sources.

### Edge Cases and Pitfalls
*   **Low Texture/Repetitive Patterns:** Classical SfM fails here due to few keypoints. VMRF avoids this by not relying on SfM [S1].
*   **Out-of-Dataset Views:** GNeRF's inversion network is disturbed by views beyond the training dataset if the accurate pose distribution is unavailable. VMRF mitigates this by using relative pose estimation via view matching, which is robust to such views [S1].
*   **Occlusion:** Causes inconsistent total mass between source and target feature distributions. Standard Optimal Transport fails by matching outliers. VMRF uses Unbalanced OT (UOT) with KL divergence penalties to handle this [S1].

## Operating characteristics
*   **Time Complexity:** The dominant cost is the Optimal Transport computation (Sinkhorn algorithm) for each image pair in each iteration, which is $O(l^2 \cdot k)$ where $l$ is the number of features and $k$ is the number of Sinkhorn iterations. Feature extraction via VGG-19 is also significant.
*   **Memory Complexity:** Requires storing the NeRF model, the discriminator, the RTP model, and the feature sets ($O(l \cdot d)$ per image).
*   **Performance:** VMRF outperforms state-of-the-art methods like GNeRF qualitatively and quantitatively on Synthetic-NeRF and DTU datasets [S1].
*   **Strengths:** Does not require prior camera poses or hand-crafted pose distributions. Robust to out-of-dataset views and low-texture scenes.
*   **Limitations:** Relies on the quality of the VGG-19 features and the effectiveness of the UOT matching. The iterative nature of joint optimization can be slow to converge compared to methods with good initial poses.

## Background
VMRF was proposed by Jiahui Zhang, Fangneng Zhan, Rongliang Wu, Yingchen Yu, Wenqing Zhang, Bai Song, Xiaoqin Zhang, and Shijian Lu in 2022 [S1]. It was published in the ACM Conference proceedings (DOI: 10.1145/3503161.3548078) [S1]. The method was developed to address the limitations of pose-free NeRF methods like GNeRF, which require accurate camera pose distributions for training their inversion networks [S1]. VMRF introduces the concept of view matching via unbalanced optimal transport to estimate relative poses, enabling effective NeRF training without prior pose knowledge [S1]. It is used for novel view synthesis on Synthetic-NeRF and DTU datasets [S1].

## Sources
[S1] VMRF: View Matching Neural Radiance Fields, https://doi.org/10.1145/3503161.3548078, fetched 2026-10-29
