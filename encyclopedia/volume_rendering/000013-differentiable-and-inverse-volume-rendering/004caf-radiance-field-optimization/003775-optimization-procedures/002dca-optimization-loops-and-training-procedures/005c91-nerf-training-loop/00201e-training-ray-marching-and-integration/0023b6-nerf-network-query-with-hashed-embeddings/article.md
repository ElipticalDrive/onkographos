# NeRF Network Query with Hashed Embeddings

Also called: HashNeRF query, Instant-NGP NeRF, NeRF with Hash Encoding, NeRF with Multiresolution Hash Encoding, Neural Radiance and Density Fields (NeRF) with Multiresolution Hash Encoding

## Overview
This method queries a neural radiance field by mapping 3D spatial coordinates and 2D view directions into high-dimensional feature vectors using a multiresolution hash encoding, which are then processed by concatenated Multi-Layer Perceptrons (MLPs) to predict volume density and view-dependent color [S2]. It sets itself apart from standard NeRF [Mildenhall et al. 2020] by replacing expensive positional encoding with a learnable, sparse hash grid that enables significantly faster training convergence and real-time rendering performance on consumer hardware [S2].

## How it works
**Inputs and Outputs**
*   **Inputs:** A batch of 3D positions $\mathbf{x} \in \mathbb{R}^{N \times 3}$ and view directions $\mathbf{d} \in \mathbb{R}^{N \times 3}$.
*   **Outputs:** Volume density $\sigma \in \mathbb{R}^{N}$ and RGB color $\mathbf{c} \in \mathbb{R}^{N \times 3}$.

**Data Structures**
*   **Hash Grid:** A collection of $L$ resolution levels. Each level $l$ consists of a hash table mapping integer coordinates to feature vectors.
    *   **Hash Table:** A flat array of size $2^{22}$ (or similar capacity) storing $F$-dimensional feature vectors. The hash function maps 3D grid coordinates to table indices.
    *   **Feature Vector:** A vector of size $F$ (typically $F=2$) stored in the hash table.
*   **MLP Weights:**
    *   **Density MLP:** 1 hidden layer, 64 neurons.
    *   **Color MLP:** 2 hidden layers, 64 neurons.

**Numbered Steps**
1.  **Reshape Inputs:** Flatten the input positions to shape $[-1, 3]$ [S1, unofficial].
2.  **Position Embedding:** Apply the multiresolution hash encoding function `embed_fn` to the positions. This involves:
    *   For each resolution level $l$, discretizing the position into grid coordinates.
    *   Hashing these coordinates to find indices in the level-$l$ hash table.
    *   Trilinearly interpolating the $F$-dimensional features from the four nearest neighbors in the hash table.
    *   Concatenating the interpolated features from all $L$ levels along the last dimension to form the position embedding $\mathbf{y}$ [S2].
    *   Generate a validity mask `keep_mask` to identify valid points (e.g., within bounds) [S1, unofficial].
3.  **Direction Embedding:** If view directions are provided:
    *   Expand directions to match the input batch shape and reshape to flat $[-1, 3]$ [S1, unofficial].
    *   Project the view direction onto the first 16 coefficients of the spherical harmonics basis (up to degree 4) [S2].
    *   Apply direction embedding `embeddirs_fn` (which may include additional hashing or fixed encoding) [S1, unofficial].
4.  **Concatenate Embeddings:** Concatenate the position embedding $\mathbf{y}$ and the direction embedding along the last dimension to form the full input vector for the network [S1, unofficial].
5.  **Batched Network Evaluation:** Apply the network function `fn` in batches of size `netchunk` (default 1024*64) to avoid Out-Of-Memory (OOM) errors [S1, unofficial].
    *   **Density MLP:** Input is $\mathbf{y}$. Output is 16 values. The first value is treated as log-space density $\log(\sigma)$ [S2].
    *   **Color MLP:** Input is the concatenation of the 16 density MLP outputs and the spherical harmonics projected view direction [S2].
    *   **Color Output:** Apply Sigmoid activation for low dynamic-range (sRGB) data or Exponential activation for high dynamic range (linear HDR) data [S2].
