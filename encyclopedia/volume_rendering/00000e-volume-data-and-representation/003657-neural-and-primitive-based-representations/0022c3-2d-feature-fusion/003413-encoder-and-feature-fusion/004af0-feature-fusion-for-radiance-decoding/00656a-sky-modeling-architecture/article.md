# Sky Modeling Architecture

Also called: Sky Modeling Network, Sky Modeling and Editing, SkyEncoder-SkyDecoder

## Overview
The Sky Modeling Architecture is an encoder-decoder neural network that compresses a Low Dynamic Range (LDR) sky panorama into a low-dimensional feature vector and decodes it into a High Dynamic Range (HDR) sky dome [S1]. It distinguishes itself from sibling methods like the Context-aware Radiance Field Decoder by explicitly predicting the sun's peak intensity and direction as part of the latent representation, enabling precise control and editing of the brightest light source [S1]. This explicit parameterization outperforms representations relying solely on latent vectors or explicit azimuths in terms of HDR reconstruction accuracy and peak direction estimation [S1].

## How it works

### Inputs and Outputs
*   **Input:** An LDR sky panorama $I_{LDR} \in \mathbb{R}^{3 \times H \times W}$ and a positional encoding tensor $P \in \mathbb{R}^{3 \times H \times W}$, where each pixel contains the unit direction vector for that pixel location in equi-rectangular projection [S1].
*   **Output:** An HDR sky dome $I_{HDR} \in \mathbb{R}^{3 \times 64 \times 256}$ [S1].
*   **Intermediate:** A sky feature vector $\hat{f} \in \mathbb{R}^{64}$ [S1].

### Data Structures

**Sky Feature Vector ($\hat{f}$)**
The vector $\hat{f}$ has a dimension of 64 and is structured into three components [S1]:
1.  **Peak Intensity ($\hat{f}_{intensity}$):** $\mathbb{R}^3$, representing the RGB peak intensity of the sun [S1].
2.  **Peak Direction ($\hat{f}_{dir}$):** $\mathbb{R}^3$, representing the 3D unit direction vector of the sun [S1].
3.  **Latent Code ($\hat{f}_{latent}$):** $\mathbb{R}^{58}$, encoding the remaining sky content [S1].

**Positional Encoding ($P$)**
A tensor of shape $\mathbb{R}^{3 \times H \times W}$ where each pixel $(u, v)$ contains the 3D unit vector $\mathbf{u}$ corresponding to the viewing direction of that pixel [S1].

### Network Architecture

The architecture consists of a Sky Encoder and a Sky Decoder (2D UNet).

**1. Sky Encoder**
The encoder comprises two separate 2D CNN branches sharing a ResNet50 backbone [S1].
*   **Branch 1 (Peak Information):** Predicts $\hat{f}_{intensity}$ and $\hat{f}_{dir}$.
*   **Branch 2 (Latent Code):** Predicts $\hat{f}_{latent}$.
*   **Structure:** Each branch consists of 5 downsampling conv-blocks.
*   **Channel Dimensions:** The intermediate output channel dimensions for the downsampling path are $(64, 128, 256, 256, 256)$ [S1].
*   **Conv-Block:** Contains two 2D convolution layers, followed by Batch Normalization and ReLU activation [S1].

**2. Sky Decoder (2D UNet)**
The decoder reconstructs the HDR sky from the encoded features.
*   **Input Construction:** The decoder takes a 7-channel input tensor constructed by concatenating:
    1.  **Peak Direction Encoding:** A 1-channel tensor $E_{dir} \in \mathbb{R}^{1 \times H \times W}$. For each pixel with direction $\mathbf{u}$, the value is computed as:
        $$ E_{dir}(\mathbf{u}) = e^{100(\mathbf{u} \cdot \hat{f}_{dir} - 1)} $$
        This creates a spherical Gaussian lobe centered at the predicted peak direction [S1].
    2.  **Peak Intensity Encoding:** A 3-channel tensor $E_{int} \in \mathbb{R}^{3 \times H \times W}$. The values are assigned as:
        $$ E_{int}(\mathbf{u}) = \begin{cases} \hat{f}_{intensity} & \text{if } E_{dir}(\mathbf{u}) \geq 0.98 \\ 0 & \text{otherwise} \end{cases} $$
        This isolates the peak intensity to the region around the sun [S1].
    3.  **Positional Encoding:** The original 3-channel $P$ tensor used in the encoder [S1].
