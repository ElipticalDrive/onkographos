# Differentiable Ocean Inverse Modeling

Also called: Dynamic ocean inverse modeling, Ocean surface reconstruction via differentiable rendering

## Overview
Differentiable Ocean Inverse Modeling is a two-step framework that reconstructs dynamic ocean models from monocular video sequences by first estimating hierarchical surface geometry and then deriving constituent wave parameters [S1]. It distinguishes itself from direct inverse methods by leveraging a differentiable ocean renderer and a differentiable ocean model as self-supervised loss functions, allowing gradients to flow from image consistency and physical plausibility back to the neural network parameters [S1].

## How it works
The method operates in two distinct phases: hierarchical geometry reconstruction from images, followed by wave parameter estimation from the resulting geometry sequence [S1].

### Inputs and Outputs
*   **Input:** A sequence of monocular images $\{I_t\}_{n}$, where $t$ denotes the time step and $n$ is the number of frames [S1].
*   **Intermediate Output:** A sequence of hierarchical ocean geometry $\{G_t\}_{n}$, where each $G_t$ comprises the macrosurface normal $N_t$ and the microfacet roughness $R_t$ [S1].
*   **Final Output:** A set of constituent wave parameters for each time step, specifically amplitude $h_i$, angular frequency $\omega_i$, and wave direction $\theta_i$ for $N$ sampled wave trains [S1].

### Data Structures and Layouts
The method relies on neural network weights and intermediate feature maps rather than explicit static data structures. The core data flow involves:
1.  **Geometry Maps:** Dense fields for normal $N_t$ (3-channel vector per pixel) and roughness $R_t$ (scalar per pixel) [S1].
2.  **Wave Parameters:** A discrete set of $N$ tuples $\{h_i, \omega_i, \theta_i\}$ representing the superposed wave trains [S1].

### Step-by-Step Method

#### Step 1: Hierarchical Geometry Reconstruction
1.  **Input Processing:** For each image $I_t$ in the sequence, pass it through two separate encoder-decoder neural networks (sub-net1 and sub-net2) [S1].
2.  **Geometry Extraction:**
    *   Sub-net1 outputs the macrosurface normal map $N_t$ [S1].
    *   Sub-net2 outputs the microfacet roughness map $R_t$ [S1].
    *   Combine these to form the hierarchical geometry $G_t = \{N_t, R_t\}$ [S1].
3.  **Sequence Assembly:** Repeat for all $t$ to obtain the geometry sequence $\{G_t\}_{n}$ [S1].

#### Step 2: Wave Parameter Estimation
1.  **Feature Extraction:** Pass the sequence of geometry maps $\{G_t\}_{n}$ through a UNet architecture to extract spatiotemporal features [S1].
2.  **Parameter Regression:** Feed the extracted features into a Multilayer Perceptron (MLP) network [S1].
3.  **Output Generation:** The MLP outputs the parameters for $N$ wave trains: amplitude $h_i$, angular frequency $\omega_i$, and direction $\theta_i$ [S1].

#### Step 3: Differentiable Ocean Modeling (Forward Model)
To enable self-supervised training, the method uses a differentiable ocean model to synthesize geometry from the estimated wave parameters [S1].

1.  **Wave Vector Definition:**
    For each wave $i$, define the wave vector $\mathbf{k}_i$ and wave number $k_i$ [S1]:
    $$
    \mathbf{k}_i = [k_i \cos \theta_i, k_i \sin \theta_i]
    $$
    $$
    k_i = \frac{\omega_i^2}{g}
    $$
    where $g$ is the acceleration due to gravity [S1]. The parameters are simplified from $\{h, \omega, k, \theta\}$ to $\{h, \omega, \theta\}$ [S1].

2.  **Nyquist Filtering and Weighting:**
    Apply a Nyquist filter to determine the contribution of each wave to the hierarchical geometry based on wavelength relative to pixel size [S1]:
    *   **Displacement (Long Waves):** Waves with wavelength $> \text{pixel size}$ affect macrosurface displacement. Weighted by a "blue" filter function [S1].
    *   **Normal (Medium/Long Waves):** Waves affecting the macrosurface normal. Weighted by a "red" filter function [S1].
    *   **Roughness (Short/Sub-pixel Waves):** Waves with wavelength $< \text{pixel size}$ do not affect height but affect microfacet roughness. Weighted by a "green" filter function [S1] [G: The specific mathematical form of the Nyquist filter weights is not provided in the source; implementation requires referencing Bruneton et al. as cited in [S1]].

