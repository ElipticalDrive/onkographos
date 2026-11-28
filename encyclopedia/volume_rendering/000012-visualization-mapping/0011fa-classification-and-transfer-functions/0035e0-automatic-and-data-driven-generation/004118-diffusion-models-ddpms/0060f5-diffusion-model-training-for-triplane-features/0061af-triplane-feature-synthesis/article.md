# Triplane Feature Synthesis

Also called: EG3D Triplane Synthesis, EG3D-style backbone, StyleGAN Triplane Generation, Triplanar NeRF Generator (EG3D-style backbone), triplanar decoder

## Overview
Triplane Feature Synthesis is a method for generating expressive 3D features by representing a scene as three orthogonal 2D feature planes (xy, xz, yz) conditioned on a latent code [S1, S2]. It differs from standard triplanar decoders by employing extra synthesis blocks for disentanglement, allowing plane-specific kernels to attend to features in each separate plane [S1]. This approach builds on [Diffusion Model Training for Triplane Features](18292-diffusion-model-training-for-triplane-features.md) by providing the generative backbone that produces the initial 3D-aware feature representation.

## How it works
**Inputs:**
*   A prior noise vector $z \sim \mathcal{N}(0, I)$ [S2].
*   A 3D point coordinate $x \in \mathbb{R}^3$ for feature query [S1, S2].

**Outputs:**
*   A synthesized triplane feature tensor $T$ [S1, S2].
*   A final feature vector for point $x$, derived from $T$ [S1, S2].

**Data Structures:**
*   **Latent Code $w$**: Intermediate latent code residing in the W space, output of the mapping network [S2].
*   **Initial Triplane $T'$**: Output of the StyleGAN2 generator layers, before disentanglement blocks [S1].
*   **Feature Planes**: Three orthogonal 2D planes corresponding to xy, xz, and yz coordinate projections [S2]. Each plane is queried at 3D coordinates via bilinear sampling [S2].
*   **MLP**: A "tiny MLP" (triplanar decoder) that takes summed features as input to output final representation values (e.g., radiance field density and color, or SDF/geometry features) [S1, S2].

