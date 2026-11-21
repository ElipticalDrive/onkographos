# Neural Transient Fields (NeTF)
## Overview
Neural Transient Fields (NeTF) is a differentiable volume rendering method for Non-Line-of-Sight (NLOS) imaging that models the plenoptic transient field using a Multi-Layer Perceptron (MLP) [S1]. It reconstructs hidden 3D scenes from sparse Time-Correlated Single-Photon Counting (TCSPC) measurements captured by a Single-Photon Avalanche Diode (SPAD) array [S1]. Unlike [NeRF](4972-neural-radiance-fields.md), which relies on dense, high-resolution RGB images to mitigate occlusion artifacts, NeTF operates on sparse spot measurements where occluded points may be visible from very few viewpoints, making occlusion handling and sampling bias critical challenges [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse SPAD measurements of transient signals $\tau_m(x', y', t)$ on a relay wall, where $(x', y')$ are the 2D coordinates of the detection spots and $t$ is time [S1].
*   **Output:** A continuous function $\tau(x', y', t)$ representing the transient arrival time at the relay wall spots $(x', y')$ and time $t$ [S1].

**Data Structures:**
*   **MLP:** Takes 3D Cartesian coordinates of hidden scene points as input [S1].
*   **Network Output:** Predicts the transient arrival time $\tau$ [S1]. (Source [S1] mentions predicting density and reflectance in one passage, but the optimization explicitly minimizes the L2 loss on $\tau$ [S1].)

**Algorithm Steps:**
1.  **Coordinate Transformation:** Convert hidden scene points to Cartesian coordinates $(x, y, z)$ [S1].
2.  **MLP Forward Pass:** Feed $(x, y, z)$ into the MLP to predict $\tau(x', y', t)$ [S1].
3.  **Loss Calculation:** Compute the L2 norm loss between predicted $\tau$ and measured $\tau_m$ at SPAD spots [S1].
4.  **Optimization:** Minimize loss via gradient descent [S1].
5.  **Two-Stage Training:** Stage 1 trains on all samples to obtain an initial reconstruction [S1]. Stage 2 calculates the loss at every measurement spot, resamples the dataset to balance the distribution (over-sampling minority classes with high loss), and retrains to reduce artifacts [S1].

**Formulas:**
The optimization objective is defined by the L2 loss function [S1]:
$$
L = \sum_{x', y', t} (\tau_m(x', y', t) - \tau(x', y', t))^2 \quad (14)
$$
Where:
*   $L$ is the total loss.
*   $\tau_m(x', y', t)$ is the measured transient at spot $(x', y')$ and time $t$ [S1].
*   $\tau(x', y', t)$ is the predicted transient at the same location and time [S1].

**Parameters:**
*   **Loss Function:** L2 norm difference [S1].
*   **Sampling:** 2D hemispherical sampling is used for the SPAD spots, differing from NeRF's 1D ray sampling [S1].

**Pitfalls:**
*   **Occlusion Artifacts:** Because SPAD captures sparse spots, occluded points may be captured from very few viewpoints. If not handled, this leads to strong reconstruction artifacts [S1].
*   **Sampling Bias:** The sparse nature of the data creates a long-tailed distribution of visibility, similar to long-tailed classification problems in machine learning [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class NeuralTransientField(nn.Module):
    def __init__(self, input_dim=3, hidden_dim=256, output_dim=1):
        super(NeuralTransientField, self).__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, output_dim)
        )

    def forward(self, xyz):
        """Predicts transient arrival time tau for 3D coordinates."""
        return self.network(xyz)

def train_netf(measured_xyz, measured_tau, num_epochs=1000):
    """Two-stage training strategy for NeTF."""
    model = NeuralTransientField()
    optimizer = optim.Adam(model.parameters(), lr=0.001)

    # Stage 1: Initial training with all samples
    for epoch in range(num_epochs // 2):
        optimizer.zero_grad()
        predicted_tau = model(measured_xyz)
        # Loss formula from source [S1]: sum of squared differences
        loss = torch.sum((predicted_tau - measured_tau) ** 2)
        loss.backward()
        optimizer.step()

    # Stage 2: Resampling based on loss
    with torch.no_grad():
        predicted_tau_stage1 = model(measured_xyz)
        losses = (predicted_tau_stage1 - measured_tau) ** 2

    # Resample dataset to balance distribution (over-sample high loss) [G]
    probs = losses / losses.sum()
    indices = torch.multinomial(probs, len(measured_xyz), replacement=True)
    xyz_resampled = measured_xyz[indices]
    tau_resampled = measured_tau[indices]

    # Retrain with resampled data
    for epoch in range(num_epochs // 2, num_epochs):
        optimizer.zero_grad()
        predicted_tau = model(xyz_resampled)
        loss = torch.sum((predicted_tau - tau_resampled) ** 2)
        loss.backward()
        optimizer.step()

    return model
```

## Subtopics

### [Non-confocal Neural Transient Field (NeTF) Formulation](6348-non-confocal-neural-transient-field-netf-formulation.md)
Formulates the transient signal for non-confocal NLOS setups where illumination and detection spots are distinct, using ellipsoidal coordinates.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Can model the plenoptic transient field using an MLP, allowing for the minimization of arbitrary differentiable losses [S1]. The two-stage training strategy effectively addresses sampling bias and occlusion artifacts inherent in sparse SPAD data [S1].
*   **Limitations:** Requires careful handling of occlusions due to sparse viewpoint coverage [S1]. The sparse nature of the input data makes it more sensitive to sampling bias than dense-image methods like NeRF [S1].
*   **When to use:** Non-Line-of-Sight imaging scenarios with sparse SPAD measurements where occlusion is a significant factor [S1].
*   **When not to use:** Dense viewpoint settings where occlusion is less significant, or when high-resolution RGB images are available (prefer NeRF) [S1].

## Background
Neural Transient Fields (NeTF) was introduced in the paper "Non-line-of-Sight Imaging via Neural Transient Fields" [S1]. It adapts the neural radiance field concept to the domain of NLOS imaging, addressing the unique challenges of sparse, time-resolved photon data [S1]. The method builds on the principles of [Radiance-field optimization](44-radiance-field-optimization.md) but modifies the sampling and loss strategies to handle the sparsity and occlusion issues specific to SPAD-based NLOS systems [S1].

## Sources
[S1] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