3.  **Superposition and Geometry Synthesis:**
    Superpose the $N$ filtered wave trains to generate the synthetic geometry sequence $\{\hat{G}_t\}_{n}$ [S1]. The ocean surface is modeled using:
    *   **Grid Level:** A triangle mesh deformed by large waves [S1].
    *   **Pixel Level:** Medium-length waves added to the normal computed from the mesh [S1].
    *   **Microfacet Level:** Roughness estimated by measuring the slope variance of small waves [S1].

#### Step 4: Differentiable Rendering
1.  **Rasterization:** Render the synthesized geometry $\hat{G}_t$ into an image $\hat{I}_t$ using the SoftRas method, which provides continuous and differentiable rasterization [S1].
2.  **Lighting Computation:** Compute reflected light (Sun and sky dome) and refracted light from water [S1].
3.  **Displacement Application:** Use the computed displacements to alter geometry, normals, and roughness during the rendering process under given lighting conditions [S1].

#### Step 5: Loss Calculation and Optimization
The networks are trained self-supervised using the following loss functions [S1]:

1.  **Image Consistency Loss:**
    $$
    L_{\text{Image}} = \mathbb{E}_{I_t, \hat{I}_t} [\|I_t - \hat{I}_t\|^2]
    $$
    where $I_t$ is the input image and $\hat{I}_t$ is the re-rendered image [S1].

2.  **Discriminator Losses:**
    To enforce spatial distribution similarity and temporal coherence, use spatial discriminators $D_n$ (for normal) and $D_r$ (for roughness), and temporal discriminators $D_{tn}$ and $D_{tr}$ [S1].
    *   $L_{D_n}$: Loss from spatial discriminator on normal maps.
    *   $L_{D_r}$: Loss from spatial discriminator on roughness maps.
    *   $L_{D_{tn}}$: Loss from temporal discriminator on normal sequences.
    *   $L_{D_{tr}}$: Loss from temporal discriminator on roughness sequences.

3.  **Total Loss:**
    $$
    L_{\text{Total}} = \alpha L_{\text{Image}} + \beta_1 L_{D_n} + \beta_2 L_{D_r} + \gamma_1 L_{D_{tn}} + \gamma_2 L_{D_{tr}}
    $$
    where $\alpha, \beta_1, \beta_2, \gamma_1, \gamma_2$ are weighting hyperparameters [S1].

4.  **Geometry Consistency Loss (Implicit):**
    The framework also measures the consistency between the generated geometry $\hat{G}_t$ (from wave estimation) and the input geometry $G_t$ (from image reconstruction) to optimize the wave estimation network [S1]. The exact formula for this geometric consistency term is not explicitly defined in the provided text, but it is part of the training loop [G: In practice, this likely involves an $L_2$ or perceptual loss between $N_t$ and $\hat{N}_t$, and $R_t$ and $\hat{R}_t$].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Placeholder for external differentiable ocean model and renderer
# These must be implemented based on the specific differentiable rendering 
# library (e.g., SoftRas) and ocean physics model described in [S1].

