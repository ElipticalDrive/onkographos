# Camera-Calibrated Wave Propagation Model Training

Also called: Hardware-in-the-loop Training

## Overview
This method trains a learned neural network to replace analytical wave propagation models (such as Angular Spectrum Method, ASM) within the Phase-Aware Gradient Descent (PAGD) optimization loop. It solves the problem of numerical models failing to replicate real-world optical imperfections, including SLM non-linearities, aberrations, stray light, and limited diffraction efficiency [S1]. By capturing ground-truth multi-plane intensity data from a physical prototype, the model learns the true system behavior, yielding a 2 dB PSNR improvement over analytical baselines [S1].

## How it works
**Inputs:**
*   A dataset of phase patterns (holograms) and their corresponding captured multi-plane intensity images.
*   The dataset comprises 5,860 phase patterns: 2D phases generated via Stochastic Gradient Descent (SGD), higher-order gradient descent, and Double-Phase Amplitude Coding (DPAC), plus true 3D phases from the Flying Planes mesh-wave field-hologram dataset [S1].
*   The Flying Planes dataset contains 1,000 randomly generated 3D scenes in OBJ format, with corresponding complex-valued wave fields, RGB-Depth images, and phase-only holograms [S1]. These scenes are created by randomly sampling position, rotation, and scale of rectangular primitives with random textures from the DIV2K dataset [S1].

**Outputs:**
*   A trained neural network model that maps an input phase hologram to predicted multi-plane intensity distributions, calibrated to the specific hardware setup [S1].

**Data Structures and Layout:**
*   **Input Phase:** Phase-only hologram displayed on the SLM.
*   **Target Data:** Multi-plane intensity images captured at 8 specific axial distances from the SLM [S1].
*   **Region of Interest (ROI):** The captured images are cropped to an ROI of $880 \times 1600$ pixels, determined by laser uniformity and modulator efficiency [S1].
*   **Target Planes:** The 8 planes are located at distances $d \in \{10.00, 10.07, 10.14, 10.21, 10.28, 10.35, 10.42, 10.49\}$ cm from the SLM [S1].

**Algorithm Steps:**
1.  **Data Acquisition:** Display the 5,860 phase patterns on the prototype SLM. Capture the resulting diffraction patterns at the 8 target planes for each phase [S1].
2.  **Model Adaptation:** Adapt the learned wave propagation model architecture from Choi et al. [S1]. Add an extra low-pass filter to the architecture to suppress high-frequency phase noise [S1].
3.  **Preprocessing:** To handle anti-aliasing during any internal numerical operations (if hybrid), magnify the wave field by a factor of two in both dimensions and apply a fill factor of 0.93 [S1]. Grid interpolation is performed in PyTorch [S1].
4.  **Training Loop:**
    *   Initialize model parameters.
    *   For each epoch (total 20):
        *   Iterate through batches of size 1.
        *   Input the phase hologram into the model to predict intensities at the 8 planes.
        *   Compute the loss $L$ by comparing the scaled reconstructed field $u_r$ with the target field $u_t$ (captured intensities) [S1].
        *   Update weights using the Adam optimizer [S1].
