# Two-Renderer Reconstruction Pipeline

Also called: Physical-Neural Renderer, Two-step rendering pipeline

## Overview
The Two-Renderer Reconstruction Pipeline is a differentiable inverse rendering method for translucent objects that decouples surface reflectance and subsurface scattering (SSS) into two distinct stages [S1]. It solves the problem of high memory/compute costs associated with full path tracing and the inability of direct-only renderers to model SSS by using a physically-based renderer for direct illumination and a neural network for the scattering effect [S1]. It differs from sibling methods like [VDN-NeRF](7383-vdn-nerf.md) and [Neural Radiative Transfer via Physics-Informed Optimization](10842-neural-radiative-transfer-via-physics-informed-optimization.md) by explicitly separating the rendering process into a non-differentiable (or fixed-parameter) physical step and a differentiable neural step, rather than optimizing a single volumetric field end-to-end.

## How it works
**Inputs and Outputs:**
The method takes three input images: a flash image $I_f \in \mathbb{R}^{3\times256\times256}$, a no-flash image $I \in \mathbb{R}^{3\times256\times256}$, and a binary mask $M \in \mathbb{R}^{256\times256}$ [S1]. The output is the estimated scene parameters: depth map $D \in \mathbb{R}^{256\times256}$, normal map $N \in \mathbb{R}^{3\times256\times256}$, roughness map $R \in \mathbb{R}^{256\times256}$, spherical harmonics $sh \in \mathbb{R}^{3\times9}$, flashlight intensity $i \in \mathbb{R}^1$, extinction coefficient $\sigma_t \in \mathbb{R}^3$, volumetric albedo $\alpha \in \mathbb{R}^3$, and Henyey-Greenstein parameter $g \in \mathbb{R}^1$ [S1].

**Data Structures:**
The estimated parameters are stored as dense maps or vectors as defined above. The neural renderer $f_n$ consists of three parts: a Surface encoder, a Scattering encoder, and a decoder [S1]. The Surface encoder takes the direct illumination image $\tilde{I}_d$ and the background image $B$ (masked-out version of $I_f$) [S1]. The Scattering encoder takes the illumination and SSS parameters [S1].

**Steps:**
1.  **Estimation:** A deep convolutional neural network with one encoder and multiple heads estimates the parameters $D, N, R, sh, i, \sigma_t, \alpha, g$ from the input images $I_f, I, M$ [S1].
2.  **Direct Illumination Rendering:** Compute the direct illumination image $\tilde{I}_d$ using a physically-based renderer $f_d$:
    $$ \tilde{I}_d = f_d(\tilde{D}, \tilde{N}, \tilde{R}, \tilde{i}, \tilde{sh}) $$
    where $f_d$ follows the implementation of [49] and has no trainable parameters [S1].
3.  **Neural SSS Rendering:** Compute the final flash image $\tilde{I}_f$ using the neural renderer $f_n$:
    $$ \tilde{I}_f = f_n(\tilde{I}_d, \tilde{sh}, \tilde{i}, \tilde{\sigma}_t, \tilde{\alpha}, \tilde{g}, B) $$
    where $f_n$ consists of a Surface encoder, a Scattering encoder, and a decoder [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_renderer_pipeline(input_flash, input_no_flash, mask):
    # Step 1: Estimate parameters using the estimator network
    # The estimator uses one encoder and multiple heads
    D, N, R, sh, i, sigma_t, alpha, g = estimator(input_flash, input_no_flash, mask)
    
    # Step 2: Compute direct illumination using physically-based renderer
    # f_d is non-trainable and follows implementation [49]
    I_d = physically_based_renderer(D, N, R, i, sh)
    
    # Extract background from flash image using mask
    # B is the masked-out version of input_flash
    B = input_flash * (1 - mask)
    
    # Step 3: Compute final image using neural renderer
    # f_n consists of Surface encoder, Scattering encoder, and decoder
    I_f = neural_renderer(I_d, sh, i, sigma_t, alpha, g, B)
    
    return I_f, D, N, R, sh, i, sigma_t, alpha, g
```

**Parameters:**
Typical input resolution is $256 \times 256$ [S1]. Spherical harmonics dimension is 9 [S1].

**Edge Cases:**
The background image $B$ must be masked out from the flash image to provide high-frequency illumination information without the object's direct reflection [S1].

## Subtopics

### Refractive Index Tomography using Neural Fields
Uses neural fields for tomography, focusing on refractive index rather than SSS parameters. [Link](4146-refractive-index-tomography-using-neural-fields.md)

### VDN-NeRF
Optimizes a single neural volume, unlike the two-stage separation. [Link](7383-vdn-nerf.md)

### Inverse Rendering and Optical Properties
General inverse rendering, may not use the specific two-step physical-neural decoupling. [Link](7914-inverse-rendering-and-optical-properties.md)

### Planar Reflection-Aware NeRF
Focuses on planar reflections, not SSS. [Link](8312-planar-reflection-aware-nerf.md)

### Neural Radiative Transfer via Physics-Informed Optimization
Uses physics-informed optimization within a single neural field framework. [Link](10842-neural-radiative-transfer-via-physics-informed-optimization.md)

### Neural Illumination Fields (NeIF)
Focuses on illumination estimation, not necessarily the two-step SSS decoupling. [Link](10930-neural-illumination-fields-neif.md)

### Multi-Sensor and Physical Priors
Uses multi-sensor data, not specifically the two-renderer pipeline. [Link](11031-multi-sensor-and-physical-priors.md)

## Operating characteristics
The method avoids the high memory and compute cost of full path tracing by using a neural renderer for SSS [S1]. The physically-based renderer $f_d$ has no trainable parameters, reducing the optimization burden [S1]. The neural renderer $f_n$ is trained to mimic SSS effects [S1]. Performance details such as specific hardware or resolution scaling beyond $256 \times 256$ are not given in the sources.

## Background
The method was proposed in "Inverse Rendering of Translucent Objects using Physical and Neural Renderers" [S1]. It builds on advances in image-to-image translation and neural networks for adding indirect illumination [S1].

## Sources
[S1] Inverse Rendering of Translucent Objects using Physical and Neural Renderers, https://doi.org/10.1109/cvpr52729.2023.01204, fetched 2026-10-26
