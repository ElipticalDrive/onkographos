# K-planes

Also called: hex-planes, k-planes factorization, tri-planes

## Overview
K-planes is a white-box, interpretable neural representation for radiance fields in arbitrary dimensions $d$ that uses planar factorization to represent a $d$-dimensional scene [S1]. It solves the scalability and memory limitations of implicit MLP-based fields and dense voxel grids by decomposing the scene into $k = \binom{d}{2}$ explicit 2D feature planes [S1]. Unlike sibling methods that add features from planes (e.g., [TriPlane NeRF](6932-triplane-nerf.md), [TensoRF](6932-triplane-nerf.md)), K-planes multiplies them element-wise, which enables spatially localized signals and allows the use of a linear feature decoder with a learned color basis instead of a black-box MLP [S1]. This approach achieves high compression (e.g., 1000x over a full 4D grid) and fast optimization while maintaining competitive reconstruction fidelity [S1].

## How it works
**Inputs:** A set of training images with known camera poses and, for dynamic scenes, timestamps. For unbounded scenes, a scene contraction function; for forward-facing scenes, normalized device coordinates (NDC).
**Outputs:** Density $\sigma(\mathbf{q})$ and color $\mathbf{c}(\mathbf{q}, \mathbf{d})$ for any query point $\mathbf{q}$ and viewing direction $\mathbf{d}$.

**Data Structures:**
The scene is represented by $k = \binom{d}{2}$ 2D planes. Each plane $\mathbf{P}_c$ has shape $N \times N \times M$, where $N$ is the symmetric spatial/temporal resolution and $M$ is the feature dimension.
- For static 3D scenes ($d=3$), $k=3$ planes: $\mathbf{P}_{xy}, \mathbf{P}_{xz}, \mathbf{P}_{yz}$ (tri-planes).
- For dynamic 4D scenes ($d=4$), $k=6$ planes: $\mathbf{P}_{xy}, \mathbf{P}_{xz}, \mathbf{P}_{yz}$ (space-only) and $\mathbf{P}_{xt}, \mathbf{P}_{yt}, \mathbf{P}_{zt}$ (space-time) (hex-planes).
- Multiscale representation: Multiple copies of these planes exist at different resolutions (e.g., 64, 128, 256, 512).

**Algorithm Steps:**
1. **Coordinate Normalization:** Normalize the query point $\mathbf{q}$ entries to the range $[0, N)$. For forward-facing scenes, apply NDC. For unbounded scenes, apply $\ell_\infty$ scene contraction.
2. **Projection:** For each plane $c \in \{1, \dots, k\}$, project $\mathbf{q}$ onto the 2D plane corresponding to the dimensions defining $c$. Let $\pi_c(\mathbf{q})$ be the 2D coordinates.
3. **Multiscale Interpolation:** For each scale $s$, retrieve the feature vector $\mathbf{f}(\mathbf{q})_{c,s}$ from plane $\mathbf{P}_{c,s}$ at coordinates $\pi_c(\mathbf{q})$ using bilinear interpolation $\psi$:
   $$ \mathbf{f}(\mathbf{q})_{c,s} = \psi(\mathbf{P}_{c,s}, \pi_c(\mathbf{q})) $$
4. **Feature Combination:** Combine features from all $k$ planes for scale $s$ using the Hadamard product (element-wise multiplication):
   $$ \mathbf{f}(\mathbf{q})_s = \prod_{c \in C} \mathbf{f}(\mathbf{q})_{c,s} $$
   where $C$ is the set of plane indices. This produces a feature vector of length $M$.
5. **Concatenation:** Concatenate the feature vectors from all $S$ scales along the feature dimension to form a single input vector $\mathbf{f}_{concat}(\mathbf{q})$ of length $S \times M$.
6. **Decoding:** Decode $\mathbf{f}_{concat}(\mathbf{q})$ into density and color using one of the following decoders:
   - **Linear Decoder (Explicit):**
     - Density: $\sigma(\mathbf{q}) = \text{exp}(\mathbf{f}_{concat}(\mathbf{q}) \cdot \mathbf{b}_\sigma)$, where $\mathbf{b}_\sigma$ is a learned basis vector.
     - Color: $\mathbf{c}(\mathbf{q}, \mathbf{d}) = \text{sigmoid}(\mathbf{f}_{concat}(\mathbf{q}) \cdot \mathbf{B}_{RGB}(\mathbf{d}))$, where $\mathbf{B}_{RGB}(\mathbf{d})$ is a view-dependent basis matrix.
   - **MLP Decoder (Hybrid):**
     - Use two small MLPs: $g_\sigma$ maps $\mathbf{f}_{concat}(\mathbf{q})$ to density $\sigma$ and intermediate features $\hat{\mathbf{f}}$; $g_{RGB}$ maps $\hat{\mathbf{f}}$ and embedded view direction $\gamma(\mathbf{d})$ to color $\mathbf{c}$.
     - $\sigma(\mathbf{q}) = \text{exp}(g_\sigma(\mathbf{f}_{concat}(\mathbf{q})))$
     - $\mathbf{c}(\mathbf{q}, \mathbf{d}) = \text{sigmoid}(g_{RGB}(\hat{\mathbf{f}}, \gamma(\mathbf{d})))$
7. **Rendering:** Render the ray using standard volumetric rendering formula with the computed $\sigma$ and $\mathbf{c}$.

