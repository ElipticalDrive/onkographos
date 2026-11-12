# Camera-Calibrated Wave Propagation Model Training

Also called: Hardware-in-the-loop Training

## Overview
This method trains a learned surrogate model to predict the optical field distribution at target planes given an input phase pattern, effectively replacing analytical propagation models (such as Angular Spectrum Method) within the Phase-Aware Gradient Descent (PAGD) optimization loop [S1]. It solves the problem of model mismatch caused by real-world optical imperfections, including SLM non-linearities, aberrations, and stray light, which analytical models fail to capture [S1]. By calibrating the propagation model against physical hardware measurements, it achieves a 2 dB PSNR improvement over analytical baselines [S1].

## How it works
**Inputs:**
*   A dataset of phase patterns $\Phi$ (2D and true 3D).
*   Corresponding ground-truth intensity measurements $I_{gt}$ captured at specific distances from the Spatial Light Modulator (SLM).

**Outputs:**
*   A trained neural network model $M_{\theta}$ that maps input phase $\Phi$ to predicted intensity $I_{pred}$ at target planes.

**Data Structures and Layout:**
*   **Phase Patterns:** Stored as 2D arrays. The dataset contains 5860 unique phase patterns [S1].
    *   Types include: 2D phases generated via Stochastic Gradient Descent (SGD), higher-order gradient descent, double-phase amplitude encoding, and true 3D phases from the Flying Planes mesh-wave field-hologram dataset [S1].
*   **Target Planes:** Defined by axial distances $z$ from the SLM. The eight distances are: $z \in \{10.00, 10.07, 10.14, 10.21, 10.28, 10.35, 10.42, 10.49\}$ cm [S1].
*   **Region of Interest (ROI):** The effective active area of the SLM is cropped to $880 \times 1600$ pixels to exclude regions with poor laser uniformity and low modulator efficiency [S1].
*   **SLM Resolution:** The physical SLM is $1920 \times 1080$ with a pixel pitch of $8 \mu m$ [S1].

**Preprocessing:**
1.  **ROI Cropping:** Extract the central $880 \times 1600$ pixel region from the raw $1920 \times 1080$ SLM data [S1].
2.  **Anti-aliasing:** Magnify the wave field by a factor of 2 in both dimensions using grid interpolation (e.g., PyTorch `grid_sample`) [S1].
3.  **Fill Factor:** Apply a fill factor of 0.93 to the magnified field to account for pixel gap effects [S1].

**Training Procedure:**
1.  **Hardware Setup:** Use an RGB single-mode fiber-coupled laser (wavelengths: 639.0 nm, 524.9 nm, 445.8 nm) illuminating a HOLOEYE Pluto-2 phase-only SLM [S1].
2.  **Data Capture:** For each phase pattern in the dataset, record the resulting intensity at all eight target planes ($z$) [S1].
3.  **Optimization:**
    *   **Optimizer:** Adam [S1].
    *   **Learning Rate:** $\eta = 0.0005$ [S1].
    *   **Batch Size:** $B = 1$ [S1].
    *   **Epochs:** $E = 20$ [S1].
    *   **Device:** Single NVIDIA RTX 4090 GPU [S1].
4.  **Loss Function:** Minimize the difference between the model's predicted intensity and the captured ground-truth intensity at the target planes. (Specific loss formulation not provided in sources [G: Standard MSE or L1 loss is implied for intensity matching]).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Hyperparameters
NUM_PHASES = 5860
NUM_TARGET_PLANES = 8
DISTANCES_CM = [10.00, 10.07, 10.14, 10.21, 10.28, 10.35, 10.42, 10.49]
ROI_H, ROI_W = 880, 1600
MAGNIFICATION = 2
FILL_FACTOR = 0.93
BATCH_SIZE = 1
EPOCHS = 20
LEARNING_RATE = 0.0005

