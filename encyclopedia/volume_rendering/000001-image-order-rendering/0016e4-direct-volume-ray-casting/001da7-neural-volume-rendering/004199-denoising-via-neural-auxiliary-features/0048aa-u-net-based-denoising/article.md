# U-Net Based Denoising

Also called: Convolutional Denoiser, Deep Learning Enhancement (U-Net), Denoiser, U-Net Denoiser, U-Net Post-processing

## Overview
A U-Net based denoiser is a convolutional neural network used to enhance reconstructed volumes or images by removing noise and artifacts [S1, unofficial]. It operates as a post-processing step, taking a noisy input to predict a clean output [S1, unofficial]. It sets itself apart from kernel-prediction methods by using a fixed encoder-decoder architecture rather than predicting spatially varying kernels [G]. It builds on the shared principles of neural auxiliary feature denoising [Parent](10536-denoising-via-neural-auxiliary-features.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A noisy volume or image slice normalized to the range $[0, 1]$ [S1, unofficial].
*   **Output:** A denoised volume or image slice [S1, unofficial].

**Data Structures:**
The model is a standard 2D U-Net architecture [S1, unofficial].
*   **Encoder (`enc`):** Sequential block of convolutional layers.
    *   Layer 1: `Conv2d(1, 32, kernel_size=3, stride=1, padding=1)` followed by `ReLU` [S1, unofficial].
    *   Layer 2: `Conv2d(32, 64, kernel_size=3, stride=2, padding=1)` followed by `ReLU` [S1, unofficial].
*   **Decoder (`dec`):** Sequential block of transposed convolutional and convolutional layers.
    *   Layer 1: `ConvTranspose2d(64, 32, kernel_size=2, stride=2)` followed by `ReLU` [S1, unofficial].
    *   Layer 2: `Conv2d(32, 1, kernel_size=1)` [S1, unofficial].
*   **Parameters:** The model weights are stored in a file (e.g., `denoiser.pt`) [S1, unofficial].

**Steps:**
1.  Load the pre-trained U-Net model from the specified model path (default `denoiser.pt`) [S1, unofficial].
2.  Normalize the input volume data to the range $[0, 1]$ using $v_{\text{norm}} = (v - v_{\min}) / (v_{\max} - v_{\min})$ [S1, unofficial].
3.  Process the volume slice-by-slice along the axial direction in inference mode [S1, unofficial].
4.  For each slice:
    a. Convert the slice to a tensor [S1, unofficial].
    b. Add batch and channel dimensions (shape becomes $1 \times 1 \times H \times W$) [S1, unofficial].
    c. Pass the tensor through the model: apply the encoder `enc` to input $x$, then pass the result through the decoder [S1, unofficial].
    d. Store the output slice in the enhanced volume buffer [S1, unofficial].
5.  Normalize the enhanced volume and save the result [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn

class UNet(nn.Module):
    def __init__(self):
        super().__init__()
        # Encoder: Conv2d(1,32,3,1,1)+ReLU, Conv2d(32,64,3,2,1)+ReLU
        self.enc = nn.Sequential(
            nn.Conv2d(1, 32, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.Conv2d(32, 64, kernel_size=3, stride=2, padding=1),
            nn.ReLU()
        )
        # Decoder: ConvTranspose2d(64,32,2,2)+ReLU, Conv2d(32,1,1)
        self.dec = nn.Sequential(
            nn.ConvTranspose2d(64, 32, kernel_size=2, stride=2),
            nn.ReLU(),
            nn.Conv2d(32, 1, kernel_size=1)
        )

    def forward(self, x):
        # x shape: (batch, 1, H, W)
        encoded = self.enc(x)
        output = self.dec(encoded)
        return output

def denoise_volume(volume_data, model_path='denoiser.pt'):
    # 1. Load model
    model = UNet()
    model.load_state_dict(torch.load(model_path, weights_only=True))
    model.eval()

    # 2. Normalize input to [0, 1]
    vol_min = volume_data.min()
    vol_max = volume_data.max()
    normalized_vol = (volume_data - vol_min) / (vol_max - vol_min)

    enhanced_vol = np.zeros_like(normalized_vol)

    # 3. Process slice-by-slice (axial direction)
    for z in range(normalized_vol.shape[0]):
        slice_2d = normalized_vol[z, :, :]
        
        # 4a. Convert to tensor and add dims
        tensor_input = torch.from_numpy(slice_2d).float().unsqueeze(0).unsqueeze(0)
        
        # 4b. Forward pass
        with torch.no_grad():
            output_tensor = model(tensor_input)
            
        # 4c. Store output
        enhanced_vol[z, :, :] = output_tensor.squeeze().numpy()

    # 5. Rescale back to original intensity range
    final_vol = enhanced_vol * (vol_max - vol_min) + vol_min
    return final_vol
```

**Parameters:**
*   `MODEL_PATH`: Path to the trained model weights. Default: `'denoiser.pt'` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Effects:** Slice-by-slice processing may introduce discontinuities at slice boundaries if the network does not account for 3D context [G].
*   **Input Dimensions:** The encoder uses stride 2, which halves spatial dimensions. Input slices must have even dimensions or appropriate padding to avoid size mismatches in the decoder [G].

## Subtopics

### Post-Processing
Uses a standard three-layer U-Net architecture with a single residual block based on Fast Fourier Convolution (FFC) to enhance the reconstruction of high-frequency details [S2]. This variant is chosen when high-frequency detail preservation is critical in implicit neural point cloud rendering [S2].

### Post-processing Network D_θ
A denoiser network $D_\theta$ trained in a supervised fashion for artifact removal within the RED framework [S3]. It is constructed as a non-expansive operator by unrolling the network $q$ times: $D_\theta^{(q)}(\tilde{f}) = [D_\theta \circ \dots \circ D_\theta](\tilde{f})$ [S3]. Training minimizes the distance between each iteration and the ground truth: $L_D(\theta) = \frac{1}{K} \sum_{i=1}^K \sum_{j=1}^q \frac{1}{n} \|D_\theta^{(j)}(\tilde{f}_i) - f^\dagger_i\|_2^2$ [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and slices processed [G].
*   **Memory Complexity:** Proportional to the model size and the input batch size [G]. The specific U-Net described has a small footprint due to its shallow depth (2 encoder layers, 2 decoder layers) [S1, unofficial].
*   **Performance:** The method effectively removes noise and artifacts from reconstructed CT volumes [S1, unofficial]. It is often combined with classical methods like FDK or iterative reconstruction (SIRT) to improve final quality [S1, unofficial].
*   **Strengths:** Simple architecture, easy to implement, effective at removing structured noise and artifacts [S1, unofficial].
*   **Limitations:** Requires training data for effective performance; may blur fine details if not properly regularized [G]. Slice-by-slice processing may lack 3D coherence [G].
*   **When to use:** When high-quality denoising is required for medical imaging (CT) and computational resources allow for GPU inference [S1, unofficial].
*   **When not to use:** When real-time performance is critical and GPU inference is not available, or when 3D coherence is more important than 2D slice quality [G].

## Background
The U-Net architecture was originally introduced for biomedical image segmentation [G]. In the context of volume rendering and CT reconstruction, it has been adapted as a deep learning enhancement step to clean up outputs from classical reconstruction algorithms like Filtered Back Projection (FDK) or Iterative Reconstruction (SIRT) [S1, unofficial]. It is used in industrial imaging systems (X-ray CT, NDE) to handle noise, sparse views, and physics constraints [S1, unofficial]. The specific implementation described in [S1, unofficial] is part of a GPU-accelerated, physics-informed CT reconstruction pipeline [S1, unofficial]. The variant using FFC is applied in implicit neural point cloud rendering for radiance field rendering [S2]. The non-expansive unrolled variant is used in neural field regularization for sparse-view X-ray CT [S3].

## Sources
[S1] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
[S2] INPC: Implicit Neural Point Clouds for Radiance Field Rendering, https://doi.org/10.1109/3dv66043.2025.00021, fetched 2026-10-26
[S3] Neural field regularization by denoising for 3D sparse-view X-ray computed tomography, https://doi.org/10.1109/3DV62453.2024.00094, fetched 2026-10-26
