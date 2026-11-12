# Neural SAR Surface Reconstruction via Inverse Rendering

Also called: Differentiable SAR Rendering, Neural DSM Reconstruction

## Overview
This method reconstructs a Digital Surface Model (DSM) and a backscattering coefficient map from Synthetic Aperture Radar (SAR) images by minimizing the difference between observed SAR intensities and synthesized intensities using a differentiable rendering model [S1]. It models elevation $Z$ and backscattering $B$ within a bounding box using a Multi-Layer Perceptron (MLP), reducing sampling degrees of freedom from 3D volume to 2D surface [S1]. Unlike volume-based approaches, this surface representation is more constrained, leading to faster and easier reconstruction of surfaces [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ SAR images with known acquisition parameters (azimuth, range, viewing geometry). Each image has dimensions $N_A \times N_R$ (azimuth $\times$ range).
*   **Outputs:** A continuous elevation field $Z(x, y)$ and a backscattering coefficient field $B(x, y)$ defined over a 2D ground plane.

**Data Structures:**
*   **MLP:** A fully-connected network.
    *   **Input:** 2D position $(x, y)$ on the ground.
    *   **Encoding:** Instant NGP’s positional encoding $\gamma_P$ [S1].
    *   **Output:** Two values: Elevation $Z(x, y)$ and Backscattering coefficient $B(x, y)$.
*   **Bounding Box:** A 3D volume of side length $L$ used to constrain the search space for $Z$ and $B$ [S1].

**Numbered Steps:**

1.  **Sampling Strategy:**
    *   For each minibatch, select a random subset of $M$ lines across all $N$ views.
    *   Constraint: $M > 80$ to ensure adequate sample distribution while meeting memory constraints [S1].
    *   For each selected line, sample $K$ points along the ground range.
    *   **Shift:** Shift the sampled ground ranges by a random value along their line of sight to avoid over-fitting at specific coordinates [S1].

2.  **Coarse-to-Fine Annealing:**
    *   Define target samples per line $K_f$ and target accuracy $\mu_f$.
    *   Initialize subsampling factor $\beta = \beta_0$.
    *   Anneal $\beta$ from $\beta_0$ to $1$ during training.
    *   Calculate current parameters: $K = K_f / \beta$ and $\mu = \mu_f \cdot \beta$ [S1].

3.  **Frequency Windowing (Nerfies):**
    *   Regulate frequency bias during early training phases [S1].
    *   For each sample $k$ of ground range $X_k$, determine the target maximum frequency level $l \in [0, L]$ such that $l$ corresponds to $|X_{k+1} - X_{k-1}| / 4$ [S1].
    *   Apply windowing on the output of positional encoding $\gamma_P$ based on $l$ [S1].

4.  **Neural Field Evaluation:**
    *   Encode the 2D position $(x, y)$ using $\gamma_P$.
    *   Pass through MLP to get raw outputs.
    *   **Backscattering:** Apply exponential activation to ensure positive values: $B = \exp(\text{MLP}_B)$ [S1].
    *   **Elevation:** Apply activation $L/8 \cdot \arctan(\text{MLP}_Z)$ to bound elevation values within reasonable limits [S1].

5.  **Differentiable Rendering:**
    *   For each sample point $P_k$ on the surface defined by $Z(x,y)$:
        *   Compute surface normal $\vec{n}_k$ from the local quadrilateral patch [S1].
        *   Compute local incidence unit vector $\vec{u}_k$ pointing from the sensor to $P_k$ [S1].
        *   Compute scattering coefficient $\sigma_k^0$ at the midpoint $P_{k+1/2}$:
            $$ \sigma_k^0(\vec{u}_k) = B(P_{k+1/2}) \cdot |\vec{u}_k \cdot \vec{n}_k| $$
            where $B$ is the backscattering coefficient and $|\vec{u}_k \cdot \vec{n}_k|$ is the cosine attenuation fitting the targeted incidence angle [S1].
    *   Render synthesized intensity $\hat{I}_{m,n}$ for each range cell $n$ in line $m$ by integrating contributions from samples along the ray [S1].

6.  **Loss Calculation:**
    *   Compute Negative Log-Likelihood Loss ($L_D$) to account for speckle distribution:
        $$ L_D(\hat{I}) = \frac{1}{M \cdot N_R} \sum_{m=1}^{M} \sum_{n=1}^{N_R} \left( \frac{I_{m,n}}{\hat{I}_{m,n}} + \log \left( \frac{\hat{I}_{m,n}}{I_{m,n}} \right) \right) $$
        where $I_{m,n}$ is observed intensity and $\hat{I}_{m,n}$ is synthesized speckle-free intensity [S1].
    *   Compute Total Variation Regularization ($L_{TV}$) on $B$ to penalize over-fitting and favor geometry fitting [S1].
    *   Total Loss: $L = L_D + \lambda L_{TV}$ with $\lambda = 0.1$ [S1].

7.  **Optimization:**
    *   Minimize $L$ using the Adam optimizer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class NeuralSARReconstruction:
    def __init__(self, L, K_f, mu_f, beta_0, lambda_tv=0.1, M_lines=80):
        """
        Initialize the reconstruction model.
        
        Args:
            L: Side length of the bounding box.
            K_f: Target number of samples per line.
            mu_f: Target accuracy.
            beta_0: Initial subsampling factor.
            lambda_tv: Weight for Total Variation regularization.
            M_lines: Number of lines to render per minibatch.
        """
        self.L = L
        self.K_f = K_f
        self.mu_f = mu_f
        self.beta = beta_0
        self.lambda_tv = lambda_tv
        self.M_lines = M_lines
        
        # MLP for Elevation (Z) and Backscattering (B)
        # Input: 2D position (x, y)
        # Output: 2 values (Z, B_raw)
        self.mlp = nn.Sequential(
            nn.Linear(2, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 2) # [Z_raw, B_raw]
        )
        
        # Instant NGP Positional Encoding (Simplified placeholder for implementation)
        # In practice, this uses hash grids or similar structures from Instant NGP.
        self.pos_enc = InstantNGPEncoding() 
        
        self.optimizer = optim.Adam(self.mlp.parameters(), lr=1e-3)

    def get_activation(self, z_raw):
        """Apply L/8 * arctan to bound elevation."""
        return (self.L / 8.0) * torch.arctan(z_raw)

    def get_backscattering(self, b_raw):
        """Apply exponential to ensure positive backscattering."""
        return torch.exp(b_raw)

    def apply_frequency_windowing(self, encoded_pos, X_k, X_k_minus_1, X_k_plus_1):
        """
        Apply Nerfies' windowing on frequency bands.
        
        Args:
            encoded_pos: Output of positional encoding.
            X_k: Current ground range sample.
            X_k_minus_1: Previous ground range sample.
            X_k_plus_1: Next ground range sample.
            
        Returns:
            Windowed encoded position.
        """
        # Determine target maximum frequency level l
        # l corresponds to |X_{k+1} - X_{k-1}| / 4
        # Note: Exact implementation of Nerfies windowing depends on the specific 
        # hash grid or sine basis implementation. Here we assume a generic 
        # frequency masking function.
        freq_limit = torch.abs(X_k_plus_1 - X_k_minus_1) / 4.0
        
        # Apply windowing to high frequencies if they exceed the limit
        # This is a conceptual representation of the windowing mechanism.
        # In Instant NGP, this involves masking specific hash entries or 
        # scaling frequency components.
        windowed_pos = apply_nerfies_window(encoded_pos, freq_limit)
        return windowed_pos

    def compute_scattering_coefficient(self, B_mid, u_k, n_k):
        """
        Compute scattering coefficient sigma_k^0.
        
        Args:
            B_mid: Backscattering coefficient at midpoint P_{k+1/2}.
            u_k: View direction vector (unit).
            n_k: Surface normal vector (unit).
            
        Returns:
            Scattering coefficient.
        """
        # sigma_k^0 = B(P_{k+1/2}) * |u_k . n_k|
        cos_angle = torch.abs(torch.dot(u_k, n_k))
        return B_mid * cos_angle

    def render_line(self, view_params, line_indices, ground_range_samples):
        """
        Render a single line (row) for a specific view.
        
        Args:
            view_params: Camera/view parameters for the current view.
            line_indices: Indices of the lines to render.
            ground_range_samples: 1D tensor of ground range positions X_k.
            
        Returns:
            Synthesized intensities for the line.
        """
        # 1. Sample points along the line
        # For each X_k, compute (x, y) based on view geometry
        # and retrieve Z(x, y) from MLP
        
        # 2. Compute normals and view directions
        # 3. Compute scattering coefficients
        # 4. Integrate along ray to get intensity
        
        # Placeholder for complex ray marching integration
        # The actual integration depends on the specific differentiable 
        # volume/surface rendering engine used.
        pass

    def compute_loss(self, I_obs, I_syn, B_pred):
        """
        Compute total loss L = L_D + lambda * L_TV.
        
        Args:
            I_obs: Observed SAR intensities.
            I_syn: Synthesized intensities.
            B_pred: Predicted backscattering coefficients.
            
        Returns:
            Total loss scalar.
        """
        # Negative Log-Likelihood Loss
        # L_D = (1 / (M * N_R)) * sum( I_obs/I_syn + log(I_syn/I_obs) )
        # Add small epsilon to avoid division by zero
        epsilon = 1e-8
        ld = torch.mean(I_obs / (I_syn + epsilon) + torch.log((I_syn + epsilon) / (I_obs + epsilon)))
        
        # Total Variation Regularization on B
        # L_TV = sum(|B(x+1) - B(x)| + |B(y+1) - B(y)|)
        # Simplified 1D TV for demonstration
        b_shifted_right = torch.roll(B_pred, shifts=-1, dims=-1)
        b_shifted_left = torch.roll(B_pred, shifts=1, dims=-1)
        l_tv = torch.mean(torch.abs(B_pred - b_shifted_right)) + torch.mean(torch.abs(B_pred - b_shifted_left))
        
        total_loss = ld + self.lambda_tv * l_tv
        return total_loss

    def step(self, batch_data):
        """
        Perform one optimization step.
        
        Args:
            batch_data: Dictionary containing SAR images, view parameters, etc.
        """
        self.optimizer.zero_grad()
        
        # 1. Select random subset of M lines
        M = self.M_lines
        N_views = len(batch_data['views'])
        selected_lines = select_random_lines(batch_data['lines'], M)
        
        # 2. Anneal beta
        K = int(self.K_f / self.beta)
        mu = self.mu_f * self.beta
        
        # 3. Render and Compute Loss
        I_syn = []
        I_obs = []
        B_pred_list = []
        
        for view_idx, line_idx in selected_lines:
            view_params = batch_data['views'][view_idx]
            line_data = batch_data['lines'][line_idx]
            
            # Shift ground ranges randomly
            X_k = line_data['ground_range'] + torch.rand_like(line_data['ground_range']) * mu
            
            # Render line
            # Note: This requires implementing the differentiable renderer
            # which calls the MLP for Z and B, computes normals, and integrates.
            I_syn_line = self.render_line(view_params, line_idx, X_k)
            
            I_syn.append(I_syn_line)
            I_obs.append(line_data['observed_intensity'])
            
            # Retrieve B for regularization (requires storing B from render)
            # This is a simplification; in practice, B is computed during rendering
            pass
            
        I_syn = torch.cat(I_syn)
        I_obs = torch.cat(I_obs)
        
        # Compute Loss
        # Note: B_pred needs to be extracted from the MLP outputs during rendering
        # For this pseudocode, we assume B_pred is available or computed separately
        # to match the shape of I_syn for regularization.
        # In a real implementation, B_pred would be returned by render_line.
        
        # Placeholder for B_pred extraction
        # B_pred = extract_backscattering_from_rendered_data(...)
        
        loss = self.compute_loss(I_obs, I_syn, B_pred)
        loss.backward()
        self.optimizer.step()
        
        # 4. Anneal beta (decrease beta from beta_0 to 1)
        self.beta = max(1.0, self.beta - annealing_rate)
        
        return loss.item()

    def train(self, data_loader, epochs):
        for epoch in range(epochs):
            for batch in data_loader:
                loss = self.step(batch)
                print(f"Epoch {epoch}, Loss: {loss}")
```

**Edge Cases and Pitfalls:**
*   **Limited Views:** Reconstruction from only ascending/descending passes (2 views) shows higher RMSE due to a grainier heightmap caused by slight over-fitting to speckle. This is due to reduced speckle realizations per spatial position [S1].
*   **Cliffs vs. Submerged Areas:** Cliffs create large shadow areas similar to submerged areas. The model does not confuse them: submerged areas are reconstructed with low backscattering coefficients, while cliff shadows are handled correctly by the geometry [S1].
*   **Over-fitting:** Without Total Variation regularization, the model may over-fit to the backscattering coefficient $B$, leading to poor geometry fitting [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Reduces sampling degrees of freedom from 3 (volume) to 2 (surface), leading to faster and easier reconstruction compared to volume rendering methods [S1]. Memory constraints are managed by rendering a random subset of $M > 80$ lines per minibatch [S1].
*   **Measured Performance:**
    *   Forez scene (5 views, 75m resolution, uniform material): RMSE 36.7m [S1].
    *   Forez scene (2 views, 75m resolution, uniform material): RMSE 52.9m [S1].
    *   Island scene (5 views, 0.75m resolution, multiple materials): RMSE 4.72m [S1].
*   **Strengths:**
    *   Separates geometry and appearance effectively [S1].
    *   Works with a limited number of views [S1].
    *   Simple backscattering representation is sufficient for limited views [S1].
*   **Limitations:**
    *   Performance degrades with fewer views (e.g., 2 views) [S1].
    *   Does not retrieve a precise map of materials, though demarcation can be observed on flatter terrain [S1].
*   **When to Use:** Suitable for reconstructing DSMs from a few SAR images where computational efficiency is important and the scene has moderate complexity.
*   **When Not to Use:** Not suitable for scenes requiring high-fidelity material reconstruction or when many views are available and volume-based methods might offer more detail (though surface is faster).

## Background
*   **Origin:** Introduced in a paper by [S1] titled "Multi-view 3D surface reconstruction from SAR images by inverse rendering" (2025).
*   **History:** Builds upon differentiable rendering and NeRF concepts, specifically adapting them for SAR data. It references Instant NGP [S1] for positional encoding and Nerfies [S1] for frequency windowing.
*   **Usage:** Demonstrated on synthetic SAR data generated by the EMPRISE® engine [S1].

## Sources
[S1] Multi-view 3D surface reconstruction from SAR images by inverse rendering, https://doi.org/10.1109/lgrs.2025.3572303, fetched 2026-10-28
