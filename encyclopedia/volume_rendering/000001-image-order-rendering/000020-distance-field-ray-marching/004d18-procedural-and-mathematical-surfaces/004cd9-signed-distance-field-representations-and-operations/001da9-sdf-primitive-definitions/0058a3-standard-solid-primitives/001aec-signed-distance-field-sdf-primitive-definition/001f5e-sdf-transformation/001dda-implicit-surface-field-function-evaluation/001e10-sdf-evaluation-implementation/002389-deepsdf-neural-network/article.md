# DeepSDF Neural Network

Also called: DeepSDF, DeepSDF Network Architecture

## Overview
DeepSDF is a neural network architecture that learns a continuous signed distance field (SDF) representation for 3D shapes by mapping a shape-specific latent code and 3D coordinates to a signed distance value [S1, unofficial][S2, unofficial]. It solves the problem of implicit surface reconstruction from sparse or unstructured data by learning a probabilistic prior over shape variations conditioned on object category [S1, unofficial]. Unlike [Vox2SDF](4047-vox2sdf.md), which encodes the specific geometry of a discrete input voxel grid into the latent vector for exact reconstruction, DeepSDF relies on a learned prior, allowing it to generalize to unseen shapes within a category but producing a probabilistic average rather than an exact replica of a specific input instance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A latent code vector $z \in \mathbb{R}^{d_z}$ representing the shape, and a 3D query point $p \in \mathbb{R}^3$.
*   **Output:** A scalar signed distance value $s \in \mathbb{R}$.

**Data Structures:**
The model is a Multi-Layer Perceptron (MLP). The specific architecture varies by implementation source:

1.  **Minimal Implementation [S1, unofficial]:**
    *   Input dimension: $d_z + 3$.
    *   Hidden Layer 1: Linear($d_z + 3$, 256), ReLU activation.
    *   Output Layer: Linear(256, 1).
    *   *Note: [S1, unofficial] describes this structure but the provided code snippet in the notes implies a more complex training loop with skip connections, suggesting the minimal description may be a simplified variant or error in the source text. The detailed layer sequence in [S2, unofficial] is used below for the primary method definition as it is more complete.*

2.  **Detailed Implementation [S2, unofficial]:**
    *   **Input Preparation:** Concatenate latent code $z$ and coordinates $p$ along the feature dimension.
    *   **Layer Sequence:**
        1.  Linear(515, 1024) + ReLU. (Input dim 515 implies $d_z=512$ and $3$ coords).
        2.  Linear(1024, 1024) + ReLU.
        3.  Linear(1024, 1024) + ReLU.
        4.  Linear(2042, 1024) + ReLU. (Skip connection from initial input concatenated here).
        5.  Linear(1024, 1024) + ReLU.
        6.  Linear(1024, 1024) + ReLU.
        7.  Linear(2042, 1024) + ReLU. (Skip connection from initial input concatenated here).
        8.  Linear(1024, 1) + Tanh activation.
    *   **Regularization:** Dropout(0.2) applied after each hidden layer except the last.

**Numbered Steps:**
1.  **Latent Code Expansion:** For a batch of queries, expand the single shape-specific latent code $z$ to match the batch size $B$, resulting in $Z \in \mathbb{R}^{B \times d_z}$ [S1, unofficial].
2.  **Input Concatenation:** For each query point $p_i \in \mathbb{R}^3$, concatenate $z_i$ and $p_i$ to form the input vector $x_i = [z_i; p_i] \in \mathbb{R}^{d_z+3}$ [S1, unofficial][S2, unofficial].
3.  **Forward Pass:** Pass $x_i$ through the MLP layers defined above.
    *   For layers $i$ where skip connections are active (e.g., indices 3 and 7 in [S2, unofficial]), concatenate the original input $x_i$ to the current hidden representation before applying the linear transformation [S2, unofficial].
    *   Apply Dropout(0.2) after the output of each hidden layer, excluding the final layer [S2, unofficial].
4.  **Output Activation:** The final layer applies a Tanh activation function, squashing the output to the range $[-1, 1]$ [S2, unofficial].
5.  **Reshape:** Reshape the output tensor to $(-1, 1)$ to return the SDF values for the batch [S1, unofficial].

**Formulas:**
*   **Input Vector:** $x = \text{concat}(z, p)$
*   **Skip Connection Input:** $x_{skip} = \text{concat}(h_{prev}, x)$ for specific layers [S2, unofficial].
*   **Output:** $s = \tanh(\text{Linear}_{last}(h_{last}))$

