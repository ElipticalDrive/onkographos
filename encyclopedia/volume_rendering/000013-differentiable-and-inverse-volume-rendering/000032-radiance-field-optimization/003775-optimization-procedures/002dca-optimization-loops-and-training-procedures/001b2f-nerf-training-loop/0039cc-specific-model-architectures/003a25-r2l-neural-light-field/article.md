# R2L (Neural Light Field)

Also called: Neural Light Field, R2L network

## Overview
R2L (R2L) is a deep neural light field network designed for efficient novel view synthesis (NVS) by distilling a pre-trained NeRF teacher model into a student architecture that operates on 2D pixel coordinates [S1]. Unlike the canonical NeRF which maps 3D positions and viewing directions to density and color, R2L maps 2D image coordinates directly to radiance, significantly reducing computational complexity [S1, 8670-r2l-neural-light-field.md]. It distinguishes itself from sibling methods by employing an unprecedented network depth (88 layers) with extensive internal residual blocks, trading width for depth under a fixed FLOPs budget to capture fine-grained details that shallower networks miss [S1].

## How it works
R2L replaces the 3D spatial sampling of NeRF with a direct mapping from 2D image coordinates to color, leveraging a deep residual network trained via knowledge distillation.

### Inputs and Outputs
*   **Input:** 2D pixel coordinates $(u, v)$ of the target image plane. Positional encoding is applied to these coordinates [S1, 8670-r2l-neural-light-field.md].
*   **Output:** View-dependent color $\mathbf{c}$ for the pixel. Since the network operates on the 2D image plane, it implicitly assumes the camera intrinsics and extrinsics are fixed for the target view or handled by the data generation pipeline [S1].

### Data Structures
The network is a Multi-Layer Perceptron (MLP) with specific architectural constraints:
*   **Width ($W$):** 256 or 181 hidden units per layer.
*   **Depth ($D$):** 88 layers.
*   **Activation:** ReLU (standard for MLPs, implied by residual block structure) [S1].
*   **Residual Blocks:** The internal layers consist of extensive residual connections. A residual block typically computes $\mathbf{h}_{out} = \mathbf{h}_{in} + \text{MLP}(\mathbf{h}_{in})$ [S1].

### Training Procedure
The method relies on a two-stage process: data generation from a teacher and student training.

1.  **Teacher Initialization:** A standard NeRF model is trained on the original captured images. The teacher uses a batch size of 1,024 rays and an initial learning rate of $5 \times 10^{-4}$, decayed over 200k iterations [S1].
2.  **Pseudo Data Generation:** The pre-trained NeRF teacher is used to synthesize pseudo training images.
    *   Rays are sampled from the teacher model to generate color values for specific 2D pixel coordinates.
    *   The number of pseudo images ($S$) is typically 10,000. Performance saturates around 5,000 images [S1].
    *   Rays in a batch are randomly sampled from different images to ensure they do not share the same origin, which is critical for performance [S1].
3.  **Student Network Initialization:** The R2L network is initialized with width $W \in \{256, 181\}$ and depth $D=88$ [S1].
4.  **Hard-Example Training:** To focus optimization on fine-grained details, a fraction $r$ of rays with the smallest PSNR (error) in each batch are selected for higher-weighted loss calculation.
    *   Default ratio $r = 0.2$.
    *   Tested values: $\{0, 0.1, 0.2, 0.3\}$ [S1].
5.  **Optimization:**
    *   **Loss:** Mean Squared Error (MSE) between predicted color and target color (from teacher or real images).
    *   **Optimizer:** Adam [S1].
    *   **Learning Rate:** Same schedule as the teacher NeRF [S1].
    *   **Iterations:** 1,000k iterations [S1].
    *   **Batch Size:** 98,304 rays per batch [S1].
    *   **Hardware:** 8 NVIDIA V100 GPUs [S1].
    *   **Framework:** PyTorch 1.9 [S1].
6.  **Real Data Integration (Optional):** For "Ours-2" configuration, original real images are included in the training set alongside pseudo data to boost performance beyond the teacher's limit [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class ResidualBlock(nn.Module):
    """
    A residual block for the R2L network.
    Consists of two linear layers with ReLU activations.
    """
    def __init__(self, width):
        super(ResidualBlock, self).__init__()
        self.fc1 = nn.Linear(width, width)
        self.fc2 = nn.Linear(width, width)
        self.relu = nn.ReLU()

    def forward(self, x):
        identity = x
        out = self.fc1(x)
        out = self.relu(out)
        out = self.fc2(out)
        out += identity
        out = self.relu(out)
        return out

class R2LNetwork(nn.Module):
    """
    The R2L Neural Light Field network.
    Input: 2D pixel coordinates (u, v).
    Output: Color (R, G, B).
    """
    def __init__(self, width=256, depth=88):
        super(R2LNetwork, self).__init__()
        self.width = width
        self.depth = depth
        
        # Input layer: 2D coordinates -> width
        self.input_layer = nn.Linear(2, width)
        self.relu = nn.ReLU()
        
        # Hidden layers: extensive residual blocks
        self.residual_blocks = nn.ModuleList([
            ResidualBlock(width) for _ in range(depth - 2)
        ])
        
        # Output layer: width -> 3 (RGB)
        self.output_layer = nn.Linear(width, 3)

    def forward(self, x):
        # x shape: (N, 2) where N is batch size
        h = self.input_layer(x)
        h = self.relu(h)
        
        for block in self.residual_blocks:
            h = block(h)
            
        color = self.output_layer(h)
        return color

