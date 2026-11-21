# SparseNeus

Also called: SparseNeuS

## Overview
SparseNeuS is a neural rendering-based method for surface reconstruction from multi-view images that generalizes to new scenes and works with sparse inputs (as few as 2 or 3 images) [S2]. It improves upon [NeuS](6060-neus.md) by introducing a geometry encoding volume with learnable image features as a hybrid representation, allowing it to recover surfaces in a coarse-to-fine manner without requiring dense views or time-consuming per-scene optimization from scratch [S1, S2]. Unlike [NeuS](6060-neus.md), which suffers from distorted and incomplete geometry with only three images due to radiance ambiguity, SparseNeuS produces high-quality, complete meshes by leveraging generalizable priors learned from image features [S2].

## How it works
The pipeline consists of three parts: Geometry reasoning, Appearance prediction, and Per-scene fine-tuning [S2].

### Inputs and Outputs
**Inputs:**
*   $N$ input images $\{I_i\}_{i=0}^{N-1}$ (typically $N=3$) [S2].
*   Camera parameters for each input image [S2].
*   A bounding box covering the region of interest, defined in the camera coordinate system of the centered input image and gridded into regular voxels [S2].

**Outputs:**
*   A Signed Distance Function (SDF) field representing the surface geometry [S2].
*   Synthesized views of the scene [S2].

### Data Structures
1.  **Geometry Encoding Volume ($M$):** A 3D sparse convolutional volume implemented with a U-Net-like architecture using `torchsparse` [S2].
    *   **Coarse Volume:** Resolution $96 \times 96 \times 96$ [S2].
    *   **Fine Volume:** Resolution $192 \times 192 \times 192$ [S2].
2.  **Cost Volume ($B$):** A 3D volume where each voxel $v$ stores the variance of projected features [S2].
3.  **Feature Maps ($F_i$):** 2D feature maps extracted from input images $I_i$ using a Feature Pyramid Network (FPN) [S2].
4.  **SDF Prediction Network ($f_\theta$):** An MLP that takes 3D coordinates and interpolated features from $M$ as input to predict SDF values [S2].

### Steps

#### 1. Geometry Reasoning
This stage constructs cascaded geometry encoding volumes to recover surfaces in a coarse-to-fine manner [S2].

1.  **Feature Extraction:** Extract 2D feature maps $\{F_i\}_{i=0}^{N-1}$ from the input images $\{I_i\}_{i=0}^{N-1}$ using a 2D feature extraction network (FPN) [S2].
2.  **Projection:** For each vertex $v$ of the bounding box, project it to each feature map $F_i$ using the camera parameters of image $I_i$. Let $\pi_i(v)$ denote the projected pixel location. Obtain the features $F_i(v) = F_i(\pi_i(v))$ by interpolation [S2].
3.  **Cost Volume Construction:** Calculate the variance of all projected features for each vertex $v$ to build the cost volume $B$:
    $$ B(v) = \operatorname{Var}\left( \{F_i(v)\}_{i=0}^{N-1} \right) $$
    where $\operatorname{Var}$ computes the variance across the $N$ feature vectors [S2].
4.  **Geometry Encoding Volume Generation:** Apply a sparse 3D CNN $\Psi$ to the cost volume $B$ to obtain the geometry encoding volume $M$:
    $$ M = \Psi(B) $$
    [S2].
5.  **Surface Extraction (Coarse):** For an arbitrary 3D location $q$, apply positional encoding $\operatorname{PE}$ to its 3D coordinates. Interpolate the features from the coarse geometry encoding volume $M_{\text{coarse}}(q)$. Concatenate $\operatorname{PE}(q)$ and $M_{\text{coarse}}(q)$ and pass them through the coarse SDF prediction network $f_{\theta, \text{coarse}}$ to predict the SDF $s(q)$ [S2].
6.  **Surface Extraction (Fine):** Use the coarse geometry to guide the fine level. Interpolate features from the fine geometry encoding volume $M_{\text{fine}}(q)$ (which is constructed similarly to $M_{\text{coarse}}$ but at higher resolution). Concatenate $\operatorname{PE}(q)$ and $M_{\text{fine}}(q)$ and pass them through the fine SDF prediction network $f_{\theta, \text{fine}}$ to predict the refined SDF $s(q)$ [S2].

#### 2. Appearance Prediction
This stage predicts colors and renders synthesized views [S2].