*   **Structure:** 5 downsampling conv-blocks and 5 upsampling conv-blocks connected with residual links [S1].
*   **Channel Dimensions:** The intermediate output channel dimensions are $(64, 128, 256, 256, 256, 256, 128, 64, 32, 16)$ [S1].
*   **Latent Fusion:** The latent code $\hat{f}_{latent}$ is concatenated with the latent vector output by the 2D UNet (specifically at the bottleneck or corresponding latent layer) to jointly decode the final HDR sky dome [S1].

### Training Procedure

1.  **Pre-training:** The sky encoder-decoder is pre-trained on a dataset of outdoor HDR panoramas [S1].
2.  **Freezing:** After pre-training, the decoder weights are frozen [S1].
3.  **Integration:** The fixed decoder is integrated into the sky prediction branch of the larger hybrid lighting prediction network [S1].
4.  **Alternating Training:** During the training of the full system, the process alternates between:
    *   **End-to-end training:** Updating the encoder and the rest of the network.
    *   **Teacher forcing:** Using ground truth sky vectors to feed the frozen decoder, stabilizing the learning of the encoder [S1].
5.  **Pseudo-label Generation:** The pre-trained network is used to generate HDR pseudo-labels for supervision in scenarios lacking real HDR data [S1].

### Peak Editing
To edit the sky, the user modifies the explicit components of the sky vector $\hat{f}$ to create an edited vector $\hat{f}^{edit} = (\hat{f}_{intensity}^{edit}, \hat{f}_{dir}^{edit}, \hat{f}_{latent})$. This edited vector is fed into the frozen sky decoder to produce an edited HDR sky dome, effectively disentangling the sun peak from the background [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SkyModelingNetwork(nn.Module):
    def __init__(self, latent_dim=58, peak_dim=3):
        super().__init__()
        # Encoder: Two separate branches for peak info and latent code
        # Both use ResNet50 backbone followed by 5 downsampling conv-blocks
        self.encoder_peak = SkyEncoderBranch(channels=[64, 128, 256, 256, 256])
        self.encoder_latent = SkyEncoderBranch(channels=[64, 128, 256, 256, 256])
        
        # Decoder: 2D UNet
        self.decoder = SkyDecoderUNet(
            in_channels=7, # 1 (dir enc) + 3 (int enc) + 3 (pos enc)
            latent_dim=latent_dim,
            channels=[64, 128, 256, 256, 256, 256, 128, 64, 32, 16]
        )
        
        # Projection heads to map ResNet features to vector components
        self.head_intensity = nn.Linear(2048, peak_dim) # Assuming ResNet50 last dim
        self.head_direction = nn.Linear(2048, peak_dim)
        self.head_latent = nn.Linear(2048, latent_dim)

    def forward(self, ldr_panorama, positional_encoding):
        """
        Args:
            ldr_panorama: Tensor of shape (B, 3, H, W)
            positional_encoding: Tensor of shape (B, 3, H, W)
        Returns:
            hdr_sky: Tensor of shape (B, 3, 64, 256)
            sky_vector: Tuple (intensity, direction, latent)
        """
        # 1. Encode
        # Concatenate LDR and Pos Enc for encoder input
        encoder_input = torch.cat([ldr_panorama, positional_encoding], dim=1)
        
        # Extract features using ResNet50 backbone (implementation details abstracted)
        # In practice, this involves the shared ResNet50 backbone
        features = self.resnet_backbone(encoder_input)
        
        # Predict components
        intensity = self.head_intensity(features)
        direction = self.head_direction(features)
        latent = self.head_latent(features)
        
        # Normalize direction to unit vector
        direction = F.normalize(direction, dim=-1)
        
        sky_vector = (intensity, direction, latent)
        
        # 2. Decode
        # Construct 7-channel input for decoder
        peak_dir_enc = self.compute_peak_direction_encoding(direction, positional_encoding)
        peak_int_enc = self.compute_peak_intensity_encoding(intensity, peak_dir_enc)
        
        # Concatenate: 1 (dir) + 3 (int) + 3 (pos)
        decoder_input = torch.cat([peak_dir_enc, peak_int_enc, positional_encoding], dim=1)
        
        # Decode
        # The latent code is fused inside the decoder logic
        hdr_sky = self.decoder(decoder_input, latent)
        
        return hdr_sky, sky_vector

    def compute_peak_direction_encoding(self, direction, positional_encoding):
        """
        Computes the 1-channel peak direction encoding.
        PeakDirEncoding(u) = e^(100(u . f_dir - 1))
        """
        # direction: (B, 3)
        # positional_encoding: (B, 3, H, W)
        # Reshape direction for broadcasting
        dir_expanded = direction.view(-1, 3, 1, 1)
        
        # Dot product: (B, 3, 1, 1) * (B, 3, H, W) -> sum over dim 1
        dot_product = torch.sum(dir_expanded * positional_encoding, dim=1)
        
        # Formula
        encoding = torch.exp(100.0 * (dot_product - 1.0))
        
        # Return as 1-channel tensor
        return encoding.unsqueeze(1)

    def compute_peak_intensity_encoding(self, intensity, peak_dir_enc):
        """
        Computes the 3-channel peak intensity encoding.
        Assign intensity if PeakDirEncoding >= 0.98, else 0.
        """
        # intensity: (B, 3)
        # peak_dir_enc: (B, 1, H, W)
        
        # Create mask
        mask = (peak_dir_enc >= 0.98).float()
        
        # Expand intensity to match spatial dimensions
        intensity_expanded = intensity.view(-1, 3, 1, 1)
        
        # Apply mask
        encoding = intensity_expanded * mask
        
        return encoding

    def edit_sky(self, edited_intensity, edited_direction, edited_latent, positional_encoding):
        """
        Generates an HDR sky from an edited sky vector.
        """
        # Normalize edited direction
        edited_direction = F.normalize(edited_direction, dim=-1)
        
        # Compute encodings
        peak_dir_enc = self.compute_peak_direction_encoding(edited_direction, positional_encoding)
        peak_int_enc = self.compute_peak_intensity_encoding(edited_intensity, peak_dir_enc)
        
        # Concatenate
        decoder_input = torch.cat([peak_dir_enc, peak_int_enc, positional_encoding], dim=1)
        
        # Decode with frozen weights
        hdr_sky = self.decoder(decoder_input, edited_latent)
        
        return hdr_sky

class SkyEncoderBranch(nn.Module):
    def __init__(self, channels):
        super().__init__()
        # 5 downsampling conv-blocks
        self.blocks = nn.ModuleList()
        in_ch = 6 # 3 (LDR) + 3 (Pos Enc)
        for out_ch in channels:
            self.blocks.append(ConvBlock(in_ch, out_ch))
            in_ch = out_ch

    def forward(self, x):
        for block in self.blocks:
            x = block(x)
        # Global average pooling or flattening to feed into MLP head
        x = F.adaptive_avg_pool2d(x, 1)
        x = x.view(x.size(0), -1)
        return x

class ConvBlock(nn.Module):
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, kernel_size=3, padding=1)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.conv2 = nn.Conv2d(out_channels, out_channels, kernel_size=3, padding=1)
        self.bn2 = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)
        # Residual connection if dimensions match, else projection
        if in_channels == out_channels:
            self.residual = nn.Identity()
        else:
            self.residual = nn.Conv2d(in_channels, out_channels, kernel_size=1)

    def forward(self, x):
        identity = self.residual(x)
        out = self.conv1(x)
        out = self.bn1(out)
        out = self.relu(out)
        out = self.conv2(out)
        out = self.bn2(out)
        out = out + identity
        out = self.relu(out)
        return out

