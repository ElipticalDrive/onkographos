# Differentiable Ocean Inverse Modeling

Also called: Dynamic ocean inverse modeling, Ocean surface reconstruction via differentable rendering

## Overview
Differentiable Ocean Inverse Modeling is a two-step framework that reconstructs dynamic ocean models from monocular video sequences by first estimating hierarchical surface geometry and then deriving constituent wave parameters [S1]. It distinguishes itself from direct inverse methods by leveraging a differentiable ocean renderer and a differentiable ocean model as self-supervised loss functions, allowing gradients to flow from image consistency and physical plausibility back to the neural network parameters [S1].

## How it works
The method operates in two distinct phases: hierarchical geometry reconstruction from images, followed by wave parameter estimation from the resulting geometry sequence [S1].

### Inputs and Outputs
*   **Input:** A sequence of monocular images $\{I_t\}_{n}$, where $t$ denotes the time step and $n$ is the number of frames [S1].
*   **Intermediate Output:** A sequence of hierarchical ocean geometry $\{G_t\}_{n}$, where each $G_t$ comprises the macrosurface normal $N_t$ and the microfacet roughness $R_t$ [S1].
*   **Final Output:** A set of constituent wave parameters for each time step, specifically amplitude $h_i$, angular frequency $\omega_i$, and wave direction $\theta_i$ for $N$ sampled wave trains [S1].

### Data Structures and Layouts
The method processes dense tensor fields and discrete parameter sets [S1].
1.  **Geometry Maps:** Dense fields for normal $N_t$ (3-channel vector per pixel) and roughness $R_t$ (scalar per pixel) [S1].
2.  **Wave Parameters:** A discrete set of $N$ tuples $\{h_i, \omega_i, \theta_i\}$ representing the superposed wave trains [S1].

### Step-by-Step Method

#### Step 1: Hierarchical Geometry Reconstruction
1.  **Input Processing:** For each image $I_t$ in the sequence, pass it through two separate encoder-decoder neural networks (sub-net1 and sub-net2) [S1].
2.  **Geometry Extraction:** Sub-net1 outputs the macrosurface normal map $N_t$, and sub-net2 outputs the microfacet roughness map $R_t$ [S1].
3.  **Sequence Assembly:** Combine the outputs to form the hierarchical geometry $G_t = \{N_t, R_t\}$ for all $t$, yielding the sequence $\{G_t\}_{n}$ [S1].

#### Step 2: Wave Parameter Estimation
1.  **Feature Extraction:** Pass the sequence of geometry maps $\{G_t\}_{n}$ through a UNet architecture to extract spatiotemporal features [S1].
2.  **Parameter Regression:** Feed the extracted features into a Multilayer Perceptron (MLP) network [S1].
3.  **Output Generation:** The MLP outputs the parameters for $N$ wave trains: amplitude $h_i$, angular frequency $\omega_i$, and direction $\theta_i$ [S1].

#### Step 3: Differentiable Ocean Modeling (Forward Model)
To enable self-supervised training, the method uses a differentiable ocean model to synthesize geometry from the estimated wave parameters [S1].

1.  **Wave Vector Definition:** For each wave $i$, define the wave vector $\mathbf{k}_i$ and wave number $k_i$ [S1]:
    $$
    \mathbf{k}_i = [k_i \cos \theta_i, k_i \sin \theta_i]
    $$
    $$
    k_i = \frac{\omega_i^2}{g}
    $$
    where $g$ is the acceleration due to gravity [S1]. The parameters are simplified from $\{h, \omega, k, \theta\}$ to $\{h, \omega, \theta\}$ [S1].

2.  **Nyquist Filtering and Weighting:** Apply a Nyquist filter to determine the contribution of each wave to the hierarchical geometry based on wavelength relative to pixel size [S1]:
    *   **Displacement (Long Waves):** Waves with wavelength $> \text{pixel size}$ affect macrosurface displacement [S1].
    *   **Normal (Medium/Long Waves):** Waves affecting the macrosurface normal [S1].
    *   **Roughness (Short/Sub-pixel Waves):** Waves with wavelength $< \text{pixel size}$ do not affect height but affect microfacet roughness [S1].

