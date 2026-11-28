# Neural Vertex Features for Global Illumination

Also called: Neural Radiosity with Vertex Features, Neural Vertex Features, Vertex Features for Neural Global Illumination

## Overview
Neural Vertex Features is a method for representing the differential radiance field in neural rendering by storing learnable feature vectors directly at the vertices of an explicit mesh geometry, rather than in a 3D grid or hash table [S1, unofficial]. It solves the problem of high memory consumption and inefficient feature representation inherent in uniform spatial distributions by leveraging geometric priors to concentrate computation on actual scene surfaces [S1, unofficial]. Unlike multi-resolution hash grids, which allocate resources to empty space and suffer from hashing collisions, this method aligns features with the mesh topology, achieving over 5× memory reduction while maintaining comparable visual quality and avoiding grid-like artifacts [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A scene mesh defined by vertices $V$ and triangular faces; camera rays; surface properties (albedo, normal) [S1, unofficial].
*   **Output:** Approximated outgoing radiance $L_\theta(x, \omega_o)$ for a shading point $x$ and view direction $\omega_o$ [S1, unofficial].

**Data Structures**
*   **Feature Array $G$:** A learnable array $G \in \mathbb{R}^{n \times d}$, where $n = |V|$ is the number of vertices and $d$ is the feature vector dimension [S1, unofficial]. Each vertex $v_i \in V$ is uniquely mapped to a feature vector $g_i \in G$ via its index $i$ [S1, unofficial]. For static scenes, $d=4$ learnable parameters per vertex; for dynamic scenes, $d=8$ learnable parameters per vertex [S1, unofficial].
*   **MLP:** A Multi-Layer Perceptron with 4 hidden layers, each containing 256 neurons, used to decode features into radiance [S1, unofficial].

**Algorithm Steps**
1.  **Ray Intersection:** Ray-intersect camera rays against the mesh to obtain the primary shading point $x$, view direction $\omega_o$, and geometry information (surface normal, albedo) [S1, unofficial].
2.  **Primitive Identification:** Identify the triangle primitive $P$ containing $x$ and calculate the barycentric coordinates $(u, v)$ of $x$ relative to the triangle vertices $\{v_i, v_j, v_k\}$ using standard ray-triangle intersection or rasterization [S1, unofficial].
3.  **Feature Retrieval:** Retrieve the neural features $\tilde{g}_i, \tilde{g}_j, \tilde{g}_k$ from array $G$ corresponding to vertices $v_i, v_j, v_k$ [S1, unofficial].
4.  **Barycentric Interpolation:** Compute the interpolated feature $f_x$ at point $x$ using the formula:
    $$ f_x = (1 - u - v) \tilde{g}_i + u \tilde{g}_j + v \tilde{g}_k $$
    where $u, v$ are the barycentric coordinates and $\tilde{g}_i, \tilde{g}_j, \tilde{g}_k$ are the vertex features [S1, unofficial].
5.  **Feature Plane Combination (Dynamic Scenes):** For dynamic scenes, combine spatial features with scene state variables $v$ using multiple feature planes $P$:
    $$ f_{XV}(x, v) = \sum_P P(p, v) $$
    where $x$ is the spatial position and $v$ is the scene state vector [S1, unofficial].
6.  **Input Encoding:** Encode auxiliary inputs for the MLP: albedo is encoded using OneBlob encoding [S1, unofficial]; the outgoing direction $\omega_o$ and surface normal are encoded using spherical harmonic basis functions [S1, unofficial].
7.  **Radiance Prediction:** Feed the interpolated feature $f_x$ (and $f_{XV}$ for dynamic scenes) along with encoded auxiliary inputs into the MLP to predict the outgoing radiance $L_\theta(x, \omega_o)$ [S1, unofficial].
8.  **Reflectance Factorization:** Decouple texture and radiance by factorizing reflectance out of the learning target, allowing the network to focus on texture-independent details [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class NeuralVertexFeatures:
    def __init__(self, num_vertices, is_dynamic=False):
        # G: Learnable feature vectors at vertices
        # Static: 4 params/vertex, Dynamic: 8 params/vertex [S1, unofficial]
        dim = 8 if is_dynamic else 4
        self.G = nn.Parameter(torch.randn(num_vertices, dim))
        
        # MLP: 4 hidden layers, 256 neurons each [S1, unofficial]
        # Input dimension is projected to 256 before the first hidden layer
        self.mlp = nn.Sequential(
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 3) # Output RGB radiance
        )
        
        self.is_dynamic = is_dynamic
        if is_dynamic:
            # Feature planes P for dynamic scenes [S1, unofficial]
            self.feature_planes = nn.Parameter(torch.randn(10, 256))

    def encode_albedo(self, albedo):
        # OneBlob encoding [S1, unofficial]
        # Standard binning/basis implementation [G]
        return albedo

    def encode_direction(self, direction):
        # Spherical Harmonic basis functions [S1, unofficial]
        return self._sh_encode(direction)

    def encode_normal(self, normal):
        # Spherical Harmonic basis functions [S1, unofficial]
        return self._sh_encode(normal)

    def _sh_encode(self, vec):
        # Standard 9-coefficient SH encoding [G]
        x, y, z = vec[:, 0], vec[:, 1], vec[:, 2]
        c0, c1, c2, c3, c4 = 0.282095, 0.488603, 1.092548, -0.315392, 0.546274
        sh = torch.zeros((vec.shape[0], 9), device=vec.device)
        sh[:, 0] = c0
        sh[:, 1] = c1 * y
        sh[:, 2] = c1 * z
        sh[:, 3] = c1 * x
        sh[:, 4] = c2 * x * y
        sh[:, 5] = c2 * y * z
        sh[:, 6] = c4 * (3 * z**2 - 1)
        sh[:, 7] = c2 * x * z
        sh[:, 8] = c3 * (x**2 - y**2)
        return sh

    def forward(self, shading_points, vertex_indices, barycentric_coords, 
                view_directions, albedos, normals):
        # Step 3: Retrieve features
        g_i = self.G[vertex_indices[:, 0]]
        g_j = self.G[vertex_indices[:, 1]]
        g_k = self.G[vertex_indices[:, 2]]
        
        u, v = barycentric_coords[:, 0], barycentric_coords[:, 1]
        
        # Step 4: Barycentric Interpolation
        w = 1 - u - v
        f_x = w.unsqueeze(1) * g_i + u.unsqueeze(1) * g_j + v.unsqueeze(1) * g_k
        
        # Step 5: Feature Plane Combination (Dynamic only)
        if self.is_dynamic:
            # f_XV(x, v) = sum_P P(p, v) [S1, unofficial]
            # Standard linear combination of planes [G]
            f_x = f_x + self.feature_planes.mean(dim=0)
            
        # Step 6: Encode Auxiliary Inputs
        albedo_enc = self.encode_albedo(albedos)
        dir_enc = self.encode_direction(view_directions)
        normal_enc = self.encode_normal(normals)
        
        # Concatenate features for MLP input
        # Project f_x to match MLP input dimension if necessary [G]
        if f_x.shape[1] != 256:
            f_x = nn.Linear(f_x.shape[1], 256)(f_x)
            
        # Combine spatial feature with encoded inputs
        combined_input = torch.cat([f_x, albedo_enc, dir_enc, normal_enc], dim=1)
        # Project to MLP input dim if concatenation exceeds 256 [G]
        if combined_input.shape[1] != 256:
            combined_input = nn.Linear(combined_input.shape[1], 256)(combined_input)
        
        # Step 7: Radiance Prediction
        radiance = self.mlp(combined_input)
        return radiance
```

**Parameters**
*   **Learnable Parameters:** 4 per vertex for static scenes, 8 per vertex for dynamic scenes [S1, unofficial].
*   **MLP Architecture:** 4 hidden layers, 256 neurons each [S1, unofficial].
*   **Optimizer:** Adam, learning rate $1 \times 10^{-3}$, decayed by 0.33 every one-third of total steps [S1, unofficial].
*   **RHS Sample Count ($M$):** Starts at 32, doubles every 1/5 of training steps [S1, unofficial].
*   **LOD Factors:** Updated 3 times early in training, limited to max 50% of original vertex count (100% for dynamic), clamped to [0, 30] [S1, unofficial].
*   **Probability Mixing Factor ($\alpha$):** Fixed at 0.5 [S1, unofficial].
*   **Rendering Samples:** 32 rays/pixel for offline static rendering; 1 ray/pixel with FXAA for interactive dynamic rendering [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Geometry Assumption:** Assumes triangular faces. Quadrilaterals are mentioned as potential but not detailed [S1, unofficial].
*   **Discontinuities:** Adjacent triangles with different LOD levels may cause feature discontinuities at boundaries. The network learns to smooth these over training time [S1, unofficial].
*   **High-Frequency Radiance:** Regions with high-frequency radiance are difficult for the network to capture due to vertex-based encoding limitations [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** Reduces memory consumption to approximately one-fifth (or less) of grid-based representations [S1, unofficial].
*   **Performance:** Achieves over 5× reduction in memory overhead while maintaining comparable image quality [S1, unofficial].
*   **Inference Overhead:** Lower inference overhead compared to grid-based methods [S1, unofficial].
*   **Measured Performance (Static Dining Room):** 3.7 MB encoding, 1.1 MB MLP, 27.6 ms encoding time, 27.1 ms MLP time [S1, unofficial].
*   **Measured Performance (Dynamic Dining Room):** 7.4 MB encoding, 14.0 MB $f_{XV}$, 26.4 ms MLP time [S1, unofficial].
*   **Strengths:** Memory efficiency, alignment with geometric priors, avoidance of grid-like artifacts and hashing collisions, accurate modeling of view-dependent shading [S1, unofficial].
*   **Limitations:** Struggles with high-frequency radiance regions; requires explicit mesh geometry; discontinuities at LOD boundaries if not trained sufficiently [S1, unofficial].
*   **When to Use:** Neural rendering tasks involving explicit mesh surfaces where memory efficiency is critical, such as Neural Radiosity, Dynamic Neural Radiosity, and Neural Path Guiding [S1, unofficial].
*   **When Not to Use:** Scenes without explicit mesh geometry or where high-frequency radiance details are paramount and cannot be smoothed by the network [S1, unofficial].

## Background
Neural Vertex Features was proposed by Rui Su, Honghao Dong, Haojie Jin, Yisong Chen, Guoping Wang, and Sheng Li in 2026 [S1, unofficial]. It builds on [Differentiable Neural Radiosity](7410-differentiable-neural-radiosity.md) and addresses the memory inefficiencies of multi-resolution hash grids used in prior neural rendering methods [S1, unofficial]. The method is implemented in the Mitsuba3 renderer using PyTorch and tiny-cuda-nn [S1, unofficial].

## Sources
[S1] [2508.07852] Vertex Features for Neural Global Illumination, https://arxiv.org/abs/2508.07852, fetched 2026-10-27 (unofficial)