class DifferentiableOceanInverseModeling:
    def __init__(self, num_wave_trains, learning_rate=1e-4):
        """
        Initialize the inverse modeling framework.
        
        Args:
            num_wave_trains (int): N, number of constituent wave trains to sample.
            learning_rate (float): Learning rate for optimizer.
        """
        # Step 1: Geometry Reconstruction Networks
        # Two encoder-decoder networks
        self.geo_encoder_decoder_normal = EncoderDecoderNetwork() # Sub-net1
        self.geo_encoder_decoder_roughness = EncoderDecoderNetwork() # Sub-net2
        
        # Step 2: Wave Parameter Estimation Network
        # UNet for feature extraction + MLP for parameters
        self.wave_feature_extractor = UNet()
        self.wave_parameter_mlp = MLP()
        
        # Discriminators
        self.disc_normal = SpatialDiscriminator()
        self.disc_roughness = SpatialDiscriminator()
        self.disc_temporal_normal = TemporalDiscriminator()
        self.disc_temporal_roughness = TemporalDiscriminator()
        
        # Hyperparameters
        self.alpha = 1.0
        self.beta1 = 0.1
        self.beta2 = 0.1
        self.gamma1 = 0.1
        self.gamma2 = 0.1
        self.num_wave_trains = num_wave_trains
        
        # Optimizer
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
        """
        Reconstruct hierarchical geometry from a single image.
        
        Args:
            image (Tensor): Input image I_t.
            
        Returns:
            Tuple[Tensor, Tensor]: Normal map N_t, Roughness map R_t.
        """
        N_t = self.geo_encoder_decoder_normal(image)
        R_t = self.geo_encoder_decoder_roughness(image)
        return N_t, R_t

    def estimate_wave_parameters(self, geometry_sequence):
        """
        Estimate wave parameters from a sequence of geometry maps.
        
        Args:
            geometry_sequence (Tensor): Sequence of {G_t} containing N_t and R_t.
            
        Returns:
            Tensor: Wave parameters [h, omega, theta] for N waves.
        """
        features = self.wave_feature_extractor(geometry_sequence)
        params = self.wave_parameter_mlp(features)
        return params

    def synthesize_geometry_from_waves(self, wave_params, time_step):
        """
        Synthesize geometry using the differentiable ocean model.
        
        Args:
            wave_params (Tensor): Amplitude, frequency, direction for N waves.
            time_step (float): Current time t.
            
        Returns:
            Tuple[Tensor, Tensor]: Synthetic Normal map N_hat, Synthetic Roughness R_hat.
        """
        # Extract h, omega, theta from wave_params
        h = wave_params[:, 0]
        omega = wave_params[:, 1]
        theta = wave_params[:, 2]
        
        # Calculate wave vectors
        g = 9.81 # Gravity
        k = (omega ** 2) / g
        kx = k * torch.cos(theta)
        ky = k * torch.sin(theta)
        
        # Apply Nyquist filters to determine weights for displacement, normal, roughness
        # Note: Specific filter functions from Bruneton et al. are required here [S1]
        w_disp = apply_nyquist_filter_displacement(wavelength_from_k(k))
        w_norm = apply_nyquist_filter_normal(wavelength_from_k(k))
        w_rough = apply_nyquist_filter_roughness(wavelength_from_k(k))
        
        # Superpose waves to generate hierarchical geometry
        # This involves mesh deformation for large waves and pixel-level normal/roughness addition
        N_hat, R_hat = differentiable_ocean_model_superpose(
            h=h, kx=kx, ky=ky, w_disp=w_disp, w_norm=w_norm, w_rough=w_rough, t=time_step
        )
        return N_hat, R_hat

    def render_image(self, geometry, extrinsics, lighting):
        """
        Render an image from geometry using SoftRas.
        
        Args:
            geometry (Tuple[Tensor, Tensor]): Normal and Roughness maps.
            extrinsics (Tensor): Camera extrinsic parameters.
            lighting (Tensor): Lighting conditions.
            
        Returns:
            Tensor: Rendered image I_hat_t.
        """
        N, R = geometry
        # SoftRas differentiable rasterization
        I_hat = soft_ras_rasterize(N, R, extrinsics, lighting)
        return I_hat

    def compute_loss(self, I_t, I_hat_t, N_t, N_hat_t, R_t, R_hat_t, 
                     N_seq_t, N_hat_seq_t, R_seq_t, R_hat_seq_t):
        """
        Compute the total loss function.
        
        Args:
            I_t (Tensor): Input image.
            I_hat_t (Tensor): Rendered image.
            N_t (Tensor): Reconstructed normal.
            N_hat_t (Tensor): Synthetic normal from waves.
            R_t (Tensor): Reconstructed roughness.
            R_hat_t (Tensor): Synthetic roughness from waves.
            N_seq_t (Tensor): Sequence of reconstructed normals.
            N_hat_seq_t (Tensor): Sequence of synthetic normals.
            R_seq_t (Tensor): Sequence of reconstructed roughness.
            R_hat_seq_t (Tensor): Sequence of synthetic roughness.
            
        Returns:
            Tensor: Total loss.
        """
        # Image Loss
        L_image = torch.mean((I_t - I_hat_t) ** 2)
        
        # Spatial Discriminator Losses
        L_Dn = compute_discriminator_loss(self.disc_normal, N_t, N_hat_t)
        L_Dr = compute_discriminator_loss(self.disc_roughness, R_t, R_hat_t)
        
        # Temporal Discriminator Losses
        L_Dtn = compute_discriminator_loss(self.disc_temporal_normal, N_seq_t, N_hat_seq_t)
        L_Dtr = compute_discriminator_loss(self.disc_temporal_roughness, R_seq_t, R_hat_seq_t)
        
        # Total Loss
        L_total = (self.alpha * L_image + 
                   self.beta1 * L_Dn + 
                   self.beta2 * L_Dr + 
                   self.gamma1 * L_Dtn + 
                   self.gamma2 * L_Dtr)
        
        return L_total

    def train_step(self, image_sequence, extrinsics, lighting):
        """
        Perform one training step.
        
        Args:
            image_sequence (Tensor): Sequence of input images {I_t}.
            extrinsics (Tensor): Camera extrinsics.
            lighting (Tensor): Lighting conditions.
        """
        self.optimizer.zero_grad()
        
        batch_size = image_sequence.shape[0]
        total_loss = 0
        
        for t in range(batch_size):
            I_t = image_sequence[t]
            
            # Step 1: Reconstruct Geometry
            N_t, R_t = self.reconstruct_geometry(I_t)
            
            # Step 2: Estimate Wave Parameters (using sequence context)
            # Note: In practice, wave estimation uses the whole sequence or a window
            # Here simplified to single frame for structure, but source implies sequence input
            G_t = torch.stack([N_t, R_t], dim=1)
            wave_params = self.estimate_wave_parameters(G_t) # Simplified
            
            # Step 3: Synthesize Geometry from Waves
            N_hat_t, R_hat_t = self.synthesize_geometry_from_waves(wave_params, t)
            
            # Step 4: Render Image
            I_hat_t = self.render_image((N_hat_t, R_hat_t), extrinsics, lighting)
            
            # Step 5: Compute Loss
            # Note: For temporal losses, we need sequences. 
            # This loop structure is simplified; full implementation requires 
            # batching sequences for temporal discriminators.
            loss = self.compute_loss(I_t, I_hat_t, N_t, N_hat_t, R_t, R_hat_t,
                                     N_t, N_hat_t, R_t, R_hat_t) # Dummy seq args for structure
            total_loss += loss
            
        total_loss.backward()
        self.optimizer.step()