3.  **Superposition and Geometry Synthesis:** Superpose the $N$ filtered wave trains to generate the synthetic geometry sequence $\{\hat{G}_t\}_{n}$ [S1]. The ocean surface is modeled using a triangle mesh deformed by large waves at the grid level, medium-length waves added to the normal at the pixel level, and roughness estimated by measuring the slope variance of small waves at the microfacet level [S1].

#### Step 4: Differentiable Rendering
1.  **Rasterization:** Render the synthesized geometry $\hat{G}_t$ into an image $\hat{I}_t$ using the SoftRas method, which provides continuous and differentiable rasterization [S1].
2.  **Lighting Computation:** Compute reflected light from the Sun and sky dome, as well as refracted light from water [S1].
3.  **Displacement Application:** Use the computed displacements to alter geometry, normals, and roughness during the rendering process under given lighting conditions [S1].

#### Step 5: Loss Calculation and Optimization
The networks are trained self-supervised using the following loss functions [S1]:

1.  **Image Consistency Loss:**
    $$
    L_{\text{Image}} = \mathbb{E}_{I_t, \hat{I}_t} [\|I_t - \hat{I}_t\|^2]
    $$
    where $I_t$ is the input image and $\hat{I}_t$ is the re-rendered image [S1].

2.  **Discriminator Losses:** To enforce spatial distribution similarity and temporal coherence, use spatial discriminators $D_n$ (for normal) and $D_r$ (for roughness), and temporal discriminators $D_{tn}$ and $D_{tr}$ [S1].
    *   $L_{D_n}$: Loss from spatial discriminator on normal maps.
    *   $L_{D_r}$: Loss from spatial discriminator on roughness maps.
    *   $L_{D_{tn}}$: Loss from temporal discriminator on normal sequences.
    *   $L_{D_{tr}}$: Loss from temporal discriminator on roughness sequences.

3.  **Total Loss:**
    $$
    L_{\text{Total}} = \alpha L_{\text{Image}} + \beta_1 L_{D_n} + \beta_2 L_{D_r} + \gamma_1 L_{D_{tn}} + \gamma_2 L_{D_{tr}}
    $$
    where $\alpha, \beta_1, \beta_2, \gamma_1, \gamma_2$ are weighting hyperparameters [S1].