class SkyDecoderUNet(nn.Module):
    def __init__(self, in_channels, latent_dim, channels):
        super().__init__()
        # channels: [64, 128, 256, 256, 256, 256, 128, 64, 32, 16]
        # 5 downsampling, 5 upsampling
        self.down_blocks = nn.ModuleList()
        self.up_blocks = nn.ModuleList()
        
        # Downsampling
        current_ch = in_channels
        for i in range(5):
            self.down_blocks.append(ConvBlock(current_ch, channels[i]))
            current_ch = channels[i]
            
        # Bottleneck latent fusion
        self.latent_fusion = nn.Linear(latent_dim, channels[5]) # Assuming bottleneck dim matches channels[5] or similar
        
        # Upsampling
        for i in range(5, 10):
            # Skip connections from downsampling
            self.up_blocks.append(UpConvBlock(channels[i], channels[i-1]))
            
        self.final_conv = nn.Conv2d(channels[9], 3, kernel_size=1)

    def forward(self, x, latent_code):
        # Downsample
        skips = []
        for block in self.down_blocks:
            x = block(x)
            skips.append(x)
            
        # Latent fusion at bottleneck
        # Note: Exact fusion point (concatenation vs addition) is not fully specified, 
        # but "concatenated with the latent vector output by the 2D UNet" suggests fusion at a latent layer.
        # We assume fusion happens after the last downsample or at the bottleneck.
        # For this implementation, we fuse into the bottleneck representation.
        b, c, h, w = x.shape
        latent_expanded = latent_code.view(b, -1, 1, 1).expand(b, -1, h, w)
        x = torch.cat([x, latent_expanded], dim=1)
        
        # Upsample
        for i, block in enumerate(self.up_blocks):
            skip = skips[4-i]
            x = block(x, skip)
            
        x = self.final_conv(x)
        return x

