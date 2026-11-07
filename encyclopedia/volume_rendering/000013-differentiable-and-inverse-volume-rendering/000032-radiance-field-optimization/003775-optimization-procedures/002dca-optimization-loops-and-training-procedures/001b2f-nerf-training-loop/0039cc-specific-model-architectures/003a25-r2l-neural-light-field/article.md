# R2L (Neural Light Field)

Also called: Neural Light Field, R2L network

## Overview
R2L (R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis) is a method for efficient novel view synthesis that distills a pre-trained NeRF teacher model into a deep Neural Light Field (NeLF) student network [S1]. It solves the high computational cost of standard NeRF by replacing the shallow, direction-dependent MLP with a much deeper, residual-heavy architecture that operates on aggregated image features rather than raw 3D coordinates and viewing directions [S1]. Unlike its siblings which often modify the sampling strategy or use sparse voxel structures, R2L retains a dense MLP topology but achieves significant FLOPs reduction (26–35×) and wall-time speedup (28–31×) by trading network width for depth and utilizing hard-example training with pseudo-data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D pixel coordinate $(u, v)$ on the target image plane. The network implicitly assumes the camera pose is known or fixed during inference, mapping the 2D input to a view-dependent color output. Unlike standard NeRF, it does not take 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as direct inputs [S1].
*   **Output:** View-dependent color $\mathbf{c} \in \mathbb{R}^3$ [S1].

**Data Structures:**
*   **Network Architecture:** A Multi-Layer Perceptron (MLP) with extensive residual blocks.
    *   **Width ($W$):** 256 (W256D88) or 181 (W181D88) [S1].
    *   **Depth ($D$):** 88 layers [S1].
    *   **Structure:** The network consists of an input layer, 88 hidden layers, and an output layer. Internal layers are connected via residual blocks [S1].
    *   **Activation:** ReLU is used in hidden layers (standard for such deep networks, implied by "barely trainable" without residuals and standard PyTorch defaults) [S1].

**Training Procedure:**
The method involves two phases: Teacher NeRF training (assumed complete) and R2L Student training.

1.  **Teacher Initialization:** A standard NeRF model is trained on the original captured images. The teacher uses a batch size of 1,024, initial learning rate $5 \times 10^{-4}$, and trains for 200k iterations [S1].
2.  **Pseudo Data Generation:** The pre-trained NeRF teacher is used to synthesize pseudo training images.
    *   Rays are sampled from the teacher model to generate images at various novel viewpoints.
    *   The number of pseudo images is denoted as $S$. Default $S = 10,000$ [S1].
    *   Performance saturates around $S = 5,000$ images [S1].
3.  **Student Network Initialization:** The R2L network is initialized with width $W \in \{256, 181\}$ and depth $D=88$ [S1].
4.  **Hard-Example Training Setup:**
    *   Define a hard-example ratio $r$. Default $r = 0.2$ [S1].
    *   For each training batch, calculate the PSNR of the predicted rays against the ground truth (or pseudo ground truth).
    *   Select the fraction $r$ of rays with the smallest PSNR (hardest examples) to focus optimization on fine-grained details [S1].
5.  **Optimization Loop:**
    *   **Batch Sampling:** Sample a batch of rays with size $B = 98,304$ [S1]. Rays are randomly sampled from different images so they do not share the same origin [S1].
    *   **Hard Example Selection:** From the batch, identify the subset of rays corresponding to the worst-performing fraction $r$ based on previous iteration's error or current forward pass error [S1].
    *   **Forward Pass:** The R2L network processes the 2D coordinates of the selected rays to predict colors [S1].
    *   **Loss Calculation:** Compute the loss (typically MSE) between predicted colors and target colors (from pseudo data or real images) [S1].
    *   **Backward Pass:** Update weights using the Adam optimizer [S1].
    *   **Learning Rate:** Use the same learning rate schedule as the teacher NeRF [S1].
    *   **Iterations:** Train for 1,000k iterations [S1].