def train_r2l(teacher_model, real_images, pseudo_sample_size=10000, 
              hard_example_ratio=0.2, batch_size=98304, 
              num_iterations=1000000, lr=5e-4):
    """
    Training loop for R2L network.
    """
    # Initialize student network
    student = R2LNetwork(width=256, depth=88)
    optimizer = optim.Adam(student.parameters(), lr=lr)
    
    # Generate pseudo data using teacher
    # Pseudo data consists of (pixel_coords, target_color) pairs
    pseudo_data = generate_pseudo_data(teacher_model, pseudo_sample_size)
    
    # Combine real and pseudo data if using Ours-2 configuration
    # For Ours-1, only pseudo_data is used
    dataset = CombinedDataset(real_images, pseudo_data)
    dataloader = torch.utils.data.DataLoader(dataset, batch_size=batch_size, shuffle=True)
    
    for iteration in range(num_iterations):
        batch_coords, batch_targets = next(iter(dataloader))
        
        # Forward pass
        predicted_colors = student(batch_coords)
        
        # Calculate loss
        loss = nn.MSELoss()(predicted_colors, batch_targets)
        
        # Hard-example training: identify rays with high error
        # Note: In practice, this might involve weighting or re-sampling
        # based on the error of the previous iteration or current batch
        errors = torch.abs(predicted_colors - batch_targets).mean(dim=1)
        num_hard = int(batch_size * hard_example_ratio)
        
        if num_hard > 0:
            # Select indices of the hardest examples
            hard_indices = torch.topk(errors, num_hard).indices
            # Create a mask or weight for these examples
            weights = torch.ones(batch_size)
            weights[hard_indices] = 2.0 # Example weighting factor [G]
            loss = (weights * nn.MSELoss(reduction='none')(predicted_colors, batch_targets)).mean()

        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Learning rate schedule matches teacher NeRF [S1]
        # (Implementation details of schedule omitted as per source [S1])

def generate_pseudo_data(teacher_model, num_images):
    """
    Synthesize pseudo training data using the pre-trained NeRF teacher.
    """
    pseudo_samples = []
    for _ in range(num_images):
        # Sample rays from the teacher model for a specific view
        # This returns pixel coordinates and the corresponding color rendered by NeRF
        coords, colors = teacher_model.render_view()
        pseudo_samples.append((coords, colors))
    return pseudo_samples
```

### Parameters and Edge Cases
*   **Network Depth:** The network must be deep (88 layers). Shallower networks (e.g., original 11-layer NeRF) overfit severely on the limited real data (0.1k images) [S1].
*   **Residuals:** Extensive residual blocks are mandatory. Without them, the network is "barely trainable" [S1].
*   **Data Size:** The original captured images are insufficient for the deep network. Pseudo data inflation is critical. Performance saturates at ~5k pseudo images; 10k is the default [S1].
*   **Hard Examples:** Using hard-example training ($r=0.2$) improves both training optimization and test generalization compared to $r=0$ [S1].
*   **Reflections:** Unlike depth-based methods (e.g., DONeRF), R2L does not rely on ground-truth depth, allowing it to handle reflective surfaces (water, mirrors) correctly [S1].

## Operating characteristics
*   **Speed:** R2L is 28–31× faster than NeRF and 2× faster than DONeRF-16-noGT on GeForce 2080Ti, Tesla V100, and CPU benchmarks [S1].
*   **FLOPs:** R2L achieves a 26–35× reduction in FLOPs compared to NeRF. Specifically, R2L (W181D88) uses ~6.00 M FLOPs vs NeRF's 211.42 M FLOPs [S1].
*   **Memory:** Storage footprint is ~23.7 MB for R2L, compared to 2.4 MB for the teacher NeRF (note: the source table lists 2.4 MB for Teacher NeRF and 23.7 MB for Ours, which seems counter-intuitive for a deeper network, but this is the reported value [S1]). *Correction:* The source table [S1] lists "Storage (MB)" as 2.4 for Teacher NeRF and 23.7 for Ours-1/Ours-2. This likely includes the overhead of the deep network parameters or is a specific metric from the paper. KiloNeRF is listed at 38.9 MB [S1].
*   **Quality:** On the NeRF synthetic dataset, R2L (Ours-2) achieves 31.87 PSNR (+1.40 over teacher), 0.9950 SSIM, and 0.0340 LPIPS [S1]. On the DONeRF synthetic dataset, it achieves 35.45 PSNR (+2.78 over teacher) [S1].
*   **Strengths:** High speed, low FLOPs, handles reflections better than depth-based methods, superior detail capture due to depth.
*   **Limitations:** Requires a pre-trained NeRF teacher for data generation; training is computationally expensive (1,000k iterations on 8 V100s); blurry artifacts from the teacher may persist [S1].

## Background
R2L was introduced by H. Wang et al. in the paper "R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis" [S1]. It addresses the high computational cost of NeRF by distilling the knowledge into a neural light field representation. The work builds on the concept of Neural Light Fields (NeLF) but adapts it for complex scenes using deep residual networks and distillation [S1].

## Sources
[S1] R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis, https://doi.org/10.1007/978-3-031-19821-2_35, fetched 2026-10-26