**Parameters:**
- $d$: Dimensionality of scene space. Typical values: $d=3$ (static), $d=4$ (dynamic) [S1].
- $k$: Number of planes, $\binom{d}{2}$. Typical values: $k=3$ (3D), $k=6$ (4D) [S1].
- $N$: Resolution of planes. Typical values: 64, 128, 256, 512 [S1].
- $M$: Feature size per plane. Not given in the sources.
- $S$: Number of scales. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Addition vs. Multiplication:** Using addition instead of multiplication prevents the linear decoder from achieving competitive quality because addition produces intersecting lines rather than spatially localized signals [S1].
- **Initialization:** Space-time planes ($\mathbf{P}_{xt}, \mathbf{P}_{yt}, \mathbf{P}_{zt}$) should be initialized to 1 (multiplicative identity) to encourage static regions to remain unchanged [S1].
- **Sampling:** Proposal sampling requires a small instance of k-planes as a density model, trained with histogram loss [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def k_planes_forward(q, planes, scales, decoder_type, view_dir=None):
    """
    q: tuple of coordinates (x, y, z[, t]) normalized to [0, N)
    planes: dict mapping plane_name (e.g., 'xy') to list of tensors [P_s1, P_s2, ...]
    scales: list of scale indices
    decoder_type: 'linear' or 'mlp'
    view_dir: viewing direction vector d (required for linear decoder or MLP)
    """
    # 1. Project q onto each plane and interpolate
    all_features = []
    for s in scales:
        scale_features = []
        for plane_name, plane_tensors in planes.items():
            P = plane_tensors[s]
            # Determine 2D coordinates based on plane_name
            if plane_name == 'xy':
                coords = (q[0], q[1])
            elif plane_name == 'xz':
                coords = (q[0], q[2])
            elif plane_name == 'yz':
                coords = (q[1], q[2])
            elif plane_name == 'xt':
                coords = (q[0], q[3])
            elif plane_name == 'yt':
                coords = (q[1], q[3])
            elif plane_name == 'zt':
                coords = (q[2], q[3])
            
            # Bilinear interpolation
            f_c = bilinear_interpolate(P, coords)
            scale_features.append(f_c)
        
        # 2. Combine features using Hadamard product
        # Initialize with ones of shape (M,)
        f_combined = torch.ones_like(scale_features[0])
        for f_c in scale_features:
            f_combined = f_combined * f_c
        
        all_features.append(f_combined)
    
    # 3. Concatenate features from all scales
    f_concat = torch.cat(all_features, dim=-1)
    
    # 4. Decode
    if decoder_type == 'linear':
        # Linear decoder with learned color basis
        # b_sigma: (M_total,)
        # B_RGB: (M_total, 3, M_view) or similar structure depending on implementation
        # For simplicity, assuming direct matrix multiplication for color
        sigma = torch.exp(torch.dot(f_concat, b_sigma))
        # Color basis depends on view_dir
        c = torch.sigmoid(torch.matmul(f_concat, view_basis(view_dir)))
    elif decoder_type == 'mlp':
        # MLP decoder
        sigma, f_hat = mlp_sigma(f_concat)
        sigma = torch.exp(sigma)
        c = torch.sigmoid(mlp_rgb(f_hat, view_dir))
        
    return sigma, c
```

## Subtopics

### K-planes MLP Decoder
A hybrid model where k-planes features are decoded by small MLPs, similar to Instant-NGP and DVGO [S1]. Features are decoded by two small MLPs: $g_\sigma$ maps features $\mathbf{f}(\mathbf{q})$ into density $\sigma$ and additional features $\hat{\mathbf{f}}$; $g_{RGB}$ maps $\hat{\mathbf{f}}$ and embedded view direction $\gamma(\mathbf{d})$ into RGB color [S1]. Predicted density and color values are normalized via exponential and sigmoid, respectively [S1]. This variant is used when the linear decoder's expressiveness is insufficient, though it reintroduces some black-box characteristics [S1].

### Appearance Code Augmentation
An extension to handle scenes with varying appearance (e.g., changing illumination) by adding a global appearance code per frame to disentangle global appearance from scene geometry [S1]. The k-planes model (either linear or MLP decoder) is augmented with a learned M-dimensional vector for each training image [S1]. This vector is passed as an additional input to the color decoder (either the learned color basis $\mathbf{B}_{RGB}$ or the MLP color decoder $g_{RGB}$), affecting color but not geometry [S1]. This allows reconstruction of scenes like phototourism collections with varying lighting [S1].

## Operating characteristics
- **Time Complexity:** Training is orders of magnitude faster than prior implicit models and on par with concurrent hybrid models [S1].
- **Memory Complexity:** Achieves 1000x compression over a full 4D grid. A 4D volume represented by K-planes uses 200 MB, whereas a direct representation at the same resolution requires >300 GB [S1].
- **Performance:** Yields competitive and often state-of-the-art reconstruction fidelity across synthetic and real, static and dynamic, fixed and varying appearance scenes [S1].
- **Strengths:** White-box, interpretable, scalable to arbitrary dimensions, low memory usage, fast optimization [S1].
- **Limitations:** Requires careful initialization of space-time planes to 1 for static regions [S1]. Importance sampling (IST) has only a modest impact on full-frame metrics but improves visual quality in dynamic regions [S1].

## Background
K-planes was introduced in 2023 by [S1]. It is inspired by explicit modeling of Plenoxels, triplane models (EG3D), tensor decomposition (TensoRF), and multiscale grid models (Instant-NGP) [S1]. It addresses the limitations of black-box implicit models (long training times, lack of interpretability) and redundant decompositions (e.g., Tensor4D uses 9 planes for 4D, while K-planes uses 6) [S1]. It is used for static 3D volumes, 3D photo collections with varying appearances, and 4D dynamic videos [S1].

## Sources
[S1] K-Planes: Explicit Radiance Fields in Space, Time, and Appearance, https://doi.org/10.1109/cvpr52729.2023.01201, fetched 2026-10-26
