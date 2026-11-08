# Deformation Network Architecture


## Overview
The Deformation Network Architecture is a lightweight multi-layer perceptron (MLP) used within the TensoRF framework to predict spatial displacements and occlusion weights for dynamic scenes. It solves the problem of mapping static voxel features to their correct temporal positions by learning a continuous deformation field. It is distinguished from its parent, the TensoRF Feature Decoding Architecture, by its specific role in handling temporal dynamics rather than static appearance, utilizing a compact 4-layer structure to maintain computational efficiency during optimization.

## How it works
**Inputs and Outputs:**
The network takes a concatenated input vector consisting of:
1.  Position $\gamma(p) \in \mathbb{R}^3$: The 3D spatial coordinates of the query point.
2.  Time $\gamma(t) \in \mathbb{R}^1$: The scalar time index or timestamp.
3.  Feature $f \in \mathbb{R}^4$: The interpolated feature vector from the voxel grid.

The network outputs:
1.  Displacement $\Delta p \in \mathbb{R}^3$: The offset vector to apply to the position.
2.  Occlusion weight $w_{occ} \in \mathbb{R}^1$: A scalar indicating occlusion probability.

**Data Structures:**
The architecture consists of four fully connected (FC) layers. Each hidden layer has a width of 64 neurons. The input layer accepts 8 dimensions ($3+1+4$). The output layer produces 4 dimensions ($3+1$).

**Activation Functions:**
*   Hidden layers: Rectified Linear Unit (ReLU).
*   Output layers: Sigmoid function to constrain outputs to the range $(0, 1)$ for $w_{occ}$ and provide bounded gradients for $\Delta p$.

**Algorithm:**
Let $x_0 = [\gamma(p), \gamma(t), f]$. The transformation for layer $i$ is defined as $x_i = \sigma(W_i x_{i-1} + b_i)$, where $W_i$ is the weight matrix and $b_i$ is the bias vector.

1.  **Layer 1:** $x_1 = \text{ReLU}(W_1 x_0 + b_1)$, where $W_1 \in \mathbb{R}^{64 \times 8}$.
2.  **Layer 2:** $x_2 = \text{ReLU}(W_2 x_1 + b_2)$, where $W_2 \in \mathbb{R}^{64 \times 64}$.
3.  **Layer 3:** $x_3 = \text{ReLU}(W_3 x_2 + b_3)$, where $W_3 \in \mathbb{R}^{64 \times 64}$.
4.  **Layer 4:** $x_4 = \text{ReLU}(W_4 x_3 + b_4)$, where $W_4 \in \mathbb{R}^{64 \times 64}$.
5.  **Output:** $[ \Delta p, w_{occ} ] = \text{Sigmoid}(W_5 x_4 + b_5)$, where $W_5 \in \mathbb{R}^{4 \times 64}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class DeformationNetwork(nn.Module):
    def __init__(self):
        super(DeformationNetwork, self).__init__()
        # Input dim: 3 (pos) + 1 (time) + 4 (feature) = 8
        # Hidden dim: 64
        # Output dim: 3 (displacement) + 1 (occlusion) = 4
        
        self.fc1 = nn.Linear(8, 64)
        self.fc2 = nn.Linear(64, 64)
        self.fc3 = nn.Linear(64, 64)
        self.fc4 = nn.Linear(64, 64)
        self.fc_out = nn.Linear(64, 4)
        
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, pos, time, feature):
        # Concatenate inputs: [pos, time, feature]
        # pos: (N, 3), time: (N, 1), feature: (N, 4)
        x = torch.cat([pos, time, feature], dim=-1)
        
        x = self.relu(self.fc1(x))
        x = self.relu(self.fc2(x))
        x = self.relu(self.fc3(x))
        x = self.relu(self.fc4(x))
        
        out = self.fc_out(x)
        
        # Split output into displacement (first 3) and occlusion weight (last 1)
        displacement = out[:, :3]
        occlusion_weight = self.sigmoid(out[:, 3:])
        
        return displacement, occlusion_weight
```

**Parameters:**
*   Hidden width: 64.
*   Number of hidden layers: 4.
*   Loss weights for training (coarse stage): $w_{ptc}=0.1$, $w_{bg}=0.01$, $w_{d\_norm}=0.1$, $w_{d\_tv}$ (value not given in sources) [S1].

**Edge Cases:**
*   **Precision:** The use of sigmoid on the occlusion weight ensures it remains strictly between 0 and 1, preventing invalid probability values [S1].
*   **Boundaries:** The displacement $\Delta p$ is not explicitly bounded by sigmoid, but the ReLU activations in hidden layers and the nature of the training data constrain the magnitude.

## Subtopics

### Neural Body
An implicit neural representation method for synthesizing new views of dynamic characters from sparse multi-view videos, using latent variables anchored to a deformable human model [S2]. It differs from the standard Deformation Network by explicitly anchoring latent variables to the vertices of a deformable human model (such as SMPL) to represent dynamic characters, addressing the ill-posed nature of performance art with large inter-frame pose and appearance variations [S2]. It is chosen when modeling human performance where structural priors are beneficial.

## Operating characteristics
*   **Time Complexity:** The network is lightweight, consisting of only four fully connected layers with width 64. This makes it significantly faster to evaluate than larger color networks (e.g., the color network in the same architecture uses three layers of width 128) [S1].
*   **Memory Complexity:** Low memory footprint due to the small number of parameters ($8 \times 64 + 64^2 \times 3 + 64 \times 4$ weights and biases).
*   **Performance:** The architecture supports fast optimization of dynamic view synthesis. Progressive scale training is employed, doubling the voxel resolution after some iterations to aid convergence [S1].
*   **Strengths:** Computational efficiency allows for real-time or near-real-time optimization of dynamic scenes.
*   **Limitations:** The simplicity of the 4-layer network may limit the complexity of deformations it can model compared to deeper networks, though it is sufficient for the voxel-based approach [S1].

## Background
The Deformation Network Architecture was introduced in the paper "Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis" [S1]. It is part of the TensoRF family of neural rendering methods, specifically designed to handle temporal dynamics in 3D scenes by predicting displacements from a static voxel grid representation.

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
