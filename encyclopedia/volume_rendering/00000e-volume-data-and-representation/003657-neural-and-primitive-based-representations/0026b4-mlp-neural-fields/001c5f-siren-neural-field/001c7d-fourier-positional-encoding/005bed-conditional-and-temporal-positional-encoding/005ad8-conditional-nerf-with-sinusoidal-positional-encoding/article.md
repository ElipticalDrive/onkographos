# Conditional NeRF with Sinusoidal Positional Encoding

Also called: Conditional NeRF

## Overview
Conditional NeRF is a neural radiance field formulation that maps spatial position, view direction, and latent codes (such as shape or appearance codes derived from text prompts) to volumetric density and view-dependent color [S1]. It solves the problem of generating or manipulating 3D scenes conditioned on external inputs, such as text descriptions, by using a feed-forward approach to update condition codes rather than optimizing them per sample [S1]. It is distinguished from standard NeRF by its explicit conditioning on latent vectors and its use of sinusoidal positional encoding for inputs, often coupled with a deformation network to handle global volume field manipulations [S1].

## How it works
The method defines a conditional neural radiance field $F_\theta$ that takes position $x$, view direction $v$, and latent codes as input [S1]. The core mechanism involves projecting inputs into a high-dimensional space using sinusoidal positional encoding and applying deformations constrained by a hyperbolic tangent function to ensure training stability [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Position $x = (x, y, z) \in \mathbb{R}^3$ [S1].
    *   View direction $v = (\phi, \theta) \in \mathbb{R}^2$ (spherical coordinates) [S1].
    *   Shape code $z_s$ and appearance code $z_a$ (latent vectors) [S1].
    *   Text prompt $t$ (used to generate $z_s$ and $z_a$ via mappers, though the core NeRF function takes the codes directly) [S1].
*   **Outputs:**
    *   Volumetric density $\sigma$ [S1].
    *   View-dependent radiance $c = (r, g, b)$ [S1].

### Data Structures and Layout
The model is parameterized by a Multi-Layer Perceptron (MLP) [S1].
*   **Positional Encoding:** The function $\Gamma(p)$ projects each coordinate $p$ of a vector $p$ to a high-dimensional space [S1]. For a vector $p$, $\Gamma(p) = \gamma(p) \mid p \in p$ [S1].
*   **Deformed Positional Encoding:** A deformation network outputs a displacement $\Delta p$ [S1]. The deformed encoding $\gamma^*$ combines the original encoding with this displacement [S1].
*   **Latent Code Integration:** The latent codes $z_s$ and $z_a$ are concatenated with the encoded inputs [S1]. In specific architectures like the Face Prior Model, the latent code is concatenated at each layer of the MLP [S2].

### Numbered Steps and Formulas

1.  **Sinusoidal Positional Encoding:**
    For each dimension $k$ of the encoding output, the function $\gamma(\cdot): \mathbb{R} \to \mathbb{R}^{2m}$ is defined as [S1]:
    $$
    \gamma(p)_k =
    \begin{cases}
    \sin(2^k \pi p), & \text{if } k \text{ is even} \\
    \cos(2^k \pi p), & \text{if } k \text{ is odd}
    \end{cases}
    $$
    Here, $p$ is a scalar coordinate, and $k$ indexes the output dimension [S1].

2.  **Deformation (Displacement):**
    To manipulate the volume field (e.g., for text-driven shape changes), a deformation network predicts a displacement vector $\Delta p \in \mathbb{R}^{2m}$ [S1]. The deformed positional encoding $\gamma^*$ is calculated as [S1]:
    $$
    \gamma^*(p, \Delta p)_k = \gamma(p)_k + \tanh(\Delta p_k)
    $$
    The scalar $p$ and vector $\Delta p$ belong to the same axis [S1].

3.  **Constraining Displacements:**
    The hyperbolic tangent function $\tanh(\cdot)$ is applied to the displacement $\Delta p$ to constrain it to the range $[-1, 1]$ [S1]. This constraint helps avoid poor local minima caused by large motions and increases training robustness [S1].

4.  **NeRF Forward Pass:**
    The conditional NeRF $F_\theta$ computes the output by concatenating the encoded inputs and latent codes [S1]:
    $$
    F_\theta(x, v, z_s, z_a) : \Gamma(x) \oplus z_s, \Gamma(v) \oplus z_a \to (c, \sigma)
    $$
    where $\oplus$ is the concatenation operator [S1].
    *   Note: In the deformation variant, $\Gamma(x)$ is replaced by $\Gamma^*(x, \Delta p)$ [S1].
    *   Note: In the Face Prior Model variant, integrated positional encodings $\tilde{\gamma}_x$ are used for the point input, while standard positional encoding $\gamma_v$ is used for the view direction without integration [S2].

### Parameters and Typical Values
*   **Tanh Constraint:** The range of $\tanh(\Delta p)$ is $[-1, 1]$ [S1].
*   **Latent Codes:** $z_s$ and $z_a$ are optimized or predicted from text embeddings using mappers $M_s$ and $M_a$ [S1].
*   **Architecture:** The base model uses an MLP [S1]. The Face Prior variant uses two MLPs: a proposal network (predicts density only) and a NeRF MLP (predicts density and color), based on MipNeRF360 [S2].

### Edge Cases and Pitfalls
*   **Large Motions:** Without the $\tanh$ constraint on $\Delta p$, large displacements can cause poor local minima during optimization [S1].
*   **View-Dependent Effects:** Unlike some generative models that condition only on latent codes, conditioning on view direction $d$ is necessary to enable view-dependent effects (e.g., specular highlights) [S2].
*   **Overfitting:** When fitting to very few images (few-shot), regularizing view direction weights is required to prevent overfitting [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sinusoidal_encoding(p, num_bands):
    """
    Computes sinusoidal positional encoding for a scalar p.
    Returns a vector of size 2 * num_bands.
    """
    enc = []
    for k in range(2 * num_bands):
        freq = 2**k * math.pi
        if k % 2 == 0:
            enc.append(math.sin(freq * p))
        else:
            enc.append(math.cos(freq * p))
    return enc

def deform_encoding(p, displacement, num_bands):
    """
    Computes deformed positional encoding using tanh-constrained displacement.
    """
    base_enc = sinusoidal_encoding(p, num_bands)
    deformed = []
    for k in range(2 * num_bands):
        constrained_disp = math.tanh(displacement[k])
        deformed.append(base_enc[k] + constrained_disp)
    return deformed

def conditional_nerf_forward(x_coords, v_coords, z_s, z_a, displacement=None, num_bands=10):
    """
    Forward pass of Conditional NeRF.
    
    Args:
        x_coords: List of 3 floats [x, y, z]
        v_coords: List of 2 floats [phi, theta]
        z_s: Shape latent code vector
        z_a: Appearance latent code vector
        displacement: Optional displacement vector for deformation (size 2*num_bands)
        num_bands: Number of frequency bands for encoding
        
    Returns:
        sigma: Density
        c: Color vector [r, g, b]
    """
    # 1. Encode Position
    x_enc = []
    for coord in x_coords:
        if displacement is not None:
            x_enc.extend(deform_encoding(coord, displacement, num_bands))
        else:
            x_enc.extend(sinusoidal_encoding(coord, num_bands))
            
    # 2. Encode View Direction
    v_enc = []
    for coord in v_coords:
        v_enc.extend(sinusoidal_encoding(coord, num_bands))
        
    # 3. Concatenate with Latent Codes
    # Input to MLP: [x_enc, z_s, v_enc, z_a]
    # Note: In some variants (Face Prior), z is concatenated at each layer.
    # Here we assume a single concatenation at input for the base definition [S1].
    input_vector = x_enc + z_s + v_enc + z_a
    
    # 4. MLP Forward Pass
    # The MLP F_theta predicts (c, sigma)
    # Implementation of F_theta depends on specific architecture (e.g., MipNeRF360 based)
    # The specific MLP computation is not provided in the sources.
    sigma, c = None, None
    
    return sigma, c
```

## Subtopics

### Face Prior Model (Conditional NeRF)
A conditional neural radiance field $F_\theta$ trained as an auto-decoder [S2]. It predicts color $\hat{c} = F_\theta(r, w)$ given a ray $r$ and a latent identity code $w$ [S2]. It uses integrated positional encodings $\tilde{\gamma}_x$ for the point input and standard positional encoding $\gamma_v$ for the view direction (without integration) [S2]. The architecture is based on MipNeRF360 with two MLPs: a proposal network (predicts density only) and a NeRF MLP (predicts density and color) [S2]. Both MLPs take the encoded point and latent code $w$ as input, with the latent code concatenated at each layer [S2]. Unlike other generative models, it conditions on view direction $d$ in addition to the latent code, enabling view-dependent effects [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Performance:** The feed-forward approach to updating condition codes from text prompts avoids the time-consuming optimization of shape and appearance codes for each target sample [S1].
*   **Strengths:** Enables text-driven manipulation of both shape and appearance [S1]. The tanh constraint on deformation improves training robustness and avoids poor local minima [S1]. Conditioning on view direction allows for realistic view-dependent effects [S2].
*   **Limitations:** Requires training mappers ($M_s, M_a$) for text-to-code mapping [S1]. The Face Prior variant requires regularization to prevent overfitting when fitting to very few images [S2].
*   **When to use:** When text-driven 3D generation or manipulation is required [S1]. When few-shot face synthesis is needed [S2].
*   **When not to use:** Not given in the sources. [G]

## Background
Conditional NeRF was introduced in the paper "CLIP-NeRF: Text-and-Image Driven Manipulation of Neural Radiance Fields" [S1]. It builds on the idea of neural radiance fields by conditioning them on external inputs like text [S1]. The deformation network is inspired by Nerfies [S1], but differs by using a shape code to deform the global volume field of various object instances rather than a single fixed object [S1]. The Face Prior Model variant was introduced in "Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis" [S2], extending MipNeRF360 with conditioning inputs [S2].

## Sources
[S1] CLIP-NeRF: Text-and-Image Driven Manipulation of Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00381, fetched 2026-10-29
[S2] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