# 1. Load and Preprocess Data
# Assume `raw_phases` is a tensor of shape (NUM_PHASES, 1080, 1920)
# Assume `gt_intensities` is a tensor of shape (NUM_PHASES, NUM_TARGET_PLANES, H_gt, W_gt)
# Note: Ground truth must be captured using the hardware setup described in [S1]

def preprocess_phase(phase_batch):
    """
    Applies ROI cropping, magnification, and fill factor.
    """
    # Crop to ROI
    # Assuming center crop or specific alignment; sources specify ROI size but not exact crop offset.
    # [G]: Assuming center crop for demonstration; implementer must verify alignment with hardware capture.
    h_start = (1080 - ROI_H) // 2
    w_start = (1920 - ROI_W) // 2
    cropped = phase_batch[:, h_start:h_start+ROI_H, w_start:w_start+ROI_W]
    
    # Magnify by factor of 2
    magnified = torch.nn.functional.interpolate(
        cropped.unsqueeze(1), 
        size=(ROI_H * MAGNIFICATION, ROI_W * MAGNIFICATION), 
        mode='bilinear', 
        align_corners=False
    ).squeeze(1)
    
    # Apply fill factor
    filled = magnified * FILL_FACTOR
    
    return filled

# 2. Define Model
# The model architecture is not explicitly detailed in the sources [G: Use a CNN or U-Net based surrogate model].
class WavePropagationModel(nn.Module):
    def __init__(self):
        super(WavePropagationModel, self).__init__()
        # Placeholder for model definition
        # The model takes preprocessed phase as input and outputs intensity at target planes
        pass
    
    def forward(self, x):
        # x: preprocessed phase batch
        # Returns: predicted intensities for all 8 target planes
        pass

# 3. Training Loop
model = WavePropagationModel()
optimizer = optim.Adam(model.parameters(), lr=LEARNING_RATE)
criterion = nn.MSELoss() # [G]: Standard loss for intensity regression

# Prepare dataset
# `phases` shape: (NUM_PHASES, 1, 1080, 1920)
# `intensities` shape: (NUM_PHASES, NUM_TARGET_PLANES, H, W)
dataset = torch.utils.data.TensorDataset(phases, intensities)
dataloader = torch.utils.data.DataLoader(dataset, batch_size=BATCH_SIZE, shuffle=True)

for epoch in range(EPOCHS):
    for phase_batch, intensity_batch in dataloader:
        # Preprocess input
        processed_phase = preprocess_phase(phase_batch)
        
        # Forward pass
        predicted_intensity = model(processed_phase)
        
        # Compute loss
        loss = criterion(predicted_intensity, intensity_batch)
        
        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

## Subtopics

### [Camera-Calibrated Learned Wave Propagation Model](12907-camera-calibrated-learned-wave-propagation-model.md)
A learned model that captures real-world optical system behavior (aberrations, SLM non-linearities, stray light, limited diffraction efficiency) to replace numerical models like ASM in the PAGD optimization loop. It provides a PSNR improvement of 2 dB compared to the analytical model (ASM).

## Operating characteristics
*   **Performance:** The method yields a 2 dB PSNR improvement over the analytical Angular Spectrum Method (ASM) [S1].
*   **Hardware Requirements:** Training is performed on a single NVIDIA RTX 4090 GPU [S1].
*   **Data Requirements:** Requires physical capture of 5860 phase patterns across 8 axial distances using a specific hardware setup (HOLOEYE Pluto-2 SLM, RGB lasers) [S1].
*   **Limitations:** The model is specific to the hardware configuration used for calibration (SLM type, laser wavelengths, optical path). It does not generalize to different optical setups without retraining [G].
*   **Complexity:** Not given in the sources.

## Background
The method was introduced in the context of full 3D holographic displays, specifically to address the limitations of analytical wave propagation models in capturing hardware-specific artifacts [S1]. It is part of a mesh-represented and learning-empowered hologram synthesis pipeline [S1]. The hardware setup involves a benchtop prototype using a HOLOEYE Pluto-2 SLM and Fisba ReadyBeam lasers [S1].

## Sources
[S1] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
