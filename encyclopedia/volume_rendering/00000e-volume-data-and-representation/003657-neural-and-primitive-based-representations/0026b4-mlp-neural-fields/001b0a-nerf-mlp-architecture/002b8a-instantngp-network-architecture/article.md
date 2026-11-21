# InstantNGP Network Architecture

Also called: Instant Neural Graphics Primitives Network, InstantNGP MLP, MonoSDF Network, Network Architecture and Implementation

## Overview
InstantNGP is a neural field architecture designed for real-time training and rendering of 3D scenes by replacing traditional positional encoding with a multiresolution hash grid [S1, unofficial]. It solves the bottleneck of MLP evaluation latency in NeRF by leveraging hardware-accelerated memory locality and bilinear interpolation [S1, unofficial]. Unlike the parent [NeRF MLP Architecture](910-nerf-mlp-architecture.md) which maps raw coordinates to density and color, InstantNGP first encodes spatial positions into a high-dimensional feature vector via hash lookups, which are then processed by a lightweight MLP [S1, unofficial]. This approach enables training convergence in seconds rather than hours, distinguishing it from standard NeRF variants [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinate $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\mathbf{d} \in \mathbb{R}^3$.
*   **Output:** Volume density $\sigma \in \mathbb{R}$ and view-dependent color $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
The core data structure is a multiresolution hash grid consisting of $L$ levels.
*   **Hash Grid:** For each level $l \in \{0, \dots, L-1\}$, a hash table maps 3D coordinates to feature vectors.
    *   **Hash Function:** $h(\mathbf{x}, l)$ maps the coordinate and level to an index in the hash table. The hash function handles the discretization of $\mathbf{x}$ to the grid resolution $N_l$ at level $l$.
    *   **Feature Vector:** Each entry in the hash table stores a learnable feature vector $\mathbf{f} \in \mathbb{R}^F$. The feature dimension $F$ is typically small (e.g., 2 or 4) to fit in cache.
    *   **Interpolation:** Features from the 8 neighboring grid vertices of $\mathbf{x}$ at level $l$ are retrieved and interpolated using trilinear interpolation to produce the level-specific feature vector $\mathbf{f}_l(\mathbf{x})$ [S1, unofficial].
*   **Concatenation:** The features from all levels are concatenated: $\mathbf{f}_{total} = [\mathbf{f}_0(\mathbf{x}); \dots; \mathbf{f}_{L-1}(\mathbf{x})]$.

**Parameters:**
*   **Hash Map Size:** The size of the hash maps for all NGPs is $2^{19}$ [S3].
*   **Optimizer:** Adam optimizer [S1, unofficial].
*   **Learning Rate Schedule:** $10^{-2} \to 10^{-4}$ [S1, unofficial].
*   **Iterations:** 20k iterations [S1, unofficial].
*   **Occupancy Grid:** Utilizes an occupancy grid to skip empty space during training [S1, unofficial].

**Algorithm Steps:**
1.  **Hash Encoding:** For the input position $\mathbf{x}$, compute the hash indices for each resolution level $l$. Retrieve the 8 corner features from the hash grid, perform trilinear interpolation to obtain $\mathbf{f}_l(\mathbf{x})$, and concatenate them into $\mathbf{f}_{total}$ [S1, unofficial].
2.  **Feature Processing:** Pass $\mathbf{f}_{total}$ through the density MLP to predict $\sigma$.
3.  **Directional Encoding:** Encode the viewing direction $\mathbf{d}$ (typically using spherical harmonics or positional encoding, though InstantNGP primarily focuses on the spatial hash grid, the direction is still processed by the color network) [S1, unofficial].
4.  **Color Prediction:** Concatenate the spatial features (or a subset thereof) with the encoded direction $\mathbf{d}$ and pass them through the color MLP to predict $\mathbf{c}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def instantngp_forward(x, d, hash_grids, density_mlp, color_mlp):
    """
    x: (3,) float tensor, 3D position
    d: (3,) float tensor, viewing direction
    hash_grids: list of L tensors, each of shape [2^19, F]
    density_mlp: MLP object
    color_mlp: MLP object
    """
    # 1. Hash Encoding
    f_total = []
    for l in range(num_levels):
        # Get grid resolution for level l
        N_l = get_resolution(l)
        # Compute hash indices for the 8 corners of x in grid l
        # This involves discretizing x to the grid and hashing
        corner_indices = get_hash_indices(x, l, N_l)
        
        # Retrieve features from hash grid
        corner_features = gather(hash_grids[l], corner_indices) # Shape: [8, F]
        
        # Compute trilinear interpolation weights based on fractional part of x
        weights = compute_trilinear_weights(x, l, N_l) # Shape: [8]
        
        # Interpolate to get level feature
        f_l = dot(weights, corner_features) # Shape: [F]
        f_total.append(f_l)
    
    # Concatenate features from all levels
    f_concat = concat(f_total) # Shape: [L * F]
    
    # 2. Density Prediction
    sigma = density_mlp(f_concat) # Scalar output
    
    # 3. Color Prediction
    # Encode direction d (implementation detail not fully specified in sources, 
    # typically involves positional encoding or spherical harmonics)
    d_encoded = encode_direction(d)
    
    # Concatenate spatial features and direction features
    # Note: Some variants may use a subset of f_concat or all of it
    input_color = concat(f_concat, d_encoded)
    color = color_mlp(input_color) # RGB output
    
    return sigma, color
```

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** With a finite hash map size ($2^{19}$), collisions are inevitable. The hash function must be robust to minimize performance degradation from collisions [S3].
*   **Negative Coordinates:** Standard hash grids are bounded to $[0, 1]^3$. Handling unbounded scenes requires modifications like negative integer mapping .
*   **Precision:** Trilinear interpolation requires careful handling of floating-point precision to avoid artifacts at grid boundaries [S1, unofficial].

## Subtopics

### [iNGP Texture Storage](5826-ingp-texture-storage.md)
This method replaces hash-table-based feature storage with explicit 3D grid and triplane textures to leverage hardware-accelerated memory locality and bilinear interpolation. It reduces the number of texture fetches per neural network evaluation by storing presummed multi-resolution features, enabling a 3× reduction in fetch overhead compared to the original InstantNGP approach. Choose this when hardware texture units are preferred over custom hash lookup kernels. Link: [iNGP Texture Storage](5826-ingp-texture-storage.md).

### [Tiny MLP for Density and Color Estimation](8555-tiny-mlp-for-density-and-color-estimation.md)
The Tiny MLP is a lightweight neural network component used in Rip-NeRF to estimate the density $\tau$ and view-dependent color $c$ of a sampled sphere $S$. It builds on [InstantNGP Network Architecture](5049-instantngp-network-architecture.md) by mapping the Ripmap-Encoding feature vector $f$ and the encoded view direction $d$ to radiance properties. It sets itself apart from sibling methods like [iNGP Texture Storage](5826-ingp-texture-storage.md) by using a dedicated small MLP for density/color prediction rather than relying solely on texture lookups for these specific outputs, and differs from [HyperINR](10461-hyperinr.md) by using a fixed-width MLP with specific activation constraints rather than hypernetworks. Link: [Tiny MLP for Density and Color Estimation](8555-tiny-mlp-for-density-and-color-estimation.md).

### [HyperINR](10461-hyperinr.md)
HyperINR is a hypernetwork architecture that directly predicts the weights of a compact Implicit Neural Representation (INR) conditioned on scene parameters, enabling interactive volumetric path tracing with inference times less than 1ms. It differs from standard INRs, which require millions of operations per forward pass and are too slow for interactive visualization, and from data interpolation methods, which operate on data values rather than network weights and are less accurate for unseen parameters. By using a shared synthesis function and interpolating encoding weights via an ensemble of multi-resolution hash encoders organized in a KD-tree, HyperINR achieves up to 100× faster inference than teacher models like CoordNet while maintaining high reconstruction quality. Link: [HyperINR](10461-hyperinr.md).

### [Hybrid Fourier-Hash Encoding](13021-hybrid-fourier-hash-encoding.md)
Hybrid Fourier-Hash Encoding is a neural geometry representation technique that combines low-dimensional Fourier positional encoding with learnable multi-scale hash features to improve the continuity of the signed distance function (SDF) and satisfy eikonal constraints. It addresses the non-smooth solutions and poor gradient continuity produced by hash encoding alone, which arises from the discontinuous nature of features on discrete grids. Unlike the baseline InstantNGP architecture which uses only hash encoding, this method decouples geometry from appearance by extracting features from an intermediate connected layer, ensuring hash encoding encodes only geometry. Link: [Hybrid Fourier-Hash Encoding](13021-hybrid-fourier-hash-encoding.md).

### [Unbounded Hash-Grid with Negative Integer Mapping](13889-unbounded-hash-grid-with-negative-integer-mapping.md)
This method extends the Instant-NGP hash-grid encoding to support unbounded input domains by modifying the hash function to handle negative integers without structured collisions. It solves the problem of applying neural field representations, originally designed for normalized $[0, 1]^d$ inputs, to large-scale scenes with arbitrary coordinate ranges. It differs from its parent [InstantNGP Network Architecture](5049-instantngp-network-architecture.md) by replacing the bounded hash lookup with a negative-integer-aware mapping and by computing positional gradients directly from interpolation weights rather than assuming non-differentiable sample positions. Link: [Unbounded Hash-Grid with Negative Integer Mapping](13889-unbounded-hash-grid-with-negative-integer-mapping.md).

### [INGeo](15132-ingeo.md)
INGeo is a method that accelerates 3D scene reconstruction training by leveraging noisy geometry priors (e.g., from RGB-D sensors or COLMAP) converted into an occupancy grid, aiming for instant reconstruction on edge devices. It builds on [InstantNGP Network Architecture](5049-instantngp-network-architecture.md) by initializing the occupancy grid with prior geometry and applying density scaling to ensure rays terminate quickly at the surface, addressing the slow convergence of randomly initialized weights. Link: [INGeo](15132-ingeo.md).

### [Scene Scale Consistency Scoring](15618-scene-scale-consistency-scoring.md)
Scene Scale Consistency Scoring (SSCS) is a post-processing heuristic used to detect and remove floating artifacts in neural radiance fields by comparing occupancy grids from models trained with different bounding box scales. It distinguishes itself from sibling methods like Occupancy Clustering by exploiting the geometric invariance of real scene structures against the scale-dependent manifestation of artifacts, thereby filtering out inconsistent density before clustering is applied. Link: [Scene Scale Consistency Scoring](15618-scene-scale-consistency-scoring.md).

### [Unrestricted Color Network (UCN) Distillation](16596-unrestricted-color-network-ucn-distillation.md)
Unrestricted Color Network (UCN) Distillation is a knowledge distillation method that trains a student network to replicate the diffuse color output of a Palette Color Network (PCN) teacher model without enforcing palette constraints, enabling Non-Photorealistic Style Editing (NPSE). Unlike traditional NeRF stylization that discards view-dependent color to maintain consistency, UCN optimizes diffuse color during training and combines it with specular color during inference to preserve lighting effects. Link: [Unrestricted Color Network (UCN) Distillation](16596-unrestricted-color-network-ucn-distillation.md).

#### Flash Cache Implementation
The Flash Cache implementation uses a fast cache NGP with a maximum grid resolution of 256 and a 4-layer 128-hidden-unit MLP to predict $S=8$ sample-point distances [S3]. The hash maps for all NGPs have a size of $2^{19}$ [S3]. This variant is used for predicting distances and features in the cache, differing from the main architecture by its specific MLP depth and hidden units optimized for cache prediction [S3].

#### Upsampling Activations
This extension reduces processing time by computing intermediate network activations at a lower resolution and upsampling them to full resolution before density proposal [S4]. Intermediate activations $A^{(\ell)}$ are recorded at a reduced resolution of $w/2 \times h/2 \times N_c/2 \times N_h$, then upsampled using nearest neighbor interpolation [S4]. Best performance is observed at layer index $\ell = 1$ with function configuration $f_2$ [S4]. On the Blender dataset, this achieves PSNR 23.69, SSIM 0.704, LPIPS 0.267 with a 44% speedup over the baseline [S4]. It differs from full-resolution activation extraction by trading negligible visual quality loss for significant speedup, whereas DONeRF filters along depth and neighboring rays for smoothing [S4]. Applied to NeRF and Mip-NeRF architectures on Blender and LLFF datasets [S4].

#### INGP-NeRF
INGP-NeRF uses a multiresolution hash-grid representation followed by lightweight density and color networks [S5, unofficial]. The density network has 1 hidden layer with 128 units, and the color network has 2 hidden layers with 128 units [S5, unofficial]. Nonlinear activations are replaced with PLIF neurons, and output PLIF layers accumulate temporal estimates of density $\sigma$ and color $\mathbf{c}$ [S5, unofficial]. The maximum time step is set to $T=8$, and the initial target time step $t^*$ is initialized to 7 after pretraining [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than standard NeRF, converging in ~20k iterations compared to 200-400k for Vanilla NeRF [S1, unofficial]. The hash grid lookups are $O(1)$ on average, but memory bandwidth can become a bottleneck due to random access patterns [S1, unofficial].
*   **Memory Complexity:** The hash grid size is fixed (e.g., $2^{19}$ entries) [S3]. Memory usage scales with the number of levels and feature dimension, but is generally lower than dense voxel grids for high-resolution scenes.
*   **Measured Performance:** On Blender and LLFF datasets, upsampling activations variant achieves 44% speedup with minimal quality loss [S4].
*   **Strengths:** Extremely fast training and rendering; hardware-friendly hash lookups; scalable to high resolutions.
*   **Limitations:** Hash collisions can degrade quality; unbounded scenes require modifications; gradient continuity can be poor without hybrid encoding [S1, unofficial].
*   **When to Use:** Real-time training and rendering; large-scale scenes; resource-constrained environments.
*   **When Not to Use:** When strict gradient continuity is required without hybrid encoding; when unbounded domains are needed without negative integer mapping.

## Background
InstantNGP was introduced as a method for fast training and rendering of neural radiance fields using multiresolution hash encoding [S1, unofficial]. It builds on the NeRF MLP Architecture by optimizing the feature extraction step [S1, unofficial]. The method has been widely adopted and extended, including variants like MonoSDF [S2, unofficial], Flash Cache [S3], and INGP-NeRF [S5, unofficial]. It is part of the broader family of neural and primitive-based representations for volume data [S1, unofficial].

## Sources
[S1] NeRF — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-NeRF_SOTA_Updated.html, fetched 2026-10-29 (unofficial)
[S2] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S3] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
[S4] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
[S5] Adaptive Time-step Training for Enhancing Spike-Based Neural Radiance Fields, https://arxiv.org/abs/2507.23033v2, fetched 2026-10-27 (unofficial)
