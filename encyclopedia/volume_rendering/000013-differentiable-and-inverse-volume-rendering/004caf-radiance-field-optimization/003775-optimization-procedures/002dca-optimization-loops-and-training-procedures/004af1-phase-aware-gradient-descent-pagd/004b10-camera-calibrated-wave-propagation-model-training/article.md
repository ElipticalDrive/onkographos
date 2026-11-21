# Camera-Calibrated Wave Propagation Model Training

Also called: Hardware-in-the-loop Training

## Overview
This method trains a data-driven surrogate model using experimentally captured phase-to-intensity data to predict light propagation from a Spatial Light Modulator (SLM) to multiple target planes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset of phase patterns $\Phi$ (2D or 3D) and their corresponding captured intensity images $I$ at eight specific propagation distances [S1].
*   **Output:** A trained neural network model $M_\theta$ that predicts intensity $I_{pred}$ given an input phase $\Phi$.

**Data Structures:**
*   **Phase Patterns:** Stored as 2D arrays representing phase values [S1].
*   **Intensity Images:** Captured by the camera at the target planes [S1].
*   **Region of Interest (ROI):** The effective usable area is defined as $880 \times 1600$ pixels, cropped from the full $1920 \times 1080$ resolution due to laser uniformity and modulator efficiency [S1].

**Preprocessing:**
1.  **Magnification:** To mitigate anti-aliasing, the wave field is magnified by a factor of two in both dimensions [S1].
2.  **Fill Factor:** A fill factor of $0.93$ is applied to the magnified field [S1].
3.  **Interpolation:** Grid interpolation is performed using PyTorch to handle spatial transformations [S1].

**Training Procedure:**
1.  **Dataset Construction:** Collect 5860 phase patterns. These include 2D phases generated via stochastic gradient descent, higher-order gradient descent, and double-phase amplitude encoding, as well as true 3D phases derived from the Flying Planes mesh-wave field-hologram dataset [S1].
2.  **Data Capture:** For each phase pattern, record the resulting intensity at eight target planes located at distances $d \in \{10.00, 10.07, 10.14, 10.21, 10.28, 10.35, 10.42, 10.49\}$ cm from the SLM [S1].
3.  **Optimization:**
    *   **Hardware:** Single NVIDIA RTX 4090 GPU [S1].
    *   **Optimizer:** Adam [S1].
    *   **Learning Rate:** $\eta = 0.0005$ [S1].
    *   **Batch Size:** $B = 1$ [S1].
    *   **Epochs:** $E = 20$ [S1].
    *   **Loss Function:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class WavePropagationModel(nn.Module):
    def __init__(self):
        super().__init__()
        # Architecture details not given in the sources.
        self.net = nn.Sequential(
            nn.Conv2d(1, 32, 3, padding=1),
            nn.ReLU(),
            nn.Conv2d(32, 8, 1)
        )

    def forward(self, phase_input):
        # Preprocessing: magnify 2x and apply fill factor 0.93
        magnified = torch.nn.functional.interpolate(phase_input, scale_factor=2, mode='bilinear', align_corners=False)
        filled = magnified * 0.93
        return self.net(filled)

def train_model(model, dataloader, device):
    optimizer = optim.Adam(model.parameters(), lr=0.0005)
    epochs = 20
    batch_size = 1

    for epoch in range(epochs):
        model.train()
        for phases, targets in dataloader:
            phases = phases.to(device)
            targets = targets.to(device)

            optimizer.zero_grad()
            predictions = model(phases)

            # Loss function not given in the sources.
            loss = nn.MSELoss()(predictions, targets)
            loss.backward()
            optimizer.step()

        print(f"Epoch {epoch+1}/{epochs}, Loss: {loss.item()}")
```

## Subtopics

### [Camera-Calibrated Learned Wave Propagation Model](12907-camera-calibrated-learned-wave-propagation-model.md)
A data-driven surrogate for numerical propagation operators. It solves the problem of model mismatch between idealized physics simulations and real-world optical hardware, specifically accounting for spatial light modulator (SLM) non-linearities, optical aberrations, stray light, and limited diffraction efficiency. Unlike analytical models that assume perfect free-space propagation, this learned model maps input phase patterns directly to captured multi-plane intensity images, providing a more accurate gradient signal for phase-aware gradient descent (PAGD).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Training is performed on a single NVIDIA RTX 4090 GPU [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths and Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.

## Background
This method originates from research on mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays [S1].

## Sources
[S1] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
