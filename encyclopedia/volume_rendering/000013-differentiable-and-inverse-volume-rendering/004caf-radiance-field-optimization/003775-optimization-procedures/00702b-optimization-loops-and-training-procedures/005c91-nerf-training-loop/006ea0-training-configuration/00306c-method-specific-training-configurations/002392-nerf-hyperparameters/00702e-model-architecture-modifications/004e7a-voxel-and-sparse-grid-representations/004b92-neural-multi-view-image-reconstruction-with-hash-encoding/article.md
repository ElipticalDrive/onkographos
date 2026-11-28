# Neural Multi-View Image Reconstruction with Hash Encoding

Also called: Ours (multi-view)

## Overview
This method performs neural multi-view image reconstruction and novel view synthesis by combining a hash-encoded geometry network with a separate light field color network [S1]. It solves the problem of reconstructing high-frequency geometric details and accurate textures from multi-view images, addressing the noise and lack of geometric detail found in standard NeRF and the poor performance in textureless areas seen in methods like NeuS and IDR [S1]. It is distinguished from its siblings by its specific use of a 16-layer hash encoding table for geometry and a thresholding strategy based on the mean Signed Distance Function (SDF) to filter invalid samples, enabling faster training and lower memory usage compared to dense voxel-based or pure neural implicit methods [S1].

## How it works
**Inputs:** A set of multi-view images with known camera parameters.
**Outputs:** A reconstructed 3D geometry (extracted as a mesh) and synthesized novel view images.

**Data Structures:**
1.  **Hash Encoding Table:** A hierarchical structure with 16 layers [S1]. Each layer contains a hash table mapping spatial coordinates to feature vectors. The feature dimension for each grid feature in each layer is 2 [S1].
2.  **Geometry Network:** A Multi-Layer Perceptron (MLP) with 6 hidden layers [S1].
3.  **Light Field Network:** A Multi-Layer Perceptron (MLP) with 4 hidden layers [S1]. Each hidden layer contains 256 units [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize the geometry network using a modified strategy from [12] (as referenced in [S1]). Set hyperparameters $\alpha=0.1$, $\beta=0.1$, and $\gamma=5 \times 10^{-4}$ for the loss function weights [S1].

2.  **Ray Sampling:** For each training iteration, sample rays from the input views. Use a hierarchical sampling strategy to sample points along these rays [S1]. The number of rays sampled per iteration is 512 [S1].

3.  **Geometry Feature Extraction:**
    For each sampled point $\mathbf{x}$:
    a. Compute the Fourier positional encoding of $\mathbf{x}$ with dimension 6 [S1].
    b. Query the hash encoding table for $\mathbf{x}$ to obtain the geometry feature vector $\mathbf{f}_{geo}$ [S1].
    c. Pass the Fourier encoding and $\mathbf{f}_{geo}$ through the Geometry Network to predict the Signed Distance Function (SDF) value $s$ and a geometry feature vector $\mathbf{f}_{out}$ [S1].

4.  **Invalid Point Filtering:**
    Calculate the mean SDF value $\bar{s}$ of all sampled points on the current ray [S1]. Eliminate any sampled points where the SDF value is significantly greater than $\bar{s}$ (specifically, points that are invalid relative to the surface threshold) [S1]. This step refines the set of points used for rendering [S1].

5.  **Color Prediction:**
    For each valid sampled point $\mathbf{x}$:
    a. Compute the gradient $\mathbf{n} = \nabla s(\mathbf{x})$ of the SDF at $\mathbf{x}$ [S1].
    b. Compute the Fourier positional encoding of the view direction $\mathbf{v}$ and the normal $\mathbf{n}$, both with dimension 6 [S1].
    c. Concatenate the Fourier encodings of $\mathbf{v}$, $\mathbf{n}$, and the geometry feature vector $\mathbf{f}_{out}$ from Step 3c [S1].
    d. Pass this concatenated vector through the Light Field Network to predict the view-dependent color $\mathbf{c}$ [S1].

6.  **Volume Rendering:**
    Render the image using the filtered points, their predicted colors $\mathbf{c}$, and SDF values $s$ (converted to density $\sigma$) according to standard volume rendering equations [S1]. The specific conversion from SDF to density is not detailed in the provided notes, but standard differentiable rendering is implied [S1].

7.  **Loss Calculation:**
    Compute the loss using the weighted sum of terms with $\alpha=0.1$, $\beta=0.1$, and $\gamma=5 \times 10^{-4}$ [S1]. The specific loss terms are not explicitly named in the notes, but they typically include photometric loss and SDF regularization (e.g., Eikonal loss) common in SDF-based methods [S1].

8.  **Optimization:**
    Update the network parameters (hash table features and MLP weights) using gradient descent [S1].

9.  **Geometry Extraction (Post-Training):**
    Extract the final geometry using the marching cubes algorithm on the predicted SDF field at a resolution of $512^3$ [S1]. A higher resolution of $2048^3$ may be used for detailed visualization [S1].

10. **Novel View Synthesis:**
    Synthesize novel view images by performing volume or surface rendering of the reconstructed geometry using the given novel view camera parameters [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class HashEncoding:
    def __init__(self, num_layers=16, feature_dim=2):
        self.num_layers = num_layers
        self.feature_dim = feature_dim
        # Initialize hash tables. In practice, these are sparse tensors or 
        # lookup tables with hash functions. Here we assume a simplified dense 
        # representation for clarity, though the source implies a hash table [S1].
        self.grids = nn.ParameterList([
            nn.Parameter(torch.randn(2**16, feature_dim)) for _ in range(num_layers)
        ])
        # Note: Real implementation requires a hash function mapping 3D coords 
        # to table indices. This is omitted as it is a standard utility [G].

    def query(self, x, layer_idx):
        # x: [N, 3] coordinates
        # Returns: [N, feature_dim] features
        # Hash function h(x) -> index
        indices = hash_function(x) % self.grids[layer_idx].shape[0]
        return self.grids[layer_idx][indices]

class GeometryNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # 6 MLP layers as per source [S1]
        self.layers = nn.ModuleList([
            nn.Linear(12, 256), # Input: 6 (Fourier x) + 6 (Hash features? Source says connected layer output)
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 1)   # Output SDF
        ])
        # Note: Source [S1] says input to first layer is Fourier x, 
        # input to third layer concatenates hash encoding and output of second layer.
        # This structure is complex. Simplified for implementation:
        self.mlp = nn.Sequential(
            nn.Linear(6, 256), # Fourier x
            nn.ReLU(),
            # Layer 2 takes Fourier x + Hash features? Source is ambiguous on exact 
            # connectivity for all 6 layers. We assume standard MLP flow for SDF [G].
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 1)
        )

    def forward(self, x_fourier, geo_features):
        # Concatenate and pass through MLP
        # Source [S1] implies specific connectivity. 
        # Simplified: x_fourier and geo_features are combined.
        x_combined = torch.cat([x_fourier, geo_features], dim=-1)
        sdf = self.mlp(x_combined)
        return sdf

class LightFieldNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # 4 hidden layers, 256 units each [S1]
        self.layers = nn.Sequential(
            nn.Linear(6 + 6 + 256, 256), # Fourier v, Fourier n, geo feature
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 3) # RGB color
        )

    def forward(self, v_fourier, n_fourier, geo_feature):
        x = torch.cat([v_fourier, n_fourier, geo_feature], dim=-1)
        color = self.layers(x)
        return color

def train_step(images, cameras, hash_encoder, geo_net, light_net, optimizer):
    # 1. Sample rays (512 rays per iteration [S1])
    rays = sample_rays(cameras, num_rays=512)
    
    # 2. Hierarchical sampling of points on rays [S1]
    points = hierarchical_sample(rays, num_samples=64) # Typical NeRF sampling
    
    # 3. Get Fourier encodings
    x_fourier = fourier_encoding(points, dim=6)
    v_fourier = fourier_encoding(rays.directions, dim=6)
    
    # 4. Query Hash Encoding for Geometry
    geo_features = []
    for layer in range(hash_encoder.num_layers):
        # Aggregate features from all layers? Source says "hash encoding ... with 16 layers"
        # and "dimension of each grid feature ... is 2". 
        # Standard hash encoding (like Instant-NGP) sums or concatenates. 
        # Source [S1] does not specify aggregation method explicitly, 
        # but "feature vector output from the connected layer" implies a single vector.
        # We assume summation or concatenation of all 16 layers' features.
        feat = hash_encoder.query(points, layer)
        geo_features.append(feat)
    geo_features = torch.stack(geo_features, dim=1).sum(dim=1) # Sum aggregation [G]
    
    # 5. Predict SDF
    sdf = geo_net(x_fourier, geo_features)
    
    # 6. Compute Gradient of SDF for Normal
    sdf_grad = compute_gradient(sdf, points)
    n = sdf_grad / (torch.norm(sdf_grad, dim=-1, keepdim=True) + 1e-5)
    n_fourier = fourier_encoding(n, dim=6)
    
    # 7. Filter Invalid Points
    mean_sdf = torch.mean(sdf, dim=0) # Mean of sampled points on ray
    # Eliminate points where SDF > mean_sdf (or similar thresholding logic [S1])
    valid_mask = sdf.squeeze() <= mean_sdf + 0.1 # Heuristic threshold [G]
    
    # 8. Predict Color for Valid Points
    # Get geometry feature for valid points (from the last layer of geo net or specific output [S1])
    # Source says "geometry feature vector output from the connected layer". 
    # We assume this is the output of the geometry network before SDF or a specific hidden layer.
    # For simplicity, we use the geo_features derived from hash encoding.
    valid_geo_features = geo_features[valid_mask]
    valid_v_fourier = v_fourier[valid_mask]
    valid_n_fourier = n_fourier[valid_mask]
    
    colors = light_net(valid_v_fourier, valid_n_fourier, valid_geo_features)
    
    # 9. Render and Compute Loss
    # Standard volume rendering with filtered points
    rendered_image = volume_render(points[valid_mask], colors, sdf[valid_mask])
    target_image = images[current_batch]
    
    loss = torch.nn.functional.mse_loss(rendered_image, target_image)
    
    # Add regularization terms with weights alpha, beta, gamma [S1]
    # Specific terms not detailed, assuming standard SDF regularization [G]
    eikonal_loss = compute_eikonal_loss(sdf_grad)
    loss += 0.1 * eikonal_loss # alpha=0.1
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    return loss