6.  **Mask Invalid Points:** Set density $\sigma$ to 0 for points where `~keep_mask` is true [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class HashNeRFQuery:
    def __init__(self, hash_table_size, num_levels, level_dim, hidden_dim=64):
        """
        Initialize the HashNeRF query module.
        :param hash_table_size: Size of the hash table (e.g., 2^22)
        :param num_levels: Number of resolution levels (L)
        :param level_dim: Dimension of features per level (F, typically 2)
        :param hidden_dim: Number of neurons in MLP hidden layers
        """
        self.num_levels = num_levels
        self.level_dim = level_dim
        self.hash_table_size = hash_table_size
        
        # Initialize hash tables for each level
        # Each table maps grid coords to F-dimensional features
        self.embeddings = nn.ModuleList([
            nn.Embedding(hash_table_size, level_dim) for _ in range(num_levels)
        ])
        
        # Initialize MLPs
        # Density MLP: 1 hidden layer, 64 neurons
        # Input dim: L * F
        # Output dim: 16
        input_dim = num_levels * level_dim
        self.density_mlp = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 16)
        )
        
        # Color MLP: 2 hidden layers, 64 neurons
        # Input dim: 16 (from density MLP) + 16 (SH directions)
        # Output dim: 3 (RGB)
        self.color_mlp = nn.Sequential(
            nn.Linear(16 + 16, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3)
        )
        
        # Activation for HDR or LDR
        self.color_activation = nn.Sigmoid() # Or nn.Softplus/Exp for HDR

    def hash_encode(self, x):
        """
        Multiresolution hash encoding for 3D positions.
        :param x: Tensor of shape [N, 3]
        :return: Concatenated features from all levels [N, L*F]
        """
        features = []
        for l in range(self.num_levels):
            # Discretize position to grid coordinates for level l
            # This step depends on the specific grid resolution for level l
            # Assuming a helper function `get_grid_coords` exists
            grid_coords = get_grid_coords(x, level=l)
            
            # Hash coordinates to indices
            indices = hash_function(grid_coords, self.hash_table_size)
            
            # Gather features from hash table
            # Note: In practice, trilinear interpolation is used.
            # For simplicity, we assume direct lookup or a placeholder for interpolation
            # [G] Trilinear interpolation requires fetching 8 neighbors and weights.
            # Here we use direct lookup for brevity, but real implementation interpolates.
            feat = self.embeddings[l](indices)
            features.append(feat)
        
        # Concatenate features from all levels
        return torch.cat(features, dim=-1)

    def directional_encoding(self, d):
        """
        Spherical Harmonics encoding for view directions.
        :param d: Tensor of shape [N, 3]
        :return: Tensor of shape [N, 16]
        """
        # Project direction onto first 16 coefficients of SH basis (degree 4)
        # [G] This involves computing SH basis functions for the direction vector.
        sh_coeffs = compute_spherical_harmonics(d, degree=4)
        return sh_coeffs[:, :16]

    def forward(self, x, d):
        """
        Forward pass for NeRF query.
        :param x: Positions [N, 3]
        :param d: View directions [N, 3]
        :return: Density [N], Color [N, 3]
        """
        # 1. Position Embedding
        pos_feat = self.hash_encode(x)
        
        # 2. Direction Embedding
        dir_feat = self.directional_encoding(d)
        
        # 3. Concatenate
        input_feat = torch.cat([pos_feat, dir_feat], dim=-1)
        
        # 4. Density MLP
        density_out = self.density_mlp(input_feat)
        log_density = density_out[:, 0]
        density = torch.exp(log_density)
        
        # 5. Color MLP
        # Input: 16 density outputs + 16 SH directions
        color_input = torch.cat([density_out, dir_feat], dim=-1)
        color_out = self.color_mlp(color_input)
        color = self.color_activation(color_out)
        
        return density, color