```

### Edge Cases and Pitfalls
*   **Aliasing:** Sub-pixel waves (wavelength < pixel size) cause signal aliasing if not filtered. The Nyquist filter is critical to prevent these waves from affecting surface height, restricting them to microfacet roughness [S1].
*   **Viewing Distance:** The impact of waves depends on viewing distance. The method assumes a specific relationship between wavelength and pixel size to categorize waves into displacement, normal, and roughness effects [S1].
*   **Differentiability:** The entire pipeline, including rasterization (SoftRas) and ocean superposition, must be differentiable to propagate gradients from the image loss to the network weights [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Depends on the depth of the encoder-decoder networks, the UNet, the MLP, and the cost of the differentiable rasterization and ocean superposition.
*   **Memory Complexity:** Not given in the sources. Depends on the batch size, image resolution, and the number of wave trains $N$.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Self-supervised; does not require ground-truth wave parameters or geometry for training; leverages physical ocean models for plausibility [S1].
*   **Limitations:** Relies on the accuracy of the differentiable ocean model and renderer; sensitive to the quality of the initial geometry reconstruction; requires careful tuning of discriminator weights [S1].
*   **When to use:** When reconstructing dynamic ocean scenes from monocular video where ground-truth 3D data is unavailable [S1].
*   **When not to use:** When high-fidelity ground-truth geometry is available and supervised learning is preferred; when real-time performance is critical and the differentiable rendering overhead is too high [G].

## Background
*   **Origin:** The method was proposed in the paper "Dynamic ocean inverse modeling based on differentiable rendering" [S1].
*   **History:** It builds upon differentiable rendering techniques (SoftRas) and hierarchical ocean geometry models (displacement, normal, roughness) [S1]. It addresses the infeasibility of direct wave estimation from images by using a two-step inverse approach [S1].
*   **Usage:** Used for dynamic ocean reconstruction from monocular video sequences [S1].

## Sources
[S1] Dynamic ocean inverse modeling based on differentiable rendering, https://doi.org/10.1007/s41095-023-0338-4, fetched 2026-10-26
