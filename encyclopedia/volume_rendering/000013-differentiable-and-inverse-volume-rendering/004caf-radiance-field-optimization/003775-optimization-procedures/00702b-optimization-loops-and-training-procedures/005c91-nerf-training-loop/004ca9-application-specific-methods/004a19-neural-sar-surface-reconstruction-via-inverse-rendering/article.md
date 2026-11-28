# Neural SAR Surface Reconstruction via Inverse Rendering

Also called: Differentiable SAR Rendering, Neural DSM Reconstruction

## Overview
This method reconstructs a Digital Surface Model (DSM) and a backscattering coefficient map from Synthetic Aperture Radar (SAR) images by minimizing the difference between observed SAR intensities and synthesized intensities using a differentiable rendering model [S1]. It models elevation $Z$ and backscattering $B$ within a bounding box using a Multi-Layer Perceptron (MLP), reducing sampling degrees of freedom from a 3D volume to a 2D surface [S1]. Unlike standard NeRF which optimizes a 3D density field, this approach enforces surface constraints by mapping 2D ground coordinates directly to elevation and appearance, leading to faster and easier reconstruction of surfaces [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** $N$ SAR images with known acquisition parameters (azimuth, range, viewing geometry). Each image has dimensions $N_A \times N_R$ (azimuth $\times$ range) [S1].
*   **Outputs:** A reconstructed DSM (elevation map $Z(x,y)$) and a backscattering coefficient map $B(x,y)$ [S1].

**Data Structures**
*   **MLP:** A fully-connected network [S1].
    *   **Input:** 2D position $(x, y)$ on the ground, encoded using Instant NGP’s positional encoding $\gamma_P$ [S1].
    *   **Output:** Two values: Elevation $Z(x, y)$ and Backscattering coefficient $B(x, y)$ [S1].
*   **Bounding Box:** A 3D volume of side length $L$ defines the spatial extent for the reconstruction [S1].

**Rendering Model**
The method uses a simplified physically-based SAR intensity rendering model [S1]. For a sample $k$ at ground range $X_k$, the scattering coefficient $\sigma_k^0$ is computed based on the surface normal $\vec{n}_k$ and the local incidence unit vector $\vec{u}_k$ pointing from the sensor to the midpoint $P_{k+1/2}$ of the sample interval [S1]:

$$ \sigma_k^0(\vec{u}_k) = B(P_{k+1/2}) \cdot |\vec{u}_k \cdot \vec{n}_k| $$

where $B(P_{k+1/2})$ is the backscattering coefficient at the midpoint [S1]. The synthesized intensity $\hat{I}$ is derived from these scattering coefficients [S1].

**Optimization Steps**
1.  **Sampling:** For each minibatch, select a random subset of $M$ lines across all $N$ views. $M > 80$ to ensure adequate sample distribution while meeting memory constraints [S1].
2.  **Ground Range Shifting:** Shift the ground ranges sampled by a random value along their line of sight to avoid over-fitting at specific coordinates [S1].
3.  **Positional Encoding:** Encode the 2D position $(x, y)$ using Instant NGP’s positional encoding $\gamma_P$ [S1].
4.  **Frequency Windowing:** Apply Nerfies’ windowing on frequency bands on the output of $\gamma_P$ to regulate frequency bias during early training phases [S1]. Dynamically determine the target maximum frequency level $l$ for each sample $k$ of ground range $X_k$ such that $l$ corresponds to $|X_{k+1} - X_{k-1}| / 4$ [S1].
5.  **MLP Forward Pass:**
    *   Compute $B$ by applying an exponential activation to the MLP output to ensure positive values [S1].
    *   Compute $Z$ by applying the activation function $L_8 \cdot \arctan$ to the MLP output to bound its values [S1].
6.  **Loss Calculation:** Compute the total loss $L = L_D + \lambda L_{TV}$ [S1].
    *   $L_D$ is the negative log-likelihood loss:
        $$ L_D(\hat{I}) = \frac{1}{M \cdot N_R} \sum_{m=1}^{M} \sum_{n=1}^{N_R} \left( \frac{I_{m,n}}{\hat{I}_{m,n}} + \log\left(\frac{\hat{I}_{m,n}}{I_{m,n}}\right) \right) $$
        where $I_{m,n}$ is observed intensity and $\hat{I}_{m,n}$ is synthesized intensity [S1].
    *   $L_{TV}$ is total variation regularization on $B$ [S1].
7.  **Update:** Minimize $L$ using the Adam optimizer [S1].

**Coarse-to-Fine Strategy**
To efficiently reconstruct multiple scales and speed up initial training, gradually increase the number of samples $K$ by annealing a subsampling factor $\beta$ from $\beta_0$ to 1 [S1]. The parameters are updated as:
$$ K = K_f / \beta $$
$$ \mu = \mu_f \cdot \beta $$
where $K_f$ is the target number of samples per line and $\mu_f$ is the target accuracy [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class NeuralSARReconstruction:
    def __init__(self, L, K_f, mu_f, beta_0, lambda_tv=0.1):
        self.L = L
        self.K_f = K_f
        self.mu_f = mu_f
        self.beta = beta_0
        self.lambda_tv = lambda_tv
        
        # MLP: Input (x,y) -> Output [Z_raw, B_raw]
        self.mlp = nn.Sequential(
            nn.Linear(2, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 2)
        )
        self.optimizer = optim.Adam(self.mlp.parameters(), lr=1e-3)

    def positional_encoding(self, x, num_frequencies=10):
        # Standard multi-frequency sinusoidal encoding [G]
        scales = torch.tensor([2**i for i in range(num_frequencies)], dtype=torch.float32)
        x = x.unsqueeze(-1) * scales.unsqueeze(0)
        return torch.cat([torch.sin(x), torch.cos(x)], dim=-1)

    def nerfies_frequency_window(self, encoded, freq_limit):
        # Apply frequency band windowing to regulate bias [G]
        # Masks frequencies exceeding the dynamic limit
        mask = torch.ones_like(encoded)
        mask[:, freq_limit:] = 0.0
        return encoded * mask

    def shift_ranges(self, ranges, view_direction):
        # Shift ground ranges by a random value along line of sight
        offset = torch.rand_like(ranges) * 0.1 * torch.norm(view_direction)
        return ranges + offset

    def compute_normals(self, Z):
        # Compute surface normals from elevation map using finite differences
        dz_dx = torch.diff(Z, dim=1, prepend=Z[:, -1:])
        dz_dy = torch.diff(Z, dim=0, prepend=Z[-1:, :])
        normal = torch.stack([-dz_dx, -dz_dy, torch.ones_like(dz_dx)], dim=-1)
        return normal / torch.norm(normal, dim=-1, keepdim=True)

    def render_line(self, line_indices, view_params, images):
        M = len(line_indices)
        N_R = view_params['range_cells']
        synthesized = []
        observed = []
        all_B = []

        for m in line_indices:
            ranges = self.shift_ranges(view_params['ranges'], view_params['view_direction'])
            K = int(self.K_f / self.beta)
            mu = self.mu_f * self.beta

            # Generate sample positions along the ray projected to ground
            sample_positions = torch.stack([ranges[:K], torch.zeros(K)], dim=-1)

            pos_encoded = self.positional_encoding(sample_positions)
            # Nerfies windowing: dynamically determine max frequency level
            freq_limit = torch.abs(sample_positions[1:, 0] - sample_positions[:-1, 0]) / 4.0
            pos_encoded_windowed = self.nerfies_frequency_window(pos_encoded, freq_limit)

            mlp_output = self.mlp(pos_encoded_windowed)
            z_raw, b_raw = mlp_output[:, 0], mlp_output[:, 1]

            Z = 8.0 * torch.arctan(z_raw)
            B = torch.exp(b_raw)
            all_B.append(B)

            n_k = self.compute_normals(Z.unsqueeze(0))
            u_k = view_params['view_direction']

            dot_product = torch.sum(u_k * n_k, dim=-1)
            sigma_k0 = B * torch.abs(dot_product)

            # Render intensity: integrate scattering along ray
            delta_s = torch.ones(K) * mu
            transmittance = torch.ones(K + 1)
            for i in range(K):
                transmittance[i+1] = transmittance[i] * torch.exp(-sigma_k0[i] * delta_s[i])
            intensity = torch.sum(transmittance[:-1] * (1 - torch.exp(-sigma_k0 * delta_s)))
            synthesized.append(intensity)
            observed.append(images[m])

        return torch.stack(synthesized), torch.stack(observed), torch.cat(all_B)

    def total_variation(self, B):
        B_grad_x = B[:, 1:] - B[:, :-1]
        B_grad_y = B[1:, :] - B[:-1, :]
        return torch.mean(torch.abs(B_grad_x)) + torch.mean(torch.abs(B_grad_y))

    def train_step(self, images, view_params):
        self.optimizer.zero_grad()
        line_indices = torch.randperm(images.shape[0])[:80]
        synthesized, observed, B = self.render_line(line_indices, view_params, images)

        ld_loss = torch.mean(observed / synthesized + torch.log(synthesized / observed))
        ltv_loss = self.total_variation(B)
        loss = ld_loss + self.lambda_tv * ltv_loss

        loss.backward()
        self.optimizer.step()

        if self.beta < 1.0:
            self.beta += 0.01
        return loss.item()
```

**Parameters**
*   $L$: Side length of the bounding box. Not given in the sources.
*   $\lambda$: Weight for Total Variation regularization, set to 0.1 [S1].
*   $M$: Number of lines per minibatch, $M > 80$ [S1].
*   $K_f$: Target number of samples per line. Not given in the sources.
*   $\mu_f$: Target accuracy. Not given in the sources.
*   $\beta_0$: Initial subsampling factor. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Limited Views:** Reconstruction from only ascending/descending passes (2 views) shows higher RMSE due to a grainier heightmap from slight over-fitting to speckle, caused by reduced speckle realizations per spatial position [S1].
*   **Shadows vs. Submerged Areas:** Cliffs create large shadow areas similar to submerged areas, but the model does not confuse them: submerged areas are reconstructed with low backscattering coefficients, while cliff shadows are handled correctly [S1].
*   **Speckle Noise:** The negative log-likelihood loss accounts for the distribution of speckle, allowing training on noisy images [S1].

## Operating characteristics
*   **Complexity:** The method reduces sampling degrees of freedom from 3 (volume) to 2 (surface), leading to faster and easier reconstruction compared to volume rendering methods [S1].
*   **Performance:**
    *   Forez scene (5 views, 75m resolution, uniform material): RMSE 36.7m [S1].
    *   Forez scene (2 views, 75m resolution, uniform material): RMSE 52.9m [S1].
    *   Island scene (5 views, 0.75m resolution, multiple materials): RMSE 4.72m [S1].
*   **Strengths:** Successfully retrieves neural elevation models from only a few images and correctly separates geometry from texture [S1]. Handles multiple materials and complex scenes (e.g., cliffs, water) effectively [S1].
*   **Limitations:** Performance degrades with fewer views (e.g., 2 views) [S1]. Does not retrieve a precise map of materials, though demarcation can be observed on flatter terrain [S1].

## Background
This method was introduced in a paper by [S1] to demonstrate the ability of a simplified differentiable SAR intensity rendering model to retrieve DSMs from only a few images. It builds on the principles of inverse rendering and differentiable volume rendering, adapting them to the specific constraints and noise characteristics of SAR data [S1]. The method establishes results beyond self-inversion through experiments on EMPRISE® simulations, a step towards reconstructions from real-world SAR data [S1].

## Sources
[S1] Multi-view 3D surface reconstruction from SAR images by inverse rendering, https://doi.org/10.1109/lgrs.2025.3572303, fetched 2026-10-28