```

**Parameters**
*   **Density MLP:** 1 hidden layer, 64 neurons [S2].
*   **Color MLP:** 2 hidden layers, 64 neurons [S2].
*   **View Direction Encoding:** First 16 coefficients of spherical harmonics (degree 4) [S2].
*   **Color Activation:** Sigmoid for sRGB, Exponential for linear HDR [S2].
*   **Batch Size (`netchunk`):** Default 1024*64 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Hash Collisions:** Cause "grainy" microstructure artifacts, especially visible in learned signed distance functions. The MLP cannot fully compensate for this [S2].
*   **Concatenation vs. Reduction:** Concatenating $F$-dimensional features from each resolution is preferred over reduction (e.g., summing) because reduction may result in dimensionality $L \times F$ being too small to encode useful information [S2].
*   **Linear Replacement:** Replacing the MLP with a single linear matrix multiplication compromises quality significantly, failing to capture specular effects and resolve hash collisions, despite being cheaper [S2].

## Subtopics

### [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md)
HybridNeRF is a neural radiance field architecture that combines dense multi-resolution 3D feature grids with multi-resolution triplanes to efficiently featurize 3D sample locations. It solves the trade-off between the high memory cost of pure 3D grids and the limited geometric expressiveness of pure 2D triplanes by using a hybrid representation. It sets itself apart from its sibling [MVS Network Query](7517-mvs-network-query.md) by explicitly summing features across resolution levels rather than concatenating them, and from [SqueezeNeRF](8704-squeezenerf.md) by utilizing a spatially adaptive surfaceness field for density estimation.

### [MVS Network Query](7517-mvs-network-query.md)
MVS Network Query prepares inputs for a neural network by embedding points and view directions, then applies the network in batches. It differs from standard NeRF network queries by explicitly handling multi-view geometric consistency through the generation of angle and point features derived from camera poses and depth candidates. It builds on [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md) and relates to [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md), [Simple-ZipNeRF](7424-simple-zipnerf.md), [VoxNeRF](8865-voxnerf.md), [Deformation Field Integration for Telepresence](10866-deformation-field-integration-for-telepresence.md), and [Multi-resolution Hash Encoding](13125-multi-resolution-hash-encoding.md).

### SqueezeNeRF
SqueezeNeRF is a memory-efficient neural rendering method that further factorizes the position-dependent Multi-Layer Perceptron (MLP) of FastNeRF into three separate 2D networks operating on coordinate pairs $(x,y)$, $(y,z)$, and $(z,x)$ to reduce cache size complexity from $O(n^3)$ to $O(n^2)$ [S1, unofficial]. It builds on [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md) by replacing the dense 3D position cache with a set of 2D position caches and a 2D direction cache, enabling real-time inference with significantly lower memory overhead than FastNeRF or NeRF-SH while maintaining high rendering speeds [S1, unofficial].

### VoxNeRF
VoxNeRF is a neural rendering method that integrates explicit scene geometry priors (mesh, point cloud, or depth map) with implicit radiance field representation to improve sampling efficiency and reconstruction accuracy [S1, unofficial]. It addresses the ill-posed nature of recovering 3D geometry from 2D images by using a Sparse Voxel Octree (SVO) to model geometric uncertainty, which guides a hybrid sampling strategy [S1, unofficial]. Unlike standard NeRF methods that sample uniformly or via stratified noise, VoxNeRF concentrates samples near the estimated surface while maintaining coverage of free space, reducing computational waste in empty regions [S1, unofficial].

### [Deformation Field Integration for Telepresence](10866-deformation-field-integration-for-telepresence.md)
This method extends the FastNeRF architecture to support dynamic, expression-driven head avatars for telepresence by integrating a learnable deformation field network ($F_{deform}$) and a non-trainable skeletal deformation field ($F_{bone}$) into the ray marching process. It solves the problem of rendering animatable 3D heads from static multi-view images by warping sample positions based on an expression vector before querying the radiance field. It is distinguished from static NeRF variants by its explicit handling of temporal expression parameters and from other dynamic NeRF methods by its specific integration with the FastNeRF caching infrastructure and grid-based pruning.

### [HDR-HexPlane](13548-hdr-hexplane.md)
HDR-HexPlane is an end-to-end neural radiance field framework for dynamic scenes that learns 3D scene representations from dynamic 2D images captured with various exposures, allowing for novel view synthesis with adjustable exposure levels. It explicitly separates the modeling of dynamic scene geometry/radiance from the learning of scene illumination (exposure), addressing color inconsistency caused by objects moving between dark and bright regions. Unlike static HDR-NeRF methods, HDR-HexPlane handles dynamic scenes by incorporating temporal information (timestamp $t$) into the HexPlane representation.

### [Multi-scale PointNeRF++ Feature Aggregation](13651-multi-scale-pointnerf-feature-aggregation.md)
Multi-scale PointNeRF++ is a method for obtaining radiance ($\rho_q$) and density ($\sigma_q$) for a query point $q$ by operating on a sparse, hierarchical point cloud representation at multiple granularity levels. It solves the problem of rendering in empty regions without relying on failure-prone region-growing heuristics. Unlike its sibling [PointNeRF](link-to-pointnerf), this method naturally fills empty regions by averaging feature embeddings over valid scale levels where neighbors exist. It builds on [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md) by replacing hash grids with a multi-scale point cloud hierarchy.

### FourieRF
FourieRF is a few-shot Neural Radiance Field (NeRF) estimation method that prevents high-frequency artifacts and catastrophic overfitting by parameterizing grid-based features with a Fourier Transform prior [S1, unofficial]. Unlike sibling methods that rely on external data priors or complex occlusion regularization, FourieRF uses an explicit curriculum training procedure to progressively increase the maximum learnable Fourier frequency, ensuring stable convergence on sparse data [S1, unofficial]. It builds on [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md) by applying frequency control to the feature grids of grid-based NeRFs (specifically TensoRF) rather than MLP positional encodings [S1, unofficial].

### [Hash-grid anti-aliasing and regularization](15594-hash-grid-anti-aliasing-and-regularization.md)
This family encompasses techniques that modify the hash-grid encoding mechanism in neural radiance fields to mitigate artifacts such as high-frequency aliasing and sparse-view floaters. It differs from standard hash-grid encoding by introducing either capacity constraints (smaller hash tables) to enforce feature sharing and geometric consistency, or statistical down-weighting factors to suppress high-frequency noise during rendering.

### [NeRF-W](15666-nerf-w.md)
NeRF-W extends Neural Radiance Fields to handle unconstrained internet photo collections by modeling temporal modifications, specifically variable illuminations and transient objects (people, cars, occlusions). It distinguishes itself from standard NeRF by introducing per-image appearance embeddings to handle photometric variations and a secondary transient MLP head to model dynamic content, allowing the rendering of static geometry while accounting for scene changes.

### [UniSDF-based Latent NeRF](17022-unisdf-based-latent-nerf.md)
A geometry estimator and appearance model using UniSDF, a state-of-the-art volume rendering approach based on a signed distance function (SDF), which decouples geometry from appearance and allows mesh extraction for radiance cue computation.

### RawNeRF
A method that modifies Neural Radiance Fields (NeRF) to reconstruct scenes in linear HDR color space by supervising directly on noisy raw input images, rather than postprocessed low dynamic range (LDR) images.

### [Neural Duplex Radiance Field](17643-neural-duplex-radiance-field.md)
A radiance field representation that uses two geometric proxy meshes (inner and outer) to define a ray segment. Instead of dense volumetric sampling, it learns to map the two endpoints of the ray segment (intersections with the inner and outer meshes) and the view direction to the final pixel color, reducing the number of sampled points per ray to two.

### Stratified PDF Sampling for NeRF
A method to resample points along a ray based on the importance (weight) of previous samples, improving detail in high-density regions [S3, unofficial]. It normalizes bin weights by adding epsilon (1e-5) and dividing by sum to create a PDF, computes the Cumulative Distribution Function (CDF) by cumsumming the PDF and padding with zeros at the start, generates uniform random values $u$ in $[0, 1]$ (or deterministic midpoints if `det=True`), finds indices in the CDF where $u$ falls using `searchsorted`, identifies lower and upper bin indices for interpolation, and interpolates the final sample position within the bin using linear interpolation based on $u$'s position in the CDF interval [S3, unofficial].

### MikeLee Method
A sparse neural rendering method that explicitly conditions the color prediction network on local features extracted from a pretrained network to improve color and feature consistency [S4]. It extracts local features $F_{gt}(r)$ from a pretrained VGG network's Relu1-1 layer for each ray $r$, concatenates bottleneck feature $b_i$, prior feature $f_i$, and view direction $d$, and predicts color $c_i = M_c(b_i, f_i, d)$ [S4]. The loss function is $L = L_c + \lambda L_f$, where $L_f = || F(r) - F^{gt}(r) ||_2$ [S4]. It is trained at low resolution for 44K iterations with a batch size of 1024 due to memory limitations, and frequency regulation ends at 40K iterations [S4]. Using deeper VGG layers introduces interpolation artifacts because their output size differs from the input image; Relu1-1 is used because it is the same size as the input image [S4].

### Appearance Matching
A post-training optimization step to ensure consistent appearance across adjacent Block-NeRFs by matching an appearance latent code from a target Block-NeRF to its neighbors [S5]. It selects a 3D matching location between pairs of adjacent Block-NeRFs where visibility prediction is high for both blocks, freezes the Block-NeRF network weights, and optimizes only the appearance code of the adjacent Block-NeRF to match the target appearance [S5]. Without appearance matching, randomly initialized appearance codes lead to different appearances in different blocks, causing inconsistencies during compositing [S5].

### Shadow-Aware Irradiance Model
A rendering model that computes NeRF-based color by combining albedo, a shading scalar, and ambient color, replacing the standard transmittance/opacity accumulation with a shadow-aware irradiance formulation to handle illumination inconsistency [S6]. The rendered color is $I_{p1} = \rho(P) \times (s(P, w) + (1 - s(P, w)) \times a(w))$, where $\rho(P)$ is the albedo of surface point $P$, $s(P, w)$ is the shading scalar (values 0 to 1), and $a(w)$ is the ambient color indicating shadowed areas [S6]. The NeRF-based rendering color is $C(r) = \sum_{i=1}^{N} T_i o_i s_{vi}$, where $C(r)$ is the rendered color, $o_i$ is opacity, $T_i$ is transmittance, and $s_{vi}$ is the shadow value at the $i$-th sampled point [S6]. Opacity is $o_i = 1 - \exp(-\sigma_i \delta_i)$, where $\sigma_i$ is the density value at the sampled point and $\delta_i$ is the distance between adjacent sampled points along the ray [S6]. Transmittance is $T_i = \prod_{j=1}^{i-1} (1 - o_j)$ [S6].

## Operating characteristics
*   **Time Complexity:** Training time is significantly reduced compared to standard NeRF. For example, accumulating 128 samples at 1080p on a single RTX 3090 GPU takes approximately 5 seconds for a modular synthesizer scene [S2].
*   **Memory Complexity:** The hash grid memory usage is determined by the hash table size and the number of levels. It is generally more memory-efficient than dense 3D grids for large scenes but can still be substantial depending on the number of levels and feature dimensions.
*   **Measured Performance:** Rendering performance can reach 10 frames per second on an RTX 3090 GPU for a large natural 360 scene [S2].
*   **Scaling:** Performance scales well with hardware acceleration due to the efficient implementation of the hash encoding and MLPs.
*   **Strengths:** Fast training convergence, real-time rendering capabilities, ability to handle large scenes with occupancy grid pruning.
*   **Limitations:** Hash collisions can cause grainy artifacts. The quality is significantly compromised if the MLP is replaced with a simple linear lookup [S2].
*   **When to Use:** When real-time rendering or fast training is required, and the scene can benefit from sparse representations.
*   **When Not to Use:** When high-fidelity specular details are critical and hardware constraints prevent the use of the MLP component, or when the scene is very small and the overhead of hash encoding is not justified.

## Background
The method originates from the "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding" paper by Müller et al. [S2]. It builds on the original NeRF architecture by Mildenhall et al. [S2], which uses positional encoding and concatenated MLPs. The key innovation is the introduction of a learnable multiresolution hash encoding that maps 3D coordinates to feature vectors, enabling efficient training and rendering. The method has been widely adopted in various NeRF variants and extensions, such as [HybridNeRF](4842-hybridnerf-model-architecture.md), [SqueezeNeRF](8704-squeezenerf.md), and [VoxNeRF](8865-voxnerf.md).

## Sources
[S1] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S2] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S3] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S4] AIM 2024 Sparse Neural Rendering Challenge: Methods and Results, https://doi.org/10.1007/978-3-031-91856-8_2, fetched 2026-10-27
[S5] Block-NeRF: Scalable Large Scene Neural View Synthesis, https://doi.org/10.1109/cvpr52688.2022.00807, fetched 2026-10-26
[S6] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
