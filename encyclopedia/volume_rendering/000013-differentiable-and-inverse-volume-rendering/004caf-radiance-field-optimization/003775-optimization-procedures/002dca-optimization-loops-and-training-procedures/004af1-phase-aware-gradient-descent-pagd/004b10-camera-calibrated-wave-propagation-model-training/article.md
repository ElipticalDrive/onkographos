# Camera-Calibrated Wave Propagation Model Training

Also called: Hardware-in-the-loop Training

## Overview
This method trains a data-driven surrogate model to predict the intensity distribution of light propagating from a Spatial Light Modulator (SLM) to multiple target planes, capturing real-world optical imperfections such as non-linearities, aberrations, and stray light. It solves the model mismatch problem inherent in analytical propagation methods (like the Angular Spectrum Method) by replacing idealized physics with empirical measurements, thereby providing accurate gradient signals for phase-aware gradient descent (PAGD) [S1]. It is distinct from analytical models by mapping input phase patterns directly to captured multi-plane intensity images rather than computing them via wave optics equations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset of phase patterns $\Phi$ (2D or 3D) and their corresponding captured intensity images $I$ at eight specific propagation distances.
*   **Output:** A trained neural network model $M_\theta$ that predicts intensity $I_{pred}$ given an input phase $\Phi$.

**Data Structures:**
*   **Phase Patterns:** Stored as 2D arrays representing phase values. The source does not specify the exact bit-depth or normalization range (e.g., $[0, 2\pi]$ or $[-1, 1]$) [G].
*   **Intensity Images:** Grayscale or RGB images captured by the camera.
*   **Region of Interest (ROI):** The effective usable area of the SLM is defined as $880 \times 1600$ pixels, cropped from the full $1920 \times 1080$ resolution due to laser uniformity and modulator efficiency [S1].

**Preprocessing:**
1.  **Magnification:** To mitigate anti-aliasing, the wave field is magnified by a factor of two in both dimensions [S1].
2.  **Fill Factor:** A fill factor of $0.93$ is applied to the magnified field [S1].
3.  **Interpolation:** Grid interpolation is performed using PyTorch to handle the spatial transformations [S1].

**Training Procedure:**
1.  **Dataset Construction:** Collect 5860 phase patterns. These include:
    *   2D phases generated via Stochastic Gradient Descent (SGD).
    *   2D phases generated via higher-order gradient descent.
    *   2D phases generated via double-phase amplitude encoding.
    *   True 3D phases derived from the "Flying Planes" mesh-wave field-hologram dataset [S1].
2.  **Data Capture:** For each phase pattern, record the resulting intensity at eight target planes located at distances $d \in \{10.00, 10.07, 10.14, 10.21, 10.28, 10.35, 10.42, 10.49\}$ cm from the SLM [S1].
3.  **Optimization:**
    *   **Hardware:** Single NVIDIA RTX 4090 GPU [S1].
    *   **Optimizer:** Adam [S1].
    *   **Learning Rate:** $\eta = 0.0005$ [S1].
    *   **Batch Size:** $B = 1$ [S1].
    *   **Epochs:** $E = 20$ [S1].
    *   **Loss Function:** Not explicitly defined in the sources, but typically Mean Squared Error (MSE) or L1 loss is used for intensity prediction [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Define the surrogate model architecture (e.g., CNN)
class WavePropagationModel(nn.Module):
    def __init__(self):
        super(WavePropagationModel, self).__init__()
        # Architecture details not provided in sources [G]
        self.layers = nn.Sequential(
            # Placeholder for actual layers
            nn.Conv2d(1, 32, kernel_size=3, padding=1),
            nn.ReLU(),
            # ... additional layers ...
            nn.Conv2d(32, 8, kernel_size=1) # Output 8 channels for 8 planes
        )

    def forward(self, phase_input):
        # phase_input shape: (Batch, 1, H, W)
        # Preprocessing: Magnify by 2x and apply fill factor 0.93
        # Note: PyTorch grid_interpolation is used for this [S1]
        magnified = torch.nn.functional.interpolate(phase_input, scale_factor=2, mode='bilinear', align_corners=False)
        filled = magnified * 0.93
        return self.layers(filled)

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
            
            # Loss calculation (assumed MSE based on standard practice [G])
            loss = nn.MSELoss()(predictions, targets)
            loss.backward()
            optimizer.step()
            
        print(f"Epoch {epoch+1}/{epochs}, Loss: {loss.item()}")

# Data loading specifics:
# - 5860 phases total [S1]
# - 8 target planes at distances: 10.00, 10.07, 10.14, 10.21, 10.28, 10.35, 10.42, 10.49 cm [S1]
# - ROI: 880x1600 pixels [S1]
```

## Subtopics

### [Camera-Calibrated Learned Wave Propagation Model](12907-camera-calibrated-learned-wave-propagation-model.md)
A data-driven surrogate for numerical propagation operators. It solves the problem of model mismatch between idealized physics simulations and real-world optical hardware, specifically accounting for spatial light modulator (SLM) non-linearities, optical aberrations, stray light, and limited diffraction efficiency. Unlike analytical models that assume perfect free-space propagation, this learned model maps input phase patterns directly to captured multi-plane intensity images, providing a more accurate gradient signal for phase-aware gradient descent (PAGD).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Training is performed on a single NVIDIA RTX 4090 GPU [S1].
*   **Strengths:** Captures real-world optical imperfections (non-linearities, aberrations, stray light) that analytical models miss [S1].
*   **Limitations:** Requires extensive hardware calibration and data capture (5860 patterns, 8 planes) [S1]. Dependent on the specific hardware setup (SLM, laser, camera) [S1].
*   **When to use:** When high-fidelity holographic reconstruction is required and analytical models fail to account for hardware-specific distortions [S1].
*   **When not to use:** When computational resources for training are unavailable or when the hardware setup differs significantly from the calibration setup [G].

## Background
This method originates from research on mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays [S1]. It is part of a broader effort to integrate differentiable rendering and optimization loops with physical hardware constraints, moving beyond idealized simulations to "hardware-in-the-loop" training [S1]. The approach builds on the need for accurate gradient signals in phase-aware gradient descent (PAGD) [S1].

## Sources
[S1] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