class UpConvBlock(nn.Module):
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.up = nn.ConvTranspose2d(in_channels, out_channels, kernel_size=2, stride=2)
        self.conv = ConvBlock(out_channels * 2, out_channels) # Concat skip

    def forward(self, x, skip):
        x = self.up(x)
        x = torch.cat([x, skip], dim=1)
        x = self.conv(x)
        return x
```

## Operating characteristics

*   **Time Complexity:** The encoder uses a ResNet50 backbone followed by lightweight CNN branches, making feature extraction comparable to standard ResNet50 inference. The decoder is a 2D UNet, which is computationally efficient for image-to-image translation.
*   **Memory Complexity:** The sky feature vector is small ($\mathbb{R}^{64}$), allowing for efficient transmission and storage compared to full HDR environment maps. The decoder requires memory proportional to the image resolution ($64 \times 256$).
*   **Performance:** The method achieves an HDR reconstruction MSE of $7.49 \times 10^{-2}$ and a median peak direction angular error of $3.38^\circ$ [S1]. This outperforms the baseline using latent code with explicit azimuth, which has an MSE of $10.56 \times 10^{-2}$ and an error of $5.67^\circ$ [S1].
*   **Strengths:**
    *   Explicit control over sun peak intensity and direction enables easy editing [S1].
    *   Accurate reconstruction of extreme intensity values (sun peaks) [S1].
    *   Can generate HDR pseudo-labels for training other models when HDR data is scarce [S1].
*   **Limitations:**
    *   Relies on the quality of the LDR input; artifacts in the LDR panorama may propagate to the HDR output.
    *   The explicit peak representation assumes a dominant sun peak; complex overcast skies without a clear peak may be less accurately represented compared to purely latent methods [S1].
*   **When to use:** When HDR sky prediction from LDR images is required, especially if control over the sun's position and intensity is needed for virtual object insertion or lighting estimation [S1].
*   **When not to use:** If the scene has no dominant light source or if the sky is uniformly overcast, the explicit peak model may introduce artifacts.

## Background

The Sky Modeling Architecture was introduced in the paper "Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion" by Z. Wang et al. [S1]. The work addresses the challenge of estimating HDR lighting for street scenes to enable realistic virtual object insertion. A key component is the sky modeling network, which learns a low-dimensional feature space for the sky dome. The authors note that previous methods [9] used a latent vector with explicit azimuth, but their approach with explicit peak direction and intensity outperforms it in reconstruction accuracy and direction estimation [S1]. The architecture is part of a larger hybrid lighting prediction network that also estimates volumetric lighting [S1].

## Sources

[S1] Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion, https://doi.org/10.1007/978-3-031-20086-1_22, fetched 2026-10-30
