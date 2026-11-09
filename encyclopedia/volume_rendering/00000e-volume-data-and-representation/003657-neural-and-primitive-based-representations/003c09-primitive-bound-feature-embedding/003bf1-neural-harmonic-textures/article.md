# Neural Harmonic Textures

Also called: Harmonic Texturing

## Overview
Neural Harmonic Textures (NHT) is a primitive-based neural representation that increases the expressive power of individual primitives by anchoring latent feature vectors to a virtual scaffold surrounding each primitive, applying periodic activations to interpolated features, and decoding the accumulated signal via a single deferred MLP pass [S1]. Unlike prior methods that process interpolated features with a lightweight MLP per primitive followed by alpha compositing (requiring numerous neural evaluations), NHT applies periodic functions before compositing and decodes the accumulated signal once per pixel [S1]. This approach yields a harmonic decomposition of the signal, bridging the gap between primitive- and neural-field-based reconstruction while maintaining real-time performance [S1]. It builds on [Primitive-bound feature embedding](9161-primitive-bound-feature-embedding.md) by defining the specific encoding and decoding mechanics of the feature vectors attached to the scaffold.

## How it works
**Inputs:**
*   A set of primitives $\mathcal{G}$ (e.g., Gaussians, triangles, tetrahedra).
*   For each primitive $i$, a set of learnable feature vectors $\mathbf{f}_{i,v}$ anchored to its virtual scaffold vertices.
*   Primitive opacity $\alpha_i$ and transmittance $T_i$ along the ray.
*   Ray direction $\mathbf{d}$ (or central camera ray for high-parallax scenes).

**Outputs:**
*   Pixel color $\mathbf{c}$ (and optionally other semantic signals).

**Data Structures:**
*   **Primitive Scaffold:** For 3D primitives (e.g., Gaussians), a bounding tetrahedron is defined. Four learnable feature vectors $\mathbf{f}_{i,0}, \mathbf{f}_{i,1}, \mathbf{f}_{i,2}, \mathbf{f}_{i,3}$ are attached to the tetrahedron vertices [S1]. For 2D primitives (2DGS, Triangle Splatting), virtual triangles are used, with three feature vectors per primitive [S1].
*   **Feature Vector Size:** The total number of features per primitive is $F$. For tetrahedral primitives, this is distributed as $F/4$ per vertex [S1]. Typical value: $F=64$ (16 per vertex) [S1].
*   **MLP Decoder:** A shallow Multi-Layer Perceptron. Typical architecture: 128-wide with 3 hidden layers [S1].

**Steps:**

1.  **Ray-Primitive Intersection:** For each pixel, cast a ray and identify the set of intersected primitives $\mathcal{G}$. Determine the point $p^*_i$ on each primitive $i$ where the projected primitive has maximum response (e.g., maximum Gaussian response) [S1].

2.  **Local Interpolation:** Barycentrically interpolate the vertex feature vectors at $p^*_i$ to obtain the interpolated feature vector $\mathbf{f}_i$ for primitive $i$ [S1].
    $$ \mathbf{f}_i = \text{Interpolate}(\{\mathbf{f}_{i,v}\}, p^*_i) $$

3.  **Harmonic Encoding:** Encode the interpolated feature vector $\mathbf{f}_i$ using sine and cosine functions into separate channels. This creates a harmonic representation where the interpolation acts as a frequency modulator [S1].
    $$ \mathbf{h}_i = \begin{bmatrix} \sin(\mathbf{f}_i) \\ \cos(\mathbf{f}_i) \end{bmatrix} $$
    Note: The sine and cosine are applied element-wise to $\mathbf{f}_i$ [S1].

4.  **Accumulation (Alpha Blending):** Composite the encoded features $\mathbf{h}_i$ along the ray, weighted by the primitive's kernel-weighted opacity $\alpha_i$ and accumulated transmittance $T_i$ [S1]. The primitive opacity modulates the amplitude of each harmonic component [S1].
    $$ \mathbf{H} = \sum_{i \in \mathcal{G}} \alpha_i T_i \mathbf{h}_i = \sum_{i \in \mathcal{G}} \alpha_i T_i \begin{bmatrix} \sin(\mathbf{f}_i) \\ \cos(\mathbf{f}_i) \end{bmatrix} $$
    where $T_i = \prod_{j < i} (1 - \alpha_j)$ [S1].

5.  **Directional Encoding (Optional):** Encode the ray direction $\mathbf{d}$ using second-degree spherical harmonics $\text{SH}_2(\mathbf{d}) \in \mathbb{R}^9$ [S1]. For indoor/high-parallax datasets, encode the central camera ray instead of the individual ray direction to regularize training [S1].