**Steps:**
1.  **Latent Mapping**: Sample a prior vector $z$ from a standard normal distribution $\mathcal{N}(0, I)$ [S2]. Pass $z$ through a mapping network to produce the latent code $w$ in the W space [S2].
2.  **Initial Synthesis**: Feed $w$ into a StyleGAN2 generator to produce an output feature map [S2]. This output is the initial triplane $T'$ [S1].
3.  **Disentanglement (Optional but Recommended)**: For each orthogonal plane $ij \in \{xy, xz, yz\}$, apply a `SynthesisBlock` to the features $f'_{ij}$ of $T'$ to generate expressive features $f_{ij}$ [S1]:
    $$f_{ij} = \text{SynthesisBlock}(f'_{ij})$$
    This design allows disentanglement between intermediate triplane features, as plane-specific kernels can attend to features in each separate plane [S1].
4.  **Feature Query**: To get features $T_x$ for a point $x$:
    *   Project $x$ to each of the three axis-aligned planes [S1, S2].
    *   Query each plane at the specific 2D coordinates using bilinear sampling [S2].
    *   Take the Hadamard product of the three resultant vectors [S1].
    *   *Note*: Some implementations sum the sampled features from the three planes element-wise before feeding them into the decoder [S2]. The sources describe both Hadamard product [S1] and summation [S2]. The Hadamard product is explicitly linked to the EG3D-style backbone in [S1], while [S2] describes the summation in the context of the unconditional generator pre-training.
5.  **Decoding**: Pass the combined features (summed or Hadamard product) into the triplanar decoder MLP to output the final representation [S1, S2].

**Parameters:**
*   **MLPSDF Initialization**: If using an SDF representation, the ReLU MLPSDF is initialized as in SAL [4] to ensure an approximately spherical SDF in the early part of training [S1].
*   **z**: Prior noise vector, sampled from $\mathcal{N}(0, I)$ [S2].
*   **w**: Intermediate latent code in W space [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def triplane_synthesis(z, generator, decoder, use_hadamard=True):
    """
    Synthesize triplane features and query them for a point.
    
    Args:
        z: Noise vector from N(0, I).
        generator: StyleGAN2 generator with mapping network.
        decoder: Triplanar decoder MLP.
        use_hadamard: If True, use Hadamard product for feature combination [S1].
                      If False, use summation [S2].
    Returns:
        final_features: Output of the decoder.
    """
    # Step 1: Latent Mapping
    w = generator.mapping_network(z)
    
    # Step 2: Initial Synthesis
    # Output is a feature map split into three orthogonal planes
    T_prime = generator.stylegan2_layers(w)
    
    # Step 3: Disentanglement (Extra Synthesis Blocks)
    # T_prime is split into f_xy, f_xz, f_yz
    f_xy_prime, f_xz_prime, f_yz_prime = split_planes(T_prime)
    
    # Apply SynthesisBlock for each plane
    f_xy = synthesis_block(f_xy_prime)
    f_xz = synthesis_block(f_xz_prime)
    f_yz = synthesis_block(f_yz_prime)
    
    # Step 4: Feature Query (Example for a single point x)
    # Note: In practice, this is done for all points in a volume or along rays
    x = np.array([0.0, 0.0, 0.0]) # Example point
    
    # Project x to 2D coordinates for each plane
    # xy plane: (x, y), xz plane: (x, z), yz plane: (y, z)
    coord_xy = (x[0], x[1])
    coord_xz = (x[0], x[2])
    coord_yz = (x[1], x[2])
    
    # Bilinear sampling
    feat_xy = bilinear_sample(f_xy, coord_xy)
    feat_xz = bilinear_sample(f_xz, coord_xz)
    feat_yz = bilinear_sample(f_yz, coord_yz)
    
    # Combine features
    if use_hadamard:
        # Hadamard product of the three resultant vectors [S1]
        combined_features = feat_xy * feat_xz * feat_yz
    else:
        # Sum the sampled features element-wise [S2]
        combined_features = feat_xy + feat_xz + feat_yz
        
    # Step 5: Decoding
    final_features = decoder(combined_features)
    
    return final_features

def split_planes(T_prime):
    """Split the output feature map into three orthogonal planes."""
    # Implementation depends on the specific tensor layout of T_prime
    # Assuming T_prime is a tensor of shape (3, C, H, W)
    return T_prime[0], T_prime[1], T_prime[2]

def synthesis_block(f):
    """Apply extra synthesis block for disentanglement."""
    # This block allows plane-specific kernels to attend to features
    # in each separate plane [S1].
    # Implementation details are specific to the StyleGAN2 architecture.
    return f # Placeholder for actual block logic

def bilinear_sample(plane, coord):
    """Bilinear sampling of a 2D plane at a specific coordinate."""
    # Implementation of bilinear interpolation
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision**: Bilinear sampling requires careful handling of coordinate boundaries to avoid out-of-bounds access [S2].
*   **Degenerate Input**: The method assumes a valid 3D point $x$ [S1, S2].
*   **Feature Combination**: Sources disagree on whether to use Hadamard product [S1] or summation [S2] for combining features from the three planes. The Hadamard product is associated with the EG3D-style backbone [S1], while summation is described in the unconditional generator pre-training [S2]. Implementers should choose based on the specific variant being implemented.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Not given in the sources.
*   **Strengths**: Allows disentanglement between intermediate triplane features, enabling plane-specific kernels to attend to features in each separate plane [S1]. It is a 3D-aware unconditional generator [S2].
*   **Limitations**: Not given in the sources.
*   **When to use**: When a 3D-aware generative model is needed that can be conditioned on a latent code [S1, S2].
*   **When not**: When view-independent features are sufficient and disentanglement is not required [S1].

## Background
*   **Origin**: The method is based on EG3D [5] and StyleGAN2 [29] [S2]. It was introduced in the context of high-fidelity geometry in 3D GANs [S1] and single-image 3D reconstruction [S2].
*   **History**: The approach builds on previous work in 3D GANs and radiance field inversion [S1, S2].
*   **Usage**: Used in 3D GANs for rendering every pixel for high-fidelity geometry [S1] and for shape, pose, and appearance estimation from a single image [S2].

## Sources
[S1] What You See is What You GAN: Rendering Every Pixel for High-Fidelity Geometry in 3D GANs, https://doi.org/10.1109/cvpr52733.2024.02148, fetched 2026-10-26
[S2] Shape, Pose, and Appearance from a Single Image via Bootstrapped Radiance Field Inversion, https://doi.org/10.1109/cvpr52729.2023.00427, fetched 2026-10-26