1.  **Color Blending:** Use a multi-scale color blending module to predict colors by aggregating information from the input images. The module uses a patch-based blending strategy with a patch size of $5 \times 5$ [S2].
2.  **Volume Rendering:** Combine the estimated geometry (SDF) with the predicted colors to render synthesized views using volume rendering [S2]. The rendering loss is minimized between the rendered view and the ground truth view [S2].

#### 3. Per-Scene Fine-Tuning
This stage improves the obtained geometry with fine-grained details [S2].

1.  **Consistency-Aware Loss:** Introduce a consistency-aware fine-tuning scheme to control inconsistent regions caused by occlusion and noise. The loss function includes a rendering loss and a consistency term that encourages consistent SDF values for points behind visible surfaces [S2].
2.  **Optimization:** Optimize the model using the same $N$ input images without adding new images. The optimization runs for 10k iterations [S2].
3.  **SDF Rescaling:** Use a hyperparameter $\tau$ to rescale the SDF value of sampled points behind visible surfaces. This encourages SDF values to be far from 0, avoiding uncontrollable free surfaces when extracting the 0-level set [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SparseNeuS:
    def __init__(self, N, coarse_res=96, fine_res=192, patch_size=5):
        """
        Initialize SparseNeuS.
        :param N: Number of input images.
        :param coarse_res: Resolution of coarse geometry encoding volume.
        :param fine_res: Resolution of fine geometry encoding volume.
        :param patch_size: Spatial size of patches for color blending.
        """
        self.N = N
        self.coarse_res = coarse_res
        self.fine_res = fine_res
        self.patch_size = patch_size
        
        # Feature extraction network (FPN)
        self.feature_extractor = FeaturePyramidNetwork()
        
        # Sparse 3D CNN for geometry encoding volume
        # Implemented with U-Net-like architecture using torchsparse
        self.sparse_cnn = Sparse3DCNN()
        
        # SDF prediction networks
        self.sdf_coarse = SDFPredictionMLP()
        self.sdf_fine = SDFPredictionMLP()
        
        # Color blending module
        self.color_blender = MultiScaleColorBlending(patch_size)
        
        # Hyperparameter for SDF rescaling
        self.tau = 1.0  # Value not specified in sources, placeholder [G]

    def extract_features(self, images):
        """
        Extract 2D feature maps from input images.
        :param images: Tensor of shape (N, C, H, W)
        :return: List of feature maps {Fi}
        """
        return self.feature_extractor(images)

    def project_features(self, vertices, feature_maps, camera_params):
        """
        Project vertices to feature maps and interpolate features.
        :param vertices: Tensor of shape (V, 3) containing 3D coordinates of bounding box vertices.
        :param feature_maps: List of feature maps {Fi} of shape (N, C, H, W).
        :param camera_params: Camera parameters for each image.
        :return: Tensor of shape (V, N, C) containing projected features Fi(v).
        """
        projected_features = []
        for i in range(self.N):
            # Project vertex v to pixel location pi_i(v)
            pixel_locs = project_to_pixel(vertices, camera_params[i])
            # Interpolate features at pixel locations
            # Fi(v) = Fi(pi_i(v))
            features = interpolate_features(feature_maps[i], pixel_locs)
            projected_features.append(features)
        return torch.stack(projected_features, dim=1) # Shape: (V, N, C)

    def build_cost_volume(self, projected_features):
        """
        Build cost volume B by computing variance of projected features.
        :param projected_features: Tensor of shape (V, N, C)
        :return: Cost volume B of shape (V, 1) or (V, C) depending on variance definition.
                 Sources say Var of all projected features, implying scalar variance per vertex.
        """
        # Variance across the N dimension
        B = torch.var(projected_features, dim=1, keepdim=True)
        return B

    def build_geometry_encoding_volume(self, cost_volume):
        """
        Apply sparse 3D CNN to cost volume to get geometry encoding volume M.
        :param cost_volume: Cost volume B.
        :return: Geometry encoding volume M.
        """
        M = self.sparse_cnn(cost_volume)
        return M

    def predict_sdf_coarse(self, vertices, M_coarse):
        """
        Predict SDF using coarse geometry encoding volume.
        :param vertices: 3D coordinates.
        :param M_coarse: Coarse geometry encoding volume.
        :return: SDF values.
        """
        # Interpolate features from M_coarse at vertices
        features = interpolate_volume(M_coarse, vertices)
        # Positional encoding
        pe = positional_encoding(vertices)
        # Concatenate and predict
        input_tensor = torch.cat([pe, features], dim=-1)
        sdf = self.sdf_coarse(input_tensor)
        return sdf

    def predict_sdf_fine(self, vertices, M_fine):
        """
        Predict SDF using fine geometry encoding volume.
        :param vertices: 3D coordinates.
        :param M_fine: Fine geometry encoding volume.
        :return: SDF values.
        """
        features = interpolate_volume(M_fine, vertices)
        pe = positional_encoding(vertices)
        input_tensor = torch.cat([pe, features], dim=-1)
        sdf = self.sdf_fine(input_tensor)
        return sdf

    def render(self, rays, sdf, colors):
        """
        Volume rendering.
        :param rays: Ray origins and directions.
        :param sdf: SDF values at sampled points along rays.
        :param colors: Colors at sampled points.
        :return: Rendered image.
        """
        # Standard volume rendering with SDF-based opacity
        # Alpha is derived from SDF
        alphas = sdf_to_alpha(sdf)
        weights = compute_weights(alphas)
        rendered_color = sum(weights * colors)
        return rendered_color

    def train_generic(self, images, camera_params, iterations=150000):
        """
        Stage 1: Train coarse level networks.
        Stage 2: Train fine level networks while keeping coarse fixed.
        """
        # Extract features
        feature_maps = self.extract_features(images)
        
        # Define bounding box and vertices
        vertices = get_bounding_box_vertices(camera_params[0], self.coarse_res)
        
        # Project features
        projected_features = self.project_features(vertices, feature_maps, camera_params)
        
        # Build cost volume
        cost_volume = self.build_cost_volume(projected_features)
        
        # Build coarse geometry encoding volume
        M_coarse = self.build_geometry_encoding_volume(cost_volume)
        
        # Train coarse SDF network
        for i in range(iterations):
            # Sample points
            points = sample_points(vertices)
            # Predict SDF
            sdf = self.predict_sdf_coarse(points, M_coarse)
            # Compute loss (rendering loss)
            loss = compute_rendering_loss(sdf, images, camera_params)
            loss.backward()
            self.sdf_coarse.step()
            
        # Build fine geometry encoding volume (guided by coarse)
        # Note: Sources say fine volume is constructed similarly, guided by coarse
        # Implementation detail: Re-project or refine vertices based on coarse SDF?
        # Sources say "coarse level to refine the geometry", implying the fine volume
        # is built on a refined bounding box or with higher resolution.
        vertices_fine = get_bounding_box_vertices(camera_params[0], self.fine_res)
        projected_features_fine = self.project_features(vertices_fine, feature_maps, camera_params)
        cost_volume_fine = self.build_cost_volume(projected_features_fine)
        M_fine = self.build_geometry_encoding_volume(cost_volume_fine)
        
        # Train fine SDF network
        for i in range(iterations):
            points = sample_points(vertices_fine)
            sdf = self.predict_sdf_fine(points, M_fine)
            loss = compute_rendering_loss(sdf, images, camera_params)
            loss.backward()
            self.sdf_fine.step()

    def fine_tune(self, images, camera_params, iterations=10000):
        """
        Per-scene fine-tuning.
        """
        # Use the same three input images
        # Optimize model (likely just the SDF networks or specific parameters)
        for i in range(iterations):
            # Sample points
            points = sample_points()
            # Predict SDF
            sdf = self.predict_sdf_fine(points, M_fine)
            # Compute consistency-aware loss
            loss = compute_consistency_aware_loss(sdf, images, camera_params, self.tau)
            loss.backward()
            # Update parameters
            # Note: Sources say "Optimizes the model", implying some parameters are updated
            # It is not specified which parameters are frozen. [G]
            # Assuming SDF networks are updated.
            self.sdf_fine.step()

class FeaturePyramidNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # FPN implementation
        pass
    def forward(self, x):
        return self.feature_maps

class Sparse3DCNN(nn.Module):
    def __init__(self):
        super().__init__()
        # U-Net-like architecture using torchsparse
        pass
    def forward(self, x):
        return self.output

class SDFPredictionMLP(nn.Module):
    def __init__(self):
        super().__init__()
        # MLP layers
        pass
    def forward(self, x):
        return self.sdf

def positional_encoding(x):
    # Standard positional encoding
    pass

def interpolate_features(feature_map, pixel_locs):
    # Bilinear interpolation
    pass

def interpolate_volume(volume, points):
    # Trilinear interpolation
    pass

def sdf_to_alpha(sdf):
    # Convert SDF to opacity
    pass

def compute_weights(alphas):
    # Compute ray marching weights
    pass

def compute_rendering_loss(sdf, images, camera_params):
    # Render and compare with ground truth
    pass

def compute_consistency_aware_loss(sdf, images, camera_params, tau):
    # Rendering loss + consistency term
    # Consistency term encourages SDF values behind visible surfaces to be far from 0
    # using tau to rescale
    pass
```

### Parameters
*   $N$: Number of input images. Typically 2 or 3 [S2].
*   $\Psi$: Sparse 3D CNN used to aggregate the cost volume [S2].
*   $\tau$: Hyperparameter to rescale the SDF value of sampled points behind visible surfaces. Value: Not specified in the sources [S2].
*   `batch_size`: Number of rays processed per training iteration. Value: 512 [S2].
*   `coarse_volume_resolution`: 3D grid resolution for the coarse geometry encoding volume. Value: $96 \times 96 \times 96$ [S2].
*   `fine_volume_resolution`: 3D grid resolution for the fine geometry encoding volume. Value: $192 \times 192 \times 192$ [S2].
*   `patch_size`: Spatial size of patches used in the patch-based blending strategy. Value: $5 \times 5$ [S2].
*   `coarse_training_iterations`: Number of iterations for the first training stage. Value: 150k [S2].
*   `fine_training_iterations`: Number of iterations for the second training stage. Value: 150k [S2].
*   `fine_tuning_iterations`: Number of iterations for per-scene fine-tuning. Value: 10k [S2].
*   `fine_tuning_time`: Approximate wall-clock time for 10k iteration fine-tuning. Value: 20 mins [S2].

### Edge Cases and Pitfalls
*   **Radiance Ambiguity:** With sparse views (e.g., 3 images), radiance ambiguity can cause distorted and incomplete geometry in methods like [NeuS](6060-neus.md) and [MVSNerf](9325-hame-nerf.md). SparseNeuS mitigates this through generalizable priors and consistency-aware fine-tuning [S2].
*   **Weak Textures:** Conventional MVS methods like COLMAP fail with weak textures. SparseNeuS produces more complete meshes for such objects [S2].
*   **Background Noise:** DTU dataset images contain large black backgrounds with considerable noise. SparseNeuS uses a simple threshold on zero RGB values as a prior to encourage empty geometry predictions in these regions [S2].
*   **Per-Scene Optimization Requirements:** Unlike [IDR](6060-neus.md), SparseNeuS does not require extra object masks for per-scene optimization [S2]. Unlike [MVSNerf](9325-hame-nerf.md), SparseNeuS successfully fine-tunes with only three input images, whereas MVSNerf fails under the same condition [S2].

## Subtopics

### [SparseNeuS Per-Scene Fine-Tuning](18088-sparseneus-per-scene-fine-tuning.md)
A stage that further improves the obtained geometry with fine-grained details using a consistency-aware fine-tuning scheme. It optimizes the model using the same input images without adding new ones, running for 10k iterations.

## Operating characteristics
*   **Time Complexity:** The generic model training involves two stages of 150k iterations each. Per-scene fine-tuning takes 10k iterations, approximately 20 minutes [S2].
*   **Memory Complexity:** Uses sparse 3D CNNs with resolutions $96 \times 96 \times 96$ and $192 \times 192 \times 192$. Memory efficiency is improved by using center-cropped images ($512 \times 640$) for training and sparse convolutions [S2].
*   **Performance:** Outperforms state-of-the-art methods (PixelNeRF, IBRNet, MVSNerf, IDR, NeuS, VolSDF, UniSurf, COLMAP) on DTU and BlendedMVS datasets with 3 input images [S2].
*   **Strengths:** Generalizes to new scenes, works with sparse views (2-3 images), produces high-quality complete meshes, avoids radiance ambiguity issues of [NeuS](6060-neus.md) and [MVSNerf](9325-hame-nerf.md), does not require object masks [S2].
*   **Limitations:** Requires known camera parameters. Performance depends on the quality of the 2D feature extraction network [S2].
*   **When to Use:** When only sparse multi-view images are available and generalization to new scenes is required.
*   **When Not to Use:** When dense views are available and per-scene optimization is acceptable (e.g., [NeuS](6060-neus.md) might be simpler). When camera parameters are unknown [S2].

## Background
SparseNeuS was introduced by Xiaoxiao Long, Cheng Lin, Peng Wang, Taku Komura, and Wenping Wang from The University of Hong Kong, Tencent Games, and Texas A&M University. It was published on arXiv on 2 Aug 2022 [S2]. The method was trained on 75 non-overlapping scenes from the DTU dataset and tested on 15 DTU scenes and 7 challenging scenes from the BlendedMVS dataset [S2]. It addresses the limitations of existing neural reconstruction approaches that heavily depend on dense views and require time-consuming per-scene optimization, lacking generalization [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