6.  **Deferred Decoding:** Decode the accumulated implicit signal $\mathbf{H}$ and the directional encoding in a single pass per pixel using the lightweight MLP $\text{MLP}_\theta$ [S1].
    $$ \mathbf{c} = \text{MLP}_\theta \left( \mathbf{H}, k \cdot \text{SH}_2(\mathbf{d}) \right) $$
    where $k$ is a scaling factor (not explicitly defined in sources, but implied by concatenation or weighting) [S1]. The MLP does not use further positional encoding [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def neural_harmonic_textures(primitives, ray_origin, ray_direction, mlp_weights):
    """
    Renders a single ray using Neural Harmonic Textures.
    
    Args:
        primitives: List of primitive objects. Each primitive has:
            - vertices: List of 4 (or 3 for 2D) feature vectors (numpy arrays).
            - opacity: Scalar alpha.
            - get_max_response_point(ray_origin, ray_direction): Returns point p* and barycentric coords.
        ray_origin: 3D point.
        ray_direction: Normalized 3D vector.
        mlp_weights: Weights for the 128x3 MLP decoder.
        
    Returns:
        color: 3D vector (RGB).
    """
    # Sort primitives by distance if necessary, though alpha blending order matters
    # Assuming primitives are sorted by depth or handled by the specific primitive type's rasterization
    # For standard 3DGS-like splatting, we iterate through sorted primitives
    
    accumulated_harmonic = np.zeros(2 * 64) # 64 features -> 128 channels (sin + cos)
    transmittance = 1.0
    
    # Determine directional encoding
    # Use central camera ray for high-parallax, else ray direction
    # For simplicity, assume ray_direction is used unless specified otherwise
    # SH2 encoding of direction
    sh2_d = spherical_harmonics_degree_2(ray_direction) # Returns 9D vector
    
    for prim in primitives:
        # 1. Find intersection/max response point
        p_star, barycentric_coords = prim.get_max_response_point(ray_origin, ray_direction)
        
        # If no intersection or max response is negligible, skip
        if p_star is None:
            continue
            
        # 2. Interpolate features
        # Barycentric interpolation of vertex features
        # prim.vertices is a list of feature vectors (size F)
        f_i = np.zeros_like(prim.vertices[0])
        for v_idx, coord in enumerate(barycentric_coords):
            f_i += coord * prim.vertices[v_idx]
            
        # 3. Harmonic Encoding
        # Encode sin and cos element-wise
        h_i_sin = np.sin(f_i)
        h_i_cos = np.cos(f_i)
        h_i = np.concatenate([h_i_sin, h_i_cos])
        
        # 4. Alpha Blending
        alpha = prim.opacity
        # Modulate harmonic amplitude by opacity
        weighted_h = alpha * transmittance * h_i
        
        accumulated_harmonic += weighted_h
        transmittance *= (1.0 - alpha)
        
        # Early exit if fully opaque
        if transmittance < 1e-4:
            break
            
    # 5. Deferred Decoding
    # Concatenate accumulated harmonic and directional encoding
    # Note: The formula shows concatenation. We assume k=1 or scaling is handled in MLP
    input_to_mlp = np.concatenate([accumulated_harmonic, sh2_d])
    
    # Decode using MLP
    color = mlp_forward(input_to_mlp, mlp_weights)
    
    return color

def spherical_harmonics_degree_2(direction):
    """
    Computes 2nd degree spherical harmonics for a direction vector.
    Returns a 9D vector.
    """
    x, y, z = direction
    # Standard SH2 basis functions (unnormalized for brevity, normalization constants can be absorbed in MLP)
    # L0: 1
    # L1: x, y, z
    # L2: xy, xz, yz, x^2-y^2, 3z^2-1 (or similar variants)
    # Using standard real SH2 basis:
    sh = np.zeros(9)
    sh[0] = 1.0 # L0
    sh[1] = x   # L1
    sh[2] = y   # L1
    sh[3] = z   # L1
    sh[4] = x * y # L2
    sh[5] = x * z # L2
    sh[6] = y * z # L2
    sh[7] = x**2 - y**2 # L2
    sh[8] = 3 * z**2 - 1 # L2
    return sh

def mlp_forward(x, weights):
    """
    Simple 3-hidden layer MLP (128 wide).
    weights: List of [W1, b1, W2, b2, W3, b3, W4, b4]
    """
    h = x
    for i in range(3):
        W, b = weights[2*i], weights[2*i+1]
        h = np.maximum(0, h @ W.T + b) # ReLU activation
    # Output layer
    W_out, b_out = weights[6], weights[7]
    out = h @ W_out.T + b_out
    return out
```

**Parameters:**
*   `features_per_primitive`: 64 (16 per vertex for tetrahedral primitives) [S1].
*   `MLP architecture`: 128-wide × 3 hidden layers [S1].
*   `training_iterations`: 30,000 (default for MipNeRF360 benchmark) [S1].
*   `primitive_cap`: 1,000,000 (1M primitives) [S1].
*   `indoor_iterations`: 45,000 (for indoor datasets with ~290 images) [S1].
*   `outdoor_iterations`: 25,000 (for outdoor datasets with ~175 images) [S1].

**Edge Cases and Pitfalls:**
*   **Convergence Speed:** MLP-based appearance models converge slower than SH-based ones [S1].
*   **High-Parallax Regularization:** For indoor/high-parallax datasets, encode the central camera ray instead of the individual ray direction to regularize training [S1].
*   **Semantic Supervision Resolution:** For semantic field reconstruction, the limited resolution of feature maps (e.g., LSEG) requires rendering and predicting at RGB resolution, then bilinearly downsampling the spatial dimension of the resulting feature maps to supervise against ground truth features [S1].
*   **2D Primitives:** For 2D primitives (2DGS, Triangle Splatting), replace bounding tetrahedra with virtual triangles and anchor feature vectors to explicit triangle geometry, using three feature vectors per primitive instead of four [S1].

## Subtopics

### [Neural Harmonic Textures (2D Image Reconstruction Adaptation)](9148-neural-harmonic-textures-2d-image-reconstruction-adaptation.md)
An adaptation of Neural Harmonic Textures for 2D image fitting that replaces 3D primitives with a connected 2D triangle mesh and modifies the interpolation scheme to eliminate high-frequency artifacts.

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Uses less storage than Spherical Harmonics (SH) and Spherical Voronoi (SV) [S1]. For semantic reconstruction, it uses ~3× lower memory footprint than Feature 3DGS (80 features + 128×3 MLP vs 176 features + 128×512 CNN) [S1].
*   **Compute:** Significantly reduces computational cost compared to per-primitive MLP decoding by decoding the accumulated signal once per pixel [S1].

**Measured Performance:**
*   **Training Time:** ~14.5 minutes on MipNeRF360 (1M primitives, RTX 5090) up to 30k iterations [S1].
*   **Rendering Speed:** 140 FPS on RTX A6000 Ada for MipNeRF360 [S1].
*   **Quality (MipNeRF360):** PSNR 28.74, SSIM 0.834, LPIPS 0.216 [S1].
*   **Quality (MipNeRF360, RTX A6000 Ada):** PSNR 28.46, SSIM 0.830, LPIPS 0.232 [S1].
*   **Scaling:** Improves over prior art across all primitive counts (1K to 4M), excelling particularly in the low primitive count regime (≤ 100k) [S1]. Achieves performance comparable to previous methods using 1M primitives with only a third of that amount [S1].

**Strengths and Limitations:**
*   **Strengths:** State-of-the-art performance in real-time novel view synthesis; bridges gap between primitive- and neural-field-based reconstruction; agnostic to underlying primitive type (triangles, 2D/3D Gaussians, tetrahedra); supports motion, deformation, and scene editing naturally; extends to higher-dimensional signals (semantic reconstruction) [S1].
*   **Limitations:** MLP-based appearance models converge slower than SH-based ones [S1]. Limited resolution of semantic feature maps requires specific downsampling strategies for supervision [S1].

**When to Use:**
*   When high-quality novel view synthesis is required with real-time rendering constraints.
*   When integrating with existing primitive-based pipelines (3DGUT, Triangle Splatting, 2DGS) to boost expressivity.
*   When semantic field reconstruction is needed with lower memory footprint than CNN-based approaches.

**When Not to Use:**
*   When extremely fast convergence is prioritized over final quality (SH-based models converge faster).
*   When scene editing or motion support is not required and global grids are sufficient (though NHT still offers quality benefits).

## Background
Neural Harmonic Textures were introduced by Nicolas M. et al. from NVIDIA and Università della Svizzera italiana in 2024 [S1]. The method was presented in the paper "Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction" [S1]. It addresses the limited expressivity of individual primitives in Lagrangian primitive-based approaches like 3D Gaussian Splatting, which struggle with high-frequency detail compared to Eulerian continuous neural radiance fields [S1]. By reinterpreting primitives as both geometric carriers and local positional encodings, NHT anchors latent feature vectors on a virtual scaffold, moving with the primitives to support operations such as motion, deformation, and scene editing [S1]. The approach integrates seamlessly into existing primitive-based pipelines and has been validated on standard novel view synthesis benchmarks (MipNeRF360, Tanks & Temples, Deep Blending) as well as 2D image fitting and semantic reconstruction tasks [S1].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