4.  **Geometry Consistency Loss:** The framework measures the consistency between the generated geometry $\hat{G}_t$ (from wave estimation) and the input geometry $G_t$ (from image reconstruction) to optimize the wave estimation network, but the exact formula is not explicitly defined in the sources [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import math

# Helper definitions to ensure self-contained, valid Python implementation

class EncoderDecoderNetwork(nn.Module):
    def __init__(self, channels=3):
        super().__init__()
        self.encoder = nn.Sequential(nn.Conv2d(channels, 64, 3, padding=1), nn.ReLU(),
                                     nn.Conv2d(64, 128, 3, padding=1), nn.ReLU())
        self.decoder = nn.Sequential(nn.Conv2d(128, 64, 3, padding=1), nn.ReLU(),
                                     nn.Conv2d(64, channels, 3, padding=1))
    def forward(self, x):
        return self.decoder(self.encoder(x))

class UNet(nn.Module):
    def __init__(self, in_ch=2, out_ch=256):
        super().__init__()
        self.conv = nn.Sequential(nn.Conv2d(in_ch, 64, 3, padding=1), nn.ReLU(),
                                  nn.Conv2d(64, 128, 3, padding=1), nn.ReLU())
        self.pool = nn.AdaptiveAvgPool2d(1)
        self.fc = nn.Linear(128, out_ch)
    def forward(self, x):
        return self.fc(self.pool(self.conv(x)))

class MLP(nn.Module):
    def __init__(self, in_ch=256, out_ch=3):
        super().__init__()
        self.net = nn.Sequential(nn.Linear(in_ch, 128), nn.ReLU(), nn.Linear(128, out_ch))
    def forward(self, x):
        return self.net(x)

class SpatialDiscriminator(nn.Module):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(nn.Conv2d(1, 32, 3, padding=1), nn.ReLU(), nn.AdaptiveAvgPool2d(1), nn.Flatten(), nn.Linear(32, 1))
    def forward(self, x, y):
        return self.net(x) - self.net(y)

class TemporalDiscriminator(nn.Module):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(nn.Conv2d(1, 32, 3, padding=1), nn.ReLU(), nn.AdaptiveAvgPool2d(1), nn.Flatten(), nn.Linear(32, 1))
    def forward(self, seq_x, seq_y):
        return self.net(seq_x) - self.net(seq_y)

def wavelength_from_k(k):
    return 2 * math.pi / k

def apply_nyquist_filter_displacement(wavelength, pixel_size=1.0):
    return torch.where(wavelength > pixel_size, torch.ones_like(wavelength), torch.zeros_like(wavelength))

def apply_nyquist_filter_normal(wavelength, pixel_size=1.0):
    return torch.where(wavelength > pixel_size * 0.5, torch.ones_like(wavelength), torch.zeros_like(wavelength))

def apply_nyquist_filter_roughness(wavelength, pixel_size=1.0):
    return torch.where(wavelength <= pixel_size, torch.ones_like(wavelength), torch.zeros_like(wavelength))

def differentiable_ocean_model_superpose(h, kx, ky, w_disp, w_norm, w_rough, t, grid_res=64):
    # Simplified superposition: sum of trochoid displacements and normal/roughness contributions
    # In practice, this computes mesh deformation and pixel-level normal/roughness variance
    displacement = torch.zeros(grid_res, grid_res)
    normal = torch.zeros(grid_res, grid_res, 3)
    roughness = torch.zeros(grid_res, grid_res)
    for i in range(h.shape[0]):
        amp = h[i] * w_disp[i]
        phase = kx[i] * torch.arange(grid_res).unsqueeze(1) + ky[i] * torch.arange(grid_res).unsqueeze(0) + t
        disp = amp * torch.sin(phase)
        displacement += disp
        normal[..., 0] += w_norm[i] * amp * torch.cos(phase)
        roughness += w_rough[i] * amp**2
    return normal, roughness

def soft_ras_rasterize(normal, roughness, extrinsics, lighting):
    # Placeholder for SoftRas differentiable rasterization
    # Computes reflected/refracted lighting based on normal, roughness, and camera/lighting
    return torch.rand_like(normal[..., 0])

def compute_discriminator_loss(disc, real, fake):
    return torch.mean(disc(real, fake)**2)

class DifferentiableOceanInverseModeling:
    def __init__(self, num_wave_trains, learning_rate=1e-4):
        self.geo_encoder_decoder_normal = EncoderDecoderNetwork(channels=3)
        self.geo_encoder_decoder_roughness = EncoderDecoderNetwork(channels=1)
        self.wave_feature_extractor = UNet(in_ch=2)
        self.wave_parameter_mlp = MLP(in_ch=256)
        self.disc_normal = SpatialDiscriminator()
        self.disc_roughness = SpatialDiscriminator()
        self.disc_temporal_normal = TemporalDiscriminator()
        self.disc_temporal_roughness = TemporalDiscriminator()
        self.alpha = 1.0
        self.beta1 = 0.1
        self.beta2 = 0.1
        self.gamma1 = 0.1
        self.gamma2 = 0.1
        self.num_wave_trains = num_wave_trains
        self.optimizer = optim.Adam(
            list(self.geo_encoder_decoder_normal.parameters()) +
            list(self.geo_encoder_decoder_roughness.parameters()) +
            list(self.wave_feature_extractor.parameters()) +
            list(self.wave_parameter_mlp.parameters()) +
            list(self.disc_normal.parameters()) +
            list(self.disc_roughness.parameters()) +
            list(self.disc_temporal_normal.parameters()) +
            list(self.disc_temporal_roughness.parameters()),
            lr=learning_rate
        )

    def reconstruct_geometry(self, image):
        N_t = self.geo_encoder_decoder_normal(image)
        R_t = self.geo_encoder_decoder_roughness(image)
        return N_t, R_t

    def estimate_wave_parameters(self, geometry_sequence):
        features = self.wave_feature_extractor(geometry_sequence)
        params = self.wave_parameter_mlp(features)
        return params

    def synthesize_geometry_from_waves(self, wave_params, time_step):
        h = wave_params[:, 0]
        omega = wave_params[:, 1]
        theta = wave_params[:, 2]
        g = 9.81
        k = (omega ** 2) / g
        kx = k * torch.cos(theta)
        ky = k * torch.sin(theta)
        wl = wavelength_from_k(k)
        w_disp = apply_nyquist_filter_displacement(wl)
        w_norm = apply_nyquist_filter_normal(wl)
        w_rough = apply_nyquist_filter_roughness(wl)
        N_hat, R_hat = differentiable_ocean_model_superpose(
            h=h, kx=kx, ky=ky, w_disp=w_disp, w_norm=w_norm, w_rough=w_rough, t=time_step
        )
        return N_hat, R_hat

    def render_image(self, geometry, extrinsics, lighting):
        N, R = geometry
        I_hat = soft_ras_rasterize(N, R, extrinsics, lighting)
        return I_hat

    def compute_loss(self, I_t, I_hat_t, N_t, N_hat_t, R_t, R_hat_t, 
                     N_seq_t, N_hat_seq_t, R_seq_t, R_hat_seq_t):
        L_image = torch.mean((I_t - I_hat_t) ** 2)
        L_Dn = compute_discriminator_loss(self.disc_normal, N_t, N_hat_t)
        L_Dr = compute_discriminator_loss(self.disc_roughness, R_t, R_hat_t)
        L_Dtn = compute_discriminator_loss(self.disc_temporal_normal, N_seq_t, N_hat_seq_t)
        L_Dtr = compute_discriminator_loss(self.disc_temporal_roughness, R_seq_t, R_hat_seq_t)
        L_total = (self.alpha * L_image + 
                   self.beta1 * L_Dn + 
                   self.beta2 * L_Dr + 
                   self.gamma1 * L_Dtn + 
                   self.gamma2 * L_Dtr)
        return L_total

    def train_step(self, image_sequence, extrinsics, lighting):
        self.optimizer.zero_grad()
        batch_size = image_sequence.shape[0]
        total_loss = 0
        for t in range(batch_size):
            I_t = image_sequence[t]
            N_t, R_t = self.reconstruct_geometry(I_t)
            G_t = torch.cat([N_t, R_t], dim=1)
            wave_params = self.estimate_wave_parameters(G_t)
            N_hat_t, R_hat_t = self.synthesize_geometry_from_waves(wave_params, t)
            I_hat_t = self.render_image((N_hat_t, R_hat_t), extrinsics, lighting)
            loss = self.compute_loss(I_t, I_hat_t, N_t, N_hat_t, R_t, R_hat_t,
                                     N_t, N_hat_t, R_t, R_hat_t)
            total_loss += loss
        total_loss.backward()
        self.optimizer.step()
```

### Edge Cases and Pitfalls
*   **Aliasing:** Sub-pixel waves (wavelength $< \text{pixel size}$) cause signal aliasing if not filtered [S1]. The Nyquist filter is critical to prevent these waves from affecting surface height, restricting them to microfacet roughness [S1].
*   **Viewing Distance:** The impact of waves depends on viewing distance [S1]. The method assumes a specific relationship between wavelength and pixel size to categorize waves into displacement, normal, and roughness effects [S1].
*   **Differentiability:** The entire pipeline, including rasterization (SoftRas) and ocean superposition, must be differentiable to propagate gradients from the image loss to the network weights [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Self-supervised; does not require ground-truth wave parameters or geometry for training; leverages physical ocean models for plausibility [S1].
*   **Limitations:** Relies on the accuracy of the differentiable ocean model and renderer; sensitive to the quality of the initial geometry reconstruction; requires careful tuning of discriminator weights [S1].
*   **When to use:** When reconstructing dynamic ocean scenes from monocular video where ground-truth 3D data is unavailable [S1].
*   **When not to use:** When high-fidelity ground-truth geometry is available and supervised learning is preferred [G].

## Background
*   **Origin:** The method was proposed in the paper "Dynamic ocean inverse modeling based on differentiable rendering" [S1].
*   **History:** It builds upon differentiable rendering techniques (SoftRas) and hierarchical ocean geometry models (displacement, normal, roughness) [S1]. It addresses the infeasibility of direct wave estimation from images by using a two-step inverse approach [S1].
*   **Usage:** Used for dynamic ocean reconstruction from monocular video sequences [S1].

## Sources
[S1] Dynamic ocean inverse modeling based on differentiable rendering, https://doi.org/10.1007/s41095-023-0338-4, fetched 2026-10-26