6.  **Real Data Integration (Optional but Recommended):** The method can be trained on "Pseudo+Real" data (Ours-2), combining the synthesized pseudo images with the original captured images, which significantly boosts PSNR compared to using only pseudo data (Ours-1) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class ResidualBlock(nn.Module):
    def __init__(self, width):
        super(ResidualBlock, self).__init__()
        self.fc1 = nn.Linear(width, width)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(width, width)
        
    def forward(self, x):
        residual = x
        out = self.fc1(x)
        out = self.relu(out)
        out = self.fc2(out)
        out += residual
        out = self.relu(out)
        return out

class R2LNetwork(nn.Module):
    def __init__(self, width=256, depth=88):
        super(R2LNetwork, self).__init__()
        self.input_layer = nn.Linear(2, width) # Input: 2D pixel coords (u, v)
        self.relu = nn.ReLU()
        
        # Construct deep network with extensive residual blocks
        layers = []
        for _ in range(depth):
            layers.append(ResidualBlock(width))
        self.residual_blocks = nn.Sequential(*layers)
        
        self.output_layer = nn.Linear(width, 3) # Output: RGB color

    def forward(self, x):
        # x shape: (N, 2)
        x = self.input_layer(x)
        x = self.relu(x)
        x = self.residual_blocks(x)
        x = self.output_layer(x)
        return x

def train_r2l(teacher_nerf, real_images, pseudo_image_count=10000, 
              batch_size=98304, hard_ratio=0.2, iterations=1000000):
    """
    Train R2L network by distilling from a pre-trained NeRF teacher.
    """
    # 1. Generate pseudo data using teacher
    # teacher_nerf is assumed to be a callable that renders images given camera poses
    pseudo_dataset = generate_pseudo_images(teacher_nerf, count=pseudo_image_count)
    
    # 2. Initialize R2L student
    student = R2LNetwork(width=256, depth=88)
    optimizer = optim.Adam(student.parameters(), lr=5e-4) # Same schedule as teacher
    
    # 3. Training Loop
    for i in range(iterations):
        # Sample batch of rays from real and/or pseudo data
        # Rays are sampled such that they do not share the same origin
        batch_rays, batch_targets = sample_batch(real_images, pseudo_dataset, batch_size)
        
        # Forward pass
        predictions = student(batch_rays)
        
        # Calculate loss
        loss = nn.MSELoss()(predictions, batch_targets)
        
        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Hard-example training logic (applied to gradient update or sampling)
        # In practice, this often involves re-weighting or sampling strategy
        # The paper states: "selecting a fraction r of rays with small PSNR in each batch"
        # This implies the batch composition or loss weighting is adjusted.
        # For implementation, one might sort rays by error and focus updates on them.
        
        if i % 1000 == 0:
            print(f"Iteration {i}, Loss: {loss.item()}")

    return student

def generate_pseudo_images(teacher_nerf, count):
    """
    Synthesize images using the pre-trained NeRF teacher.
    """
    # Sample novel camera poses
    poses = sample_novel_poses(count)
    pseudo_images = []
    for pose in poses:
        # Render image using teacher NeRF
        img = teacher_nerf.render(pose)
        pseudo_images.append(img)
    return pseudo_images

def sample_batch(real_images, pseudo_images, batch_size):
    """
    Sample rays from real and pseudo images.
    """
    # Combine real and pseudo datasets
    all_images = real_images + pseudo_images
    
    # Randomly select images and rays
    # Ensure rays do not share the same origin (different images)
    selected_rays = []
    selected_targets = []
    
    for _ in range(batch_size):
        img_idx = torch.randint(0, len(all_images), (1,)).item()
        img = all_images[img_idx]
        # Sample a random pixel coordinate (u, v)
        u, v = torch.randint(0, img.shape[0], (2,))
        ray = torch.tensor([u, v])
        target = img[u, v]
        selected_rays.append(ray)
        selected_targets.append(target)
        
    return torch.stack(selected_rays), torch.stack(selected_targets)