def extract_geometry(sdf_field, resolution=512):
    # Marching cubes on SDF field [S1]
    vertices, faces = marching_cubes(sdf_field, level=0.0, isovalue=0.0)
    return vertices, faces
```

**Edge Cases and Pitfalls:**
*   **Precision:** The hash encoding relies on integer hashing. Collisions are inevitable; the source does not specify the hash function or collision resolution strategy [S1].
*   **Boundaries:** The bounding volume $\Omega$ must be defined. The source mentions uniformly sampling points in $\Omega$ for point cloud tasks, but for multi-view, the volume is likely defined by the camera frustums [S1].
*   **Degenerate Input:** Textureless areas are a known challenge for baselines (NeuS, IDR), but this method handles them better [S1]. However, if input images have no overlap or poor calibration, the SDF estimation will fail.

## Operating characteristics
*   **Time Complexity:** Training time is 1.8 hours for the tested scenes on a GeForce RTX 3090 GPU [S1]. This is significantly faster than IDR (5.2 hours), NeuS (7.2 hours), UNISURF (21 hours), VolSDF (9 hours), and NeRF (9.1 hours) [S1].
*   **Memory Complexity:** Memory usage is 5 GB [S1]. This is lower than IDR (6.5 GB), NeuS (7 GB), UNISURF (6 GB), VolSDF (13.8 GB), and NeRF (8 GB) [S1].
*   **Performance:** Achieves a mean Chamfer-L2 distance of $0.72 \times 10^{-6}$ and a mean PSNR of 30.85 dB on the DTU dataset [S1]. This outperforms IDR (CD: $0.90 \times 10^{-6}$, PSNR: 23.20 dB) and NeuS (CD: $0.77 \times 10^{-6}$, PSNR: 29.66 dB) [S1].
*   **Scaling:** The method uses 512 rays per iteration, compared to 2048 for IDR and 512 for NeuS [S1]. The number of iterations is $120 \times 10^3$ [S1].
*   **Strengths:** Reconstructs more geometric detail than baselines, especially in textureless areas and high-frequency details (e.g., bird feathers) [S1]. Faster training and lower memory usage [S1].
*   **Limitations:** Requires multi-view images with known camera parameters. The quality of novel view synthesis depends on the accuracy of the reconstructed geometry and color network [S1].

## Background
The method was proposed by Z. Deng, H. Xiao, Y. Lang, et al. in the paper "Multi-scale hash encoding based neural geometry representation" [S1]. It was developed as an improvement over existing neural implicit representations like NeuS, IDR, and NeRF, which struggle with geometric detail and computational efficiency. The work builds on the concept of hash encoding for efficient spatial indexing, adapting it specifically for multi-view image reconstruction and SDF-based geometry extraction [S1].

## Sources
[S1] Multi-scale hash encoding based neural geometry representation, https://doi.org/10.1007/s41095-023-0340-x, fetched 2026-10-26
