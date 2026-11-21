# TriPlane NeRF

Also called: Tri-Plane representation, Tri-Planes, TriPlane(x, Triplane Feature Sampling

## Overview
TriPlane NeRF is an explicit-implicit scene representation that models a 3D scene using three axis-aligned orthogonal feature planes [S1][S3, unofficial]. It solves the scaling problems of voxel-based representations by maintaining efficiency and expressiveness while allowing for a small decoder network [S1]. Unlike Voxel NeRF, which requires large memory for high-resolution grids, TriPlane NeRF keeps the decoder small and uses explicit feature representation, enabling planes to be calculated with gradient-based optimization or produced by CNNs like StyleGAN or diffusion models [S1]. It builds on the planar factorization concept of [K-planes](3317-k-planes.md) by restricting the factorization to the three primary coordinate planes ($XY, XZ, YZ$) for static scenes.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D position $x \in \mathbb{R}^3$ and a viewing direction $d$.
*   **Output:** Color $c$ and density $\sigma$.
*   **Parameters:** $\Theta$ (decoder network parameters), $P$ (feature planes).

**Data Structures:**
The representation consists of three feature planes $P = (P_{xy}, P_{xz}, P_{yz})$ [S1][S3, unofficial].
*   Each plane $P_{ij}$ is a 2D feature map with resolution $N \times N$ spatial dimensions and $C$ channels [S1, S2].
*   Typical values: $N = 512$, $C = 8$ [S2].
*   Initialization: Random [S2].
*   Optimization: Optimized during reconstruction via gradient descent [S2].

**Steps:**
1.  **Projection:** Project the 3D position $x = (x, y, z)$ orthogonally onto the three feature planes [S1, S2].
    *   $x$ projects to $(x, y)$ on $P_{xy}$.
    *   $x$ projects to $(x, z)$ on $P_{xz}$.
    *   $x$ projects to $(y, z)$ on $P_{yz}$.
2.  **Sampling:** Identify the four nearest grid points on each plane [S2]. Retrieve the corresponding feature vectors $F_{xy}, F_{xz}, F_{yz}$ via bilinear interpolation from each plane [S1, S2][S3, unofficial].
3.  **Aggregation:** Aggregate the three feature vectors. Sources describe two aggregation methods:
    *   Summation: $F_{agg} = F_{xy} + F_{xz} + F_{yz}$ [S1][S3, unofficial].
    *   Concatenation: $f_{tri} = \text{concat}(f_{XY}, f_{YZ}, f_{ZX})$ [S2].
    *   *Note:* The specific aggregation method depends on the implementation variant (e.g., [S1] uses summation, [S2] uses concatenation).
4.  **Decoding:** Pass the aggregated feature vector into a small neural network (MLP) [S1][S3, unofficial].
    *   The MLP interprets the aggregated 3D features to generate intermediate features [S2].
    *   The intermediate features are combined with the viewing direction $d$ (often encoded via spherical harmonics) [S2].
    *   A Color MLP predicts the final color $c$ [S2].
    *   Density $\sigma$ is also predicted by the decoder [S1][S3, unofficial].
5.  **Rendering:** Use the predicted color $c$ and density $\sigma$ for volume rendering [S1][S3, unofficial].

**Formula:**
The query function is defined as [S1]:
$$ F_{\text{TriPlaneNeRF}}(x, d; \Theta, P) = (c, \sigma) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triplane_query(x, planes, decoder):
    """
    Query TriPlane NeRF for color and density at position x.
    
    Args:
        x: 3D position (x, y, z)
        planes: Dictionary containing P_xy, P_xz, P_yz. 
                Each is a tensor of shape [N, N, C].
        decoder: Function or MLP that takes aggregated features and direction d.
                 Returns (color, density).
    
    Returns:
        color: RGB color
        density: Volume density
    """
    x_val, y_val, z_val = x
    
    # Step 1 & 2: Project and sample via bilinear interpolation
    # Note: Coordinates must be normalized to [0, 1] or [-1, 1] depending on plane definition.
    # Assuming normalized coordinates [0, 1] for grid indexing.
    
    # Sample from XY plane (project to x, y)
    f_xy = bilinear_interpolate(planes['P_xy'], x_val, y_val)
    
    # Sample from XZ plane (project to x, z)
    f_xz = bilinear_interpolate(planes['P_xz'], x_val, z_val)
    
    # Sample from YZ plane (project to y, z)
    f_yz = bilinear_interpolate(planes['P_yz'], y_val, z_val)
    
    # Step 3: Aggregate features
    # Variant A: Summation [S1][S3, unofficial]
    f_agg = f_xy + f_xz + f_yz
    
    # Variant B: Concatenation [S2]
    # f_agg = concatenate([f_xy, f_xz, f_yz])
    
    # Step 4: Decode
    # The decoder typically also takes viewing direction d.
    # This pseudocode assumes d is passed to decoder.
    color, density = decoder(f_agg, d)
    
    return color, density

def bilinear_interpolate(grid, u, v):
    """
    Perform bilinear interpolation on a 2D grid.
    
    Args:
        grid: Tensor [H, W, C]
        u, v: Normalized coordinates in [0, 1]
        
    Returns:
        Feature vector of shape [C]
    """
    # Map normalized coordinates to grid indices
    h, w, c = grid.shape
    u_idx = u * (w - 1)
    v_idx = v * (h - 1)
    
    # Get integer and fractional parts
    u0 = floor(u_idx)
    v0 = floor(v_idx)
    u1 = u0 + 1
    v1 = v0 + 1
    
    # Clamp to boundaries
    u0 = clamp(u0, 0, w - 1)
    v0 = clamp(v0, 0, h - 1)
    u1 = clamp(u1, 0, w - 1)
    v1 = clamp(v1, 0, h - 1)
    
    # Get fractional weights
    du = u_idx - u0
    dv = v_idx - v0
    
    # Retrieve values at four corners
    q00 = grid[v0, u0, :] # Top-left
    q01 = grid[v0, u1, :] # Top-right
    q10 = grid[v1, u0, :] # Bottom-left
    q11 = grid[v1, u1, :] # Bottom-right
    
    # Interpolate
    # Interpolate in u direction
    top = lerp(q00, q01, du)
    bottom = lerp(q10, q11, du)
    
    # Interpolate in v direction
    result = lerp(top, bottom, dv)
    
    return result

def lerp(a, b, t):
    return a + t * (b - a)
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Coordinates outside the $[0, 1]$ (or defined bounding box) range must be clamped or handled via padding to avoid out-of-bounds interpolation errors [G].
*   **Aggregation Choice:** The choice between summation and concatenation affects the decoder's input dimension and capacity. Summation keeps the channel dimension constant ($C$), while concatenation increases it ($3C$) [S1, S2].
*   **Occlusion:** Using only local features (if combined with view projection) can lead to ineffective performance for occlusions and faraway views, while using only global features can lead to hallucinations [S5]. TriPlane NeRF primarily relies on global explicit features, which may struggle with complex occlusions without auxiliary mechanisms [S5].

## Subtopics

### [MultiPlaneGAN](7441-multiplanegan.md)
MultiPlaneGAN is a GAN-based 3D generation framework that synthesizes 3D objects by replacing the tri-plane representation of EG3D with a MultiPlane 3D representation consisting of 32 planes posed on a sphere. It differs from its parent [TriPlane NeRF](6932-triplane-nerf.md) and the baseline EG3D by using a spherical distribution of feature planes rather than axis-aligned orthogonal planes, while retaining the EG3D decoder, super-resolution module, and discriminator architecture.

### [Platonic Solid Projection](8518-platonic-solid-projection.md)
Platonic Solid Projection is a 3D space factorization method that projects 3D areas onto the faces of a Platonic solid, allowing anisotropic 3D areas to be represented on 2D planes with distinguishable characterization. It reduces memory consumption from $O(n^3)$ to $O(n^2)$ by replacing 3D volumes with 2D feature grids. Unlike the orthogonal tri-plane representation (derived from a regular hexahedron/cube), this method uses various Platonic solids to provide a flexible trade-off between rendering quality and efficiency by selecting solids with different numbers of faces.

### [Spatial Planes](9808-spatial-planes.md)
Spatial Planes is a static 3D representation that extends the TriPlane NeRF concept by utilizing three axis-aligned feature planes ($P_{XY}, P_{XZ}, P_{YZ}$) to encode volumetric information. It serves as a baseline variant of HexPlane, designed to evaluate the impact of spatial symmetry in feature plane configurations. Unlike TriPlane NeRF, which typically uses $XY, XZ, YZ$ planes for static scenes, Spatial Planes explicitly isolates these three spatial projections without temporal components, making it incapable of representing dynamic scenes on its own. It is distinguished from siblings like MultiPlaneGAN and Platonic Solid Projection by its specific selection of orthogonal spatial planes rather than arbitrary orientations or generative adversarial structures.

### [Tri-Mip Encoding](11551-tri-mip-encoding.md)
Tri-Mip Encoding is a method to model the pre-filtered 3D feature space by factorizing it into three orthogonal 2D mipmaps, enabling efficient area sampling for anti-aliased rendering. It decomposes 3D space into three axis-aligned planes (XY, XZ, YZ) and represents each with a mipmap structure, allowing queries to adaptively select resolution based on the sampling area size. Unlike [TriPlane NeRF](6932-triplane-nerf.md), which uses static feature planes, Tri-Mip Encoding uses mipmaps to explicitly model pre-filtered features, ensuring coherence across different viewing distances and resolutions.

### [Tri-plane Spatial Encoding with Sigma/Color MLPs](15264-tri-plane-spatial-encoding-with-sigma-color-mlps.md)
This method is the spatial representation component of the KFD-NeRF pipeline, designed to encode canonical 3D points for dynamic neural radiance fields. It solves the problem of balancing high-fidelity 3D reconstruction with computational efficiency and model size, which are critical for dynamic scenes. It sets itself apart from pure MLPs by avoiding convergence difficulties and from multi-scale voxel grids by achieving significantly smaller memory footprints while maintaining comparable or superior rendering quality. It builds on the TriPlane NeRF representation by adding a specific multi-scale feature aggregation strategy and a two-MLP decoding architecture (Sigma and Color).

### HexPlane-Swap
HexPlane-Swap is a variant that groups planes with repeated axes like $P_{XY}, P_{XT}$ [S4]. It achieves inferior results because its axes are not complementary, losing features from the particular axis [S4]. On the D-NeRF dataset, HexPlane-Swap achieves PSNR 28.562, SSIM 0.954, LPIPS 0.056, and Training Time 11m 44s [S4].

### HexPlane-SH
HexPlane-SH is a pure explicit model without MLPs on the D-NeRF dataset [S4]. It stores spherical harmonics (SH) as appearance features and directly regresses RGB from it rather than using MLPs [S4]. On the D-NeRF dataset, HexPlane-SH achieves PSNR 29.284, SSIM 0.952, LPIPS 0.056, and Training Time 10m 42s [S4].

### Deep Residual Local Features
Deep Residual Local Features use local features $f_r$ as a residual connection into the rendering MLP [S5]. These features are obtained by projecting the world point $x$ into source views using camera parameters $\gamma_i, K_i$ and extracting features via bilinear interpolation [S5]. Using only local features leads to ineffective performance for occlusions and faraway 360° views, while using only global features leads to hallucinations; the method combines both to mitigate these issues [S5].

### SDF-based 3D GAN with Spatially-Varying Surface Tightness
This method uses an implicit surface representation based on VolSDF, leveraging a spatially-varying parameter similar to Adaptive Shells to control the softness of the surface [S6]. The network uses a StyleGAN-like architecture to map from a Gaussian prior noise vector $z$ (dimension $R^{512}$) to a triplane representation $T$ [S6]. The triplane $T$ is conditioned by $w = \text{Mapping}(z)$ and has dimensions $T(w) \in R^{3 \times 32 \times 512 \times 512}$, corresponding to three axis-aligned 2D feature grids ($f_{xy}, f_{xz}, f_{yz}$) [S6]. Previous radiance field representations lack a concrete definition for surface geometry, resulting in bumpy surfaces; SDFs resolve this by providing an implicit surface definition [S6]. Unlike StyleSDF which lacks spatial-variance and leads to over-smoothed geometry, this method uses spatially-varying surface tightness to handle both hard boundaries (skin, eyes) and soft volumetric representations (hair) [S6].

### Tri-Plane NeuS
Tri-Plane NeuS is the baseline model using only tri-planes and MLPs to model the SDF, defined as $S(x, y, z) = \text{MLP}([T(x, y, z)])$ [S7].

### Micro-Macro Tri-Planes
Micro-Macro Tri-Planes decompose scene representations into globally shared 'Macro' components and per-scene 'Micro' components to reduce model complexity and memory cost [S3, unofficial]. The method trains $M$ globally shared representations (Macro) that learn common information across scenes, and for each scene, learns only the specific differences (Micro) on top of the shared Macro representation [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is generally faster than voxel-based methods due to the smaller decoder and explicit nature of the planes [S3, unofficial]. The complexity of querying is $O(1)$ per plane, making it efficient for rendering.
*   **Memory Complexity:** Memory usage is $O(N^2 \cdot C)$ for the three planes, which is significantly lower than the $O(N^3)$ of voxel grids [S1][S3, unofficial].
*   **Measured Performance:** TriPlane NeRF (referred to as MultiPlaneNeRF in [S1]) shows competitive performance. For example, on the Chair scene, it achieves PSNR 32.81 and SSIM 0.972, outperforming SRN and NV, and comparable to NeRF and NSFV [S1].
*   **Strengths:** Efficient scaling to higher resolutions compared to voxels; small decoder size; compatibility with GANs and diffusion models [S1].
*   **Limitations:** Static representation (unless extended with temporal planes like HexPlane); potential for hallucinations if global features are used without local context [S5]; bumpy surfaces if not combined with SDFs [S6].
*   **When to use:** When memory is constrained but high resolution is needed; when integrating with generative models (GANs/Diffusion); for static scenes.
*   **When not to use:** For dynamic scenes without temporal extensions; for scenes requiring precise surface geometry without SDF integration.

## Background
TriPlane NeRF was introduced as a solution to the scaling problems of voxel representations in neural radiance fields [S1]. It aligns explicit features along three axis-aligned orthogonal feature planes, enabling efficient querying of 3D points by projecting them onto these planes and aggregating features [S1][S3, unofficial]. The method allows for gradient-based optimization of the planes or their generation by CNNs like StyleGAN or diffusion models [S1]. It is a specific case of planar factorization, related to [K-planes](3317-k-planes.md), but restricted to the three primary coordinate planes for static scene representation [S1][S3, unofficial].

## Sources
[S1] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
[S2] TriDF: Triplane-Accelerated Density Fields for Few-Shot Remote Sensing Novel View Synthesis, https://doi.org/10.1109/tgrs.2025.3643432, fetched 2026-10-30
[S3] Exploring 3D-aware Latent Spaces for Efficiently Learning Numerous Scenes, https://doi.org/10.48550/arXiv.2403.11678, fetched 2026-10-29 (unofficial)
[S4] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
[S5] NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes, https://doi.org/10.1109/iccv51070.2023.00843, fetched 2026-10-26
[S6] What You See is What You GAN: Rendering Every Pixel for High-Fidelity Geometry in 3D GANs, https://doi.org/10.1109/cvpr52733.2024.02148, fetched 2026-10-26
[S7] PET-NeuS: Positional Encoding Tri-Planes for Neural Surfaces, http://arxiv.org/abs/2305.05594v1, fetched 2026-10-27