**Parameters:**
*   **latent_dim ($d_z$):** Dimensionality of the latent code. Default 16 in [S1, unofficial], default 512 in [S2, unofficial].
*   **hidden_dim:** Width of hidden layers. Default 1024 in [S2, unofficial].
*   **num_hidden_layers:** Number of hidden MLP layers. Default 8 in [S2, unofficial].
*   **skip_layers:** Indices of layers receiving skip connections. Default [3, 7] in [S2, unofficial].
*   **lr:** Learning rate for Adam optimizer. Default 0.001 [S1, unofficial].
*   **batch_size:** Number of samples per batch. Default 128 [S1, unofficial].
*   **num_epochs:** Number of training epochs. Default 10 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The Tanh output limits the SDF range to $[-1, 1]$. This requires the input geometry to be normalized to fit within this range, or the SDF values will be clipped [S2, unofficial].
*   **Boundary:** The latent code is category-specific. Querying with a latent code from one category on geometry from another yields undefined results [S1, unofficial].
*   **Degenerate Input:** If the latent code is all zeros or noise, the network outputs a default shape determined by the training distribution [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class DeepSDFNetwork(nn.Module):
    def __init__(self, latent_dim=512, hidden_dim=1024, num_hidden_layers=8, skip_layers=[3, 7]):
        super(DeepSDFNetwork, self).__init__()
        self.latent_dim = latent_dim
        self.hidden_dim = hidden_dim
        self.skip_layers = set(skip_layers)
        
        # Layer 0
        self.layers = nn.ModuleList()
        self.layers.append(nn.Sequential(
            nn.Linear(latent_dim + 3, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.2)
        ))
        
        # Hidden Layers 1 to num_hidden_layers-1
        for i in range(1, num_hidden_layers):
            if i in self.skip_layers:
                self.layers.append(nn.Sequential(
                    nn.Linear(hidden_dim + latent_dim + 3, hidden_dim),
                    nn.ReLU(),
                    nn.Dropout(0.2)
                ))
            else:
                self.layers.append(nn.Sequential(
                    nn.Linear(hidden_dim, hidden_dim),
                    nn.ReLU(),
                    nn.Dropout(0.2)
                ))
        
        # Output Layer
        self.layers.append(nn.Sequential(
            nn.Linear(hidden_dim, 1),
            nn.Tanh()
        ))

    def forward(self, latent_code, points):
        """
        Args:
            latent_code: Tensor of shape (B, latent_dim)
            points: Tensor of shape (B, 3)
        Returns:
            sdf_values: Tensor of shape (B, 1)
        """
        # Concatenate latent code and points
        x = torch.cat([latent_code, points], dim=-1)
        
        h = x
        for i, layer in enumerate(self.layers):
            # Skip connections are applied before the linear layer of specific indices
            # Note: The source [S2, unofficial] lists layers sequentially. 
            # Index 0 is the first layer. 
            # If i-1 is in skip_layers, we concatenate x to the input of layer i.
            # However, the source description says "if i-1 in skip_layers... concatenate".
            # Let's map the source's layer sequence to this logic.
            # Source: [Linear(515,1024)] -> [Linear(1024,1024)] -> [Linear(1024,1024)] -> [Linear(2042,1024)]
            # This implies the skip happens at the 4th layer (index 3).
            # So if current layer index i is 3, we use skip.
            
            if i in self.skip_layers:
                h = torch.cat([h, x], dim=-1)
            
            h = layer(h)
            
        return h
```

## Subtopics

### Vox2SDF
Vox2SDF is a neural network pipeline that converts a discrete 3D voxel grid into a continuous Signed Distance Field (SDF) representation [S1, unofficial]. It solves the problem of implicit surface reconstruction from discrete inputs by encoding the voxel structure into a global latent vector, which is then decoded by a position-conditional Multi-Layer Perceptron (MLP) to predict SDF values at arbitrary 3D coordinates [S1, unofficial]. Unlike [DeepSDF](3051-deepsdf-neural-network.md), which relies on a learned prior over shape variations via a latent code conditioned on object category, Vox2SDF directly encodes the specific geometry of the input voxel grid into the latent vector, allowing for exact reconstruction of the input shape rather than a probabilistic average [S1, unofficial].

### [DeepSDF Training Loop](4738-deepsdf-training-loop.md)
The DeepSDF Training Loop is the optimization procedure for the DeepSDF model, which jointly learns a continuous signed distance field (SDF) representation for 3D shapes and a per-shape latent code. It solves the problem of reconstructing arbitrary 3D geometry from sparse or unstructured point cloud data by mapping (latent code, 3D coordinate) pairs to signed distance values using a multi-layer perceptron. It differs from Vox2SDF by optimizing a continuous implicit function and per-shape latent codes rather than relying solely on a discrete grid decoder.

### DeepSDF Training with Image Latents
Jointly trains a ResNet-18 encoder and DeepSDF to learn an implicit surface per image from a training set [S3]. It encodes the rendering of each shape into a latent vector using ResNet-18, concatenates this latent vector with a query point, and learns the SDF for that point using DeepSDF [S3]. Query points are sampled close to the surface (400,000 points) [S3]. It is trained for 2,000 epochs per category using the ADAM optimizer [S3].

### Learnable Signed Distance Encoding
Uses a learnable sign indicator to compute interpolated signed distances for spatial query points to provide spatial distinguishability perpendicular to the surface, avoiding issues with non-watertight meshes where physical SDF sign might reverse [S4]. It differs from physically computed signed distance (Liu et al. [21]) which fails on general object meshes that are not watertight or have predefined skinning weights, causing ray-to-mesh intersection counting confusion and unexpected sign reversal [S4]. It differs from relative position encoding (PointNet [38]) which concatenates codes and relative coordinate offsets encoded by a shallow MLP (2 hidden layers, 64 hidden sizes) and uses inverse distance weighting. Learnable signed distance shows better performance (PSNR 27.254 vs 26.308 on Lego, 27.881 vs 27.270 on DTU 69) [S4].

### Independent SDF Prediction
Uses two independent SDF networks for deformable and non-deformable entities to learn semantically separable geometry in canonical space [S5]. It assigns a semantic label weight $\omega_j$ to each 3D point based on distance to the nearest 3D joint in the per-entity canonical skeleton $J_{j p0}$: $\omega_j = \exp(-\text{dist}^2 / \sigma^2)$ [S5]. It assigns background weight $\omega_{bg}$ for points far from skeletons: $\omega_{bg} = 1.0 - \text{clamp}(\sum_j \omega_j, 1.0)$ [S5]. It concatenates semantic label (dim 3), canonical point, and pose input (total dim $3+3+n_b$) and passes through the SDF prediction network [S5]. The network $F_{c \to \Omega}^j$ takes input dimension $\mathbb{R}^{3+3+n_b}$ and outputs dimension $\mathbb{R}^{1+256}$ (SDF scalar + 256-dim global feature) [S5].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Inference is $O(L \cdot H)$ per query point, where $L$ is the number of layers and $H$ is the hidden dimension. Training time depends on the number of epochs and batch size.
*   **Memory:** Memory usage is dominated by the latent code storage ($O(N \cdot d_z)$ for $N$ shapes) and the MLP weights ($O(L \cdot H^2)$).

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Continuous representation allows for high-resolution mesh extraction; latent code enables compact storage and interpolation of shapes [S1, unofficial].
*   **Limitations:** Requires a large dataset for training; generalization to unseen categories is poor without retraining; the Tanh output limits the SDF range [S2, unofficial].

**When to Use:**
*   Use when a continuous, differentiable representation of 3D shapes is needed for tasks like rendering, editing, or interpolation [S1, unofficial].

**When Not to Use:**
*   Do not use when exact reconstruction of a specific input geometry is required without a learned prior (use [Vox2SDF](4047-vox2sdf.md) instead) [S1, unofficial].

## Background
DeepSDF was introduced by [S1, unofficial] and [S2, unofficial] as a method for learning continuous signed distance fields. It builds on the idea of using neural networks to represent implicit surfaces, allowing for the learning of shape priors from large datasets of 3D models [S1, unofficial]. The method has been extended to various tasks, including single-view reconstruction [S3], texture editing [S4], and dynamic scene reconstruction [S5].

## Sources
[S1] Kulakov-Nikita/RayMarchingAI, https://github.com/Kulakov-Nikita/RayMarchingAI, fetched 2026-10-26 (unofficial)
[S2] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
[S3] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
[S4] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
[S5] TFS-NeRF: Template-Free NeRF for Semantic 3D Reconstruction of Dynamic Scene, https://arxiv.org/abs/2409.17459v4, fetched 2026-10-26