5.  **Loss Function:** The final loss is computed by comparing the scaled reconstructed field with the target field [S1]. The specific formula is cited as being derived from Choi et al. [S1], but the exact mathematical expression is not provided in the source text beyond the variable definitions [S1].

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $0.0005$ [S1].
*   **Batch Size:** $1$ [S1].
*   **Epochs:** $20$ [S1].
*   **Hardware:** Single NVIDIA RTX 4090 GPU [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for training the Camera-Calibrated Wave Propagation Model
# Based on [S1]

def train_camera_calibrated_model(dataset, target_distances, roi_shape):
    """
    Trains the learned wave propagation model using hardware-captured data.
    
    Args:
        dataset: List of tuples (phase_hologram, captured_multi_plane_images)
        target_distances: List of float distances in cm [10.00, ..., 10.49]
        roi_shape: Tuple (880, 1600)
        
    Returns:
        trained_model: The neural network model
    """
    # 1. Initialize Model
    # Adapted from Choi et al. with extra low-pass filter [S1]
    model = LearnedWavePropagationModel(
        num_planes=len(target_distances),
        add_low_pass_filter=True
    )
    
    # 2. Setup Optimizer
    optimizer = Adam(model.parameters(), lr=0.0005)
    
    # 3. Training Loop
    epochs = 20
    batch_size = 1
    
    for epoch in range(epochs):
        # Iterate through dataset in batches of 1 [S1]
        for phase, captured_images in dataset:
            
            # Ensure ROI cropping if not done during capture [S1]
            # captured_images are already cropped to 880x1600 in acquisition [S1]
            
            # Forward pass: Predict intensities at 8 planes
            predicted_images = model(phase)
            
            # Compute Loss
            # L is computed by comparing scaled reconstructed field (predicted) 
            # with target field (captured) [S1]
            # Exact formula from Choi et al. [S1]
            loss = compute_loss(predicted_images, captured_images)
            
            # Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
    return model

def compute_loss(predicted, target):
    """
    Computes discrepancy between predicted and captured intensities.
    Formula details refer to Choi et al. [S1].
    """
    # Placeholder for the specific loss function from Choi et al. [S1]
    # Typically involves MSE or similar intensity comparison
    return torch.mean((predicted - target) ** 2)
```

**Edge Cases and Pitfalls:**
*   **SLM Non-linearities:** The model specifically addresses SLM non-linearities which analytical models miss [S1].
*   **High-Frequency Noise:** The added low-pass filter is critical to suppress high-frequency phase noise inherent in the captured data [S1].
*   **ROI Selection:** Using the full SLM resolution may introduce errors due to laser non-uniformity and modulator edge effects; the $880 \times 1600$ ROI mitigates this [S1].

## Subtopics

### Camera-Calibrated Learned Wave Propagation Model
This is the specific instance of the trained model. It is a neural network that captures real-world optical system behavior (aberrations, SLM non-linearities, stray light, limited diffraction efficiency) to replace numerical models like ASM in the PAGD optimization loop [S1]. It is set apart from analytical models by its ability to learn these physical imperfections directly from data, resulting in a 2 dB PSNR improvement over the analytical ASM model [S1]. It is chosen when high-fidelity reconstruction on a specific hardware prototype is required, particularly for handling tilted wave fields in 3D meshes [S1].

## Operating characteristics
*   **Performance:** The camera-calibrated learned model provides a PSNR improvement of 2 dB compared to the analytical model (ASM) [S1].
*   **Hardware:** Training is performed on a single NVIDIA RTX 4090 GPU [S1].
*   **Dataset Size:** The training dataset consists of 5,860 phase patterns [S1].
*   **Strengths:** Accurately replicates real-world optical systems including aberrations, SLM non-linearities, stray light, and limited diffraction efficiency, which analytical models struggle to capture [S1]. Superior reconstruction quality for 3D meshes, handling tilted wave fields effectively [S1].
*   **Limitations:** Requires a physical prototype and extensive data capture (5,860 phases across 8 planes) [S1]. The model is specific to the hardware setup (HOLOEYE Pluto-2 SLM, specific laser wavelengths) [S1].
*   **When to use:** When using PAGD for hologram synthesis on a specific hardware prototype where analytical propagation models yield insufficient quality due to hardware imperfections [S1].
*   **When not to use:** When a generic simulation is sufficient, or when hardware-specific calibration is not feasible [S1].

## Background
The method originates from research into mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays [S1]. It addresses the limitations of numerical models like ASM, which are proficient in simulations but fail to replicate real-world optical systems [S1]. The approach adapts the learned wave propagation model introduced by Choi et al. [S1]. The dataset used, Flying Planes, was created by randomly sampling position, rotation, and scale of rectangular primitives with random textures from the DIV2K dataset [S1]. The complex-valued wave field for the dataset was generated using the proposed POS-WRP, and phases were synthesized with various encoders including DPAC, SGD, and PAGD with ASM [S1].

## Sources
[S1] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
