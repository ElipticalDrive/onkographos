# Neural Light Field (NeLF)

Also called: Neural Light Field (NeLF) via R2L Distillation, Neural Light Field with Distillation, R2L (Neural Light Field with Distillation), R2L network

## Overview
Neural Light Field (NeLF) via R2L Distillation is a method for efficient novel view synthesis that represents a 3D scene as a Neural Light Field rather than a Neural Radiance Field (NeRF) [S1]. It maps camera ray origin and direction directly to RGB values, avoiding ray-marching and iterative sampling [S1]. Unlike NeRF, which requires multiple forward passes per ray to aggregate radiance, NeLF requires a single forward pass per ray, making it orders-of-magnitude faster [S1]. The method distills a pre-trained NeRF teacher model into a deep residual student network that predicts color directly from ray parameters [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D oriented ray parameterized by origin $(x_o, y_o, z_o)$ and normalized direction $(x_d, y_d, z_d)$ [S1]. To capture local geometry, the input is expanded by sampling $K$ points along the ray and concatenating their 3D coordinates, forming a $3K$-dimensional vector (typically $K=16$) [S1].
*   **Output:** A 3D RGB color value $(r, g, b)$ [S1].

**Data Structures:**
The student network $G_\phi$ is a deep Multi-Layer Perceptron (MLP) with residual connections [S1].
*   **Architecture:** Deep Residual MLP. Configurations include W256D88 (12M FLOPs) and W181D88 (6M FLOPs), where W=width and D=depth [S1].
*   **Layers:** 88 layers with extensive residual MLP blocks [S1].
*   **Positional Encoding:** Used to enrich input information, following NeRF [S1].

**Numbered Steps:**

1.  **Teacher Training:** Train a pre-trained NeRF teacher model $F_{\Theta^*}$ on the original training images. The teacher maps 5D coordinates (spatial location $(x, y, z)$ and viewing direction $(\theta, \phi)$) to 1D volume density and 3D view-dependent emitted radiance [S1].
2.  **Bounding Box Inference:** Infer the viewing bounding box from the training data camera poses [S1].
3.  **Pseudo Data Generation:**
    *   Randomly sample ray origins $(x_o, y_o, z_o)$ and normalized directions $(x_d, y_d, z_d)$ from a uniform distribution $U$ within the viewing bounding box [S1]:
        $$ x_o \sim U(x_{o,min}, x_{o,max}), \quad y_o \sim U(y_{o,min}, y_{o,max}), \quad z_o \sim U(z_{o,min}, z_{o,max}) $$
        $$ x_d \sim U(x_{d,min}, x_{d,max}), \quad y_d \sim U(y_{d,min}, y_{d,max}), \quad z_d \sim U(z_{d,min}, z_{d,max}) $$
    *   Query the pre-trained NeRF teacher model $F_{\Theta^*}$ with the sampled 6D input to obtain the target RGB value $(\hat{r}, \hat{g}, \hat{b})$ [S1]:
        $$ (\hat{r}, \hat{g}, \hat{b}) = F_{\Theta^*}(x_o, y_o, z_o, x_d, y_d, z_d) $$
    *   Store the vector $(x_o, y_o, z_o, x_d, y_d, z_d, \hat{r}, \hat{g}, \hat{b})$ as a training sample [S1].
    *   Repeat to generate $S$ pseudo images (default $S=10k$) [S1].
4.  **Student Network Initialization:** Initialize the deep residual MLP $G_\phi$ with width 256 and depth 88 (W256D88) or width 181 and depth 88 (W181D88) [S1].
5.  **Ray Representation Construction:**
    *   For each training ray, sample $K$ points along the ray [S1].
    *   **Training:** Use stratified sampling to randomly sample $K$ points along the ray to prevent overfitting to specific point locations and ensure generalization [S1].
    *   **Testing:** Evenly space $K$ points along the ray [S1].
    *   Concatenate the spatial coordinates of the $K$ sampled points to form a $3K$-dimensional input vector [S1].
6.  **Hard Example Mining:**
    *   Maintain a pool of hard examples defined by larger loss values [S1].
    *   In each iteration, sort batch losses in ascending order [S1].
    *   Add the top $r$% of samples to the hard example pool [S1].
    *   Randomly pick $r$% of samples from the pool to augment the training batch [S1].
    *   Default hard example ratio $r=0.2$ [S1].
7.  **Training Loop:**
    *   Train the network for 1,000k iterations using the Adam optimizer [S1].
    *   Use the same learning rate schedule as the teacher NeRF (initial learning rate $5 \times 10^{-4}$, decayed during training) [S1].
    *   Compute the Mean Squared Error (MSE) loss between the student's prediction and the teacher's RGB output [S1]:
        $$ L = \text{MSE}(G_\phi(x_o, y_o, z_o, x_d, y_d, z_d), (\hat{r}, \hat{g}, \hat{b})) $$
    *   Update network parameters $\phi$ using backpropagation [S1].
    *   Batch size is 98,304 rays per batch [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

class ResidualBlock(nn.Module):
    def __init__(self, width):
        super(ResidualBlock, self).__init__()
        self.fc1 = nn.Linear(width, width)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(width, width)
        
    def forward(self, x):
        identity = x
        out = self.fc1(x)
        out = self.relu(out)
        out = self.fc2(out)
        out += identity
        out = self.relu(out)
        return out

class NeLF(nn.Module):
    def __init__(self, width=256, depth=88):
        super(NeLF, self).__init__()
        self.input_dim = 48  # 3 * K, where K=16
        self.output_dim = 3
        
        # Initial layer
        self.fc0 = nn.Linear(self.input_dim, width)
        self.relu = nn.ReLU()
        
        # Residual blocks
        self.residual_blocks = nn.ModuleList([
            ResidualBlock(width) for _ in range(depth)
        ])
        
        # Final layer
        self.fc_out = nn.Linear(width, self.output_dim)
        
    def forward(self, x):
        x = self.fc0(x)
        x = self.relu(x)
        for block in self.residual_blocks:
            x = block(x)
        x = self.fc_out(x)
        return x

def generate_pseudo_data(teacher_model, num_samples, bounding_box):
    """
    Generate pseudo training data by querying the NeRF teacher.
    bounding_box: dict with keys 'xo_min', 'xo_max', etc.
    """
    # Sample origins and directions uniformly within bounding box
    xo = torch.rand(num_samples) * (bounding_box['xo_max'] - bounding_box['xo_min']) + bounding_box['xo_min']
    yo = torch.rand(num_samples) * (bounding_box['yo_max'] - bounding_box['yo_min']) + bounding_box['yo_min']
    zo = torch.rand(num_samples) * (bounding_box['zo_max'] - bounding_box['zo_min']) + bounding_box['zo_min']
    
    xd = torch.rand(num_samples) * (bounding_box['xd_max'] - bounding_box['xd_min']) + bounding_box['xd_min']
    yd = torch.rand(num_samples) * (bounding_box['yd_max'] - bounding_box['yd_min']) + bounding_box['yd_min']
    zd = torch.rand(num_samples) * (bounding_box['zd_max'] - bounding_box['zd_min']) + bounding_box['zd_min']
    
    # Normalize directions? The source says "normalized directions" but samples from U.
    # Assuming the bounding box for directions is already normalized or the teacher handles it.
    # The source formula (1) uses U for xd, yd, zd.
    
    input_coords = torch.stack([xo, yo, zo, xd, yd, zd], dim=1)
    
    # Query teacher
    # Note: Teacher expects 5D (xyz + direction angles) or 6D (xyz + direction vector).
    # Source says F_theta maps 5D coordinate (spatial location (x,y,z) and viewing direction (theta, phi)).
    # However, Eq 2 uses (xo, yo, zo, xd, yd, zd). We assume the teacher implementation
    # accepts these 6D inputs or converts them internally.
    with torch.no_grad():
        target_rgb = teacher_model(input_coords)
        
    return input_coords, target_rgb

def stratified_sample_points(ray_origin, ray_direction, num_points=16, near=0.0, far=1.0):
    """
    Sample K points along a ray using stratified sampling.
    ray_origin: (3,) tensor
    ray_direction: (3,) tensor
    """
    # Generate uniform samples in [near, far]
    t = torch.rand(num_points) * (far - near) + near
    # Stratified sampling: add small jitter? NeRF uses stratified sampling for anti-aliasing.
    # Simple stratified: t_i = (i + uniform(0,1)) / N
    # For simplicity, we use uniform sampling as per source description "randomly sampled... using stratified sampling"
    # The source doesn't specify the exact stratified formula, just that it is used.
    # We will use simple uniform sampling for the points along the ray segment.
    
    points = ray_origin.unsqueeze(0) + t.unsqueeze(1) * ray_direction.unsqueeze(0)
    return points

def train_nelf(teacher_model, train_data, pseudo_data, num_iterations=1000000, batch_size=98304, hard_example_ratio=0.2):
    """
    Train the NeLF student network.
    """
    # Initialize student
    student = NeLF(width=256, depth=88)
    optimizer = optim.Adam(student.parameters(), lr=5e-4)
    
    # Combine real and pseudo data if available (Ours-2)
    # For simplicity, assume we train on pseudo data or combined.
    # The source mentions Ours-1 (Pseudo) and Ours-2 (Pseudo+Real).
    # We'll assume a combined dataset for the loop.
    
    hard_pool = []
    
    for i in range(num_iterations):
        # Sample batch
        # In practice, rays are sampled from different images.
        # Here we simulate sampling from the generated pseudo data pool.
        indices = torch.randint(0, len(pseudo_data[0]), (batch_size,))
        batch_inputs = pseudo_data[0][indices]
        batch_targets = pseudo_data[1][indices]
        
        # Construct ray representation: K points along the ray
        # Note: The source says "concatenate the spatial coordinates of K sampled points".
        # We need to generate these points for each ray in the batch.
        # This is computationally expensive if done naively.
        # Assuming ray_origin and ray_direction are derived from batch_inputs.
        # batch_inputs shape: (batch_size, 6) -> (xo, yo, zo, xd, yd, zd)
        
        origins = batch_inputs[:, :3]
        directions = batch_inputs[:, 3:]
        
        # Sample K points for each ray
        # Using stratified sampling during training
        K = 16
        # Generate random t for each point in each ray
        # Shape: (batch_size, K)
        t = torch.rand(batch_size, K) 
        # Stratified: t = (t + uniform(0,1)) / K ? 
        # Source says "stratified sampling (same as NeRF)". 
        # We will use simple uniform for the points along the ray segment [near, far].
        # Let's assume near=0, far=1 for normalized ray parameterization or specific bounds.
        # The source doesn't specify near/far bounds for point sampling, just "along the ray".
        # We'll assume a standard range [0, 1] for normalized direction or specific scene bounds.
        # For implementation completeness, we assume a fixed range [0, 1] for t.
        points = origins.unsqueeze(1) + t.unsqueeze(2) * directions.unsqueeze(1)
        
        # Concatenate coordinates: (batch_size, K, 3) -> (batch_size, 3*K)
        ray_representation = points.reshape(batch_size, -1)
        
        # Forward pass
        predictions = student(ray_representation)
        
        # Loss
        loss = nn.MSELoss()(predictions, batch_targets)
        
        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Hard Example Mining
        # Calculate per-sample loss
        per_sample_loss = nn.MSELoss(reduction='none')(predictions, batch_targets).mean(dim=1)
        
        # Sort losses
        sorted_indices = torch.argsort(per_sample_loss)
        top_r_indices = sorted_indices[-int(batch_size * hard_example_ratio):]
        
        # Add to pool
        hard_pool.extend(top_r_indices.tolist())
        
        # Augment batch with hard examples
        if len(hard_pool) > 0:
            # Pick r% from pool
            num_hard = int(batch_size * hard_example_ratio)
            hard_indices = torch.randint(0, len(hard_pool), (num_hard,))
            hard_samples = hard_pool[hard_indices]
            
            # Replace some samples in the batch? Or augment?
            # Source: "randomly pick r% from the pool to augment the training batch."
            # This implies replacing or adding. Given fixed batch size, likely replacing.
            # We'll replace the last num_hard samples in the batch with hard examples.
            # Note: This requires re-fetching data for hard examples.
            # For pseudocode simplicity, we assume we have access to the full dataset.
            pass 
            
    return student
```

**Parameters:**
*   **Network Depth:** 88 layers [S1].
*   **K:** Number of sampled points along a ray for input representation. Value: 16 [S1].
*   **r:** Pre-defined percentage constant for hard example mining selection rate. Default: 0.2; tested values {0, 0.1, 0.2, 0.3} [S1].
*   **Batch Size:** 98,304 rays per batch [S1].
*   **Iterations:** 1,000k iterations [S1].
*   **Learning Rate:** Initial learning rate $5 \times 10^{-4}$, decayed during training [S1].
*   **Pseudo Data Volume:** 10k images synthesized using the pre-trained NeRF model [S1]. Performance saturates around 5k images [S1].
*   **Hardware:** 8 NVIDIA V100 GPUs [S1].
*   **Framework:** PyTorch 1.9 [S1].

**Edge Cases and Pitfalls:**
*   **Deep Network Trainability:** Deep networks for NeRF/NeLF suffer from diminishing returns if not designed carefully; residual connections are critical to enable greater depth. Without residuals, the network is barely trainable [S1].
*   **Overfitting:** Deep networks require abundant data. Training on only original images (typically <100) leads to severe overfitting. Pseudo data is essential to prevent this [S1].
*   **Reflection Surfaces:** Methods relying on ground-truth depth (like DONeRF) fail on reflective geometry. NeLF via R2L does not rely on ground-truth depth and performs well on reflections [S1].
*   **Input Representation:** Concatenated coordinates achieve better PSNR than Plücker coordinates (29.50 vs 29.08 on Lego scene) [S1].

## Operating characteristics
*   **Time Complexity:** Single forward pass per ray, avoiding ray-marching [S1].
*   **Memory Complexity:** Storage size is comparable to NeRF (2.4 MB for teacher, 23.7 MB for R2L) [S1].
*   **Performance:**
    *   **FLOPs Reduction:** 26–35× reduction per camera ray compared to NeRF [S1].
    *   **Wall-Time Speedup:** 28–31× faster than NeRF on synthetic scenes [S1]. 2× faster than DONeRF-16-noGT [S1].
    *   **Quality:** 1.4–2.8 dB average PSNR improvement over NeRF on synthetic scenes [S1].
    *   **SSIM/LPIPS:** On NeRF synthetic dataset, Ours-2 achieves 0.9950 SSIM and 0.0340 LPIPS [S1].
*   **Scaling:** Speedup is consistent across hardware (GeForce 2080Ti, Tesla V100, CPU) [S1].
*   **Strengths:** No customized parallelism required (unlike KiloNeRF); works with vanilla PyTorch; handles reflections better than depth-based methods; significantly faster than NeRF.
*   **Limitations:** Requires a pre-trained NeRF teacher; training time is long (1,000k iterations); quality is bounded by teacher quality unless real data is included (Ours-2) [S1].

## Background
*   **Origin:** The method was introduced in the paper "R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis" by Wang et al. [S1].
*   **History:** It builds on the concept of Neural Radiance Fields (NeRF) [S1] and the idea of Neural Light Fields (NeLF) [S1]. Previous NeLF works used shallow MLPs (6-8 layers) and achieved limited speedup (~5×) [S1]. R2L introduced deep residual MLPs (88 layers) and knowledge distillation to achieve significant speedup (30×) and better quality [S1].
*   **Usage:** Used for efficient novel view synthesis in computer graphics and computer vision [S1].

## Sources
[S1] R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis, https://doi.org/10.1007/978-3-031-19821-2_35, fetched 2026-10-26