```

**Parameters:**
*   **Batch Size:** 98,304 rays per batch [S1].
*   **Hard Example Ratio ($r$):** Fraction of hard examples per batch. Default 0.2. Tested values $\{0, 0.1, 0.2, 0.3\}$ [S1].
*   **Pseudo Sample Size ($S$):** Number of pseudo images. Default 10,000. Performance saturates around 5,000 [S1].
*   **Learning Rate:** Same schedule as the teacher NeRF (initial $5 \times 10^{-4}$, decayed) [S1].
*   **Optimizer:** Adam [S1].
*   **Hardware:** 8 NVIDIA V100 GPUs [S1].
*   **Framework:** PyTorch 1.9 [S1].

**Edge Cases and Pitfalls:**
*   **Overfitting:** The deep network is over-parameterized. Using only the original captured images (small dataset) leads to severe overfitting (e.g., 19.81 PSNR on Lego) [S1]. Pseudo data is critical to prevent this [S1].
*   **Trainability:** Without extensive residual blocks in internal layers, the network is "barely trainable" [S1].
*   **Reflection Surfaces:** Unlike methods relying on ground-truth depth (e.g., DONeRF), R2L handles reflective surfaces (water, mirrors) well because it does not use depth as input [S1].

## Operating characteristics
*   **Time Complexity:** The method achieves a 28–31× wall-time speedup compared to the original NeRF and 2× speedup compared to DONeRF-16-noGT on GeForce 2080Ti, Tesla V100, and CPU benchmarks [S1].
*   **Memory Complexity:** Storage footprint is 23.7 MB for the R2L model, compared to 2.4 MB for the teacher NeRF (note: the table lists 2.4 MB for Teacher NeRF, but R2L is larger due to depth; the text emphasizes FLOPs reduction) [S1]. *Correction:* Table 2 lists Teacher NeRF storage as 2.4 MB and R2L as 23.7 MB. This indicates R2L uses more memory for weights but significantly fewer FLOPs [S1].
*   **FLOPs:** R2L reduces FLOPs to 11.79 M (compared to 303.82 M for Teacher NeRF), a reduction of ~26× [S1].
*   **Performance:**
    *   **NeRF Synthetic Dataset:** Ours-2 (Pseudo+Real) achieves 31.87 PSNR (+1.40 over teacher), 0.9950 SSIM, 0.0340 LPIPS [S1].
    *   **DONeRF Synthetic Dataset:** Ours-2 achieves 35.45 PSNR (+2.78 over teacher), 0.047 FLIP [S1].
*   **Strengths:**
    *   Significantly faster rendering than NeRF and comparable methods.
    *   Better handling of reflective surfaces compared to depth-based methods like DONeRF [S1].
    *   Can outperform the teacher NeRF in PSNR when trained with real data [S1].
*   **Limitations:**
    *   Requires a pre-trained NeRF teacher for distillation.
    *   Blurry parts in NeRF results may also appear in R2L results, as it learns from NeRF-generated data [S1].
    *   Requires substantial pseudo data generation to train the deep network effectively [S1].

## Background
R2L was introduced by H. Wang et al. in the paper "R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis" [S1]. The work addresses the computational inefficiency of NeRF by proposing a deep neural light field network. It differs from baseline NeRF by using a much deeper network with extensive internal residual blocks, trading width for depth under the same FLOPs budget [S1]. The method was evaluated on NeRF synthetic and real-world datasets, demonstrating significant improvements in speed and quality over existing efficient NVS methods like KiloNeRF, RSEN, and DONeRF [S1].

## Sources
[S1] R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis, https://doi.org/10.1007/978-3-031-19821-2_35, fetched 2026-10-26
