# Conditional and Latent Representations

Also called: Conditional NeRF, Disentangled Conditional NeRF, CLIP-NeRF

## Overview
Conditional and Latent Representations extend the standard NeRF [Parent](3065-nerf-hyperparameters.md) to model an entire object category (e.g., chairs) rather than a single static scene, using instance-specific latent vectors for shape and appearance [S1, S2]. This approach solves the problem of efficiently representing diverse 3D instances within a unified model, enabling editing and generation without training separate models per instance [S1]. It differs from sibling methods by explicitly decoupling shape and appearance into separate learned codes, often utilizing a shared shape branch or deformation networks to enforce geometric consistency and prevent mutual interference between geometry and color [S1, S2].

## Shared principles
The core mechanism involves a neural network $F$ that maps a 3D location $\mathbf{x}$, viewing direction $\mathbf{d}$, and instance-specific latent codes to volumetric density $\sigma$ and radiance $\mathbf{c}$ [S1]. The latent codes are typically split into a shape code $\mathbf{z}^{(s)}$ and an appearance/color code $\mathbf{z}^{(c)}$ [S1].

1.  **Latent Code Injection:** The latent codes are injected into the network to condition the output. In standard conditional formulations, $\mathbf{z}^{(s)}$ and $\mathbf{z}^{(c)}$ are concatenated with positional and directional encodings [S1]. In disentangled variants, $\mathbf{z}^{(s)}$ is used to deform the positional encoding of $\mathbf{x}$, while $\mathbf{z}^{(c)}$ is deferred to the radiance prediction to ensure shape and appearance are independent [S1].
2.  **Rendering:** The rendering process follows standard volume rendering [Parent](3065-nerf-hyperparameters.md). For a ray $r$, $N_c$ points $\{t_i\}$ are sampled. Density $\sigma_i$ and color $\mathbf{c}_i$ are computed using $F$. The pixel color $\hat{C}$ is computed via alpha compositing:
    $$
    \hat{C}(r, \mathbf{z}^{(s)}, \mathbf{z}^{(c)}) = \sum_{i=1}^{N_c} \mathbf{c}_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
    $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $\delta_i = t_{i+1} - t_i$ [S1].
3.  **Disentanglement:** The density output $\sigma$ is designed to be independent of the viewing direction $\mathbf{d}$, while radiance $\mathbf{c}$ depends on both $\mathbf{x}$ and $\mathbf{d}$ [S1]. Disentanglement is achieved either by architectural separation (shared shape branch vs. instance-specific branches) [S1] or by deformation networks that map $\mathbf{z}^{(s)}$ to displacement vectors for positional encodings [S1].

## Subtopics

### [Conditional Radiance Field Training](15963-conditional-radiance-field-training.md)
A method for modeling shape and appearance variations across multiple 3D instances by training a single unified radiance field conditioned on instance-specific latent codes. It solves the problem of efficiently representing a dataset of diverse objects without training separate NeRF models for each, while maintaining rendering quality comparable to independent per-instance models. It differs from sibling methods by explicitly decoupling shape and appearance into separate learned codes and utilizing a shared shape branch to enforce geometric consistency across instances.

### [Conditional Radiance Field with Shared Branch](15964-conditional-radiance-field-with-shared-branch.md)
The Conditional Radiance Field with Shared Branch is a neural radiance field architecture designed for shape and color disentanglement, enabling efficient editing by allowing specific submodules to be finetuned. It maps a 3D location $\mathbf{x}$, viewing direction $\mathbf{d}$, and instance-specific shape code $\mathbf{z}^{(s)}$ and color code $\mathbf{z}^{(c)}$ to radiance $\mathbf{c}$ and scalar density $\sigma$. It sets itself apart from standard NeRF [Parent](3065-nerf-hyperparameters.md) by introducing a modular structure with a shared shape branch and instance-specific branches, rather than a single monolithic MLP or simple input concatenation.

### [Conditional NeRF Implementation Details](16755-conditional-nerf-implementation-details.md)
Conditional NeRF is a generative model for a specific object category that maps 5D coordinates (spatial position $\mathbf{x}$ and view direction $\mathbf{v}$) along with dedicated latent vectors for shape ($\mathbf{z}_s$) and appearance ($\mathbf{z}_a$) to volumetric density $\sigma$ and view-dependent radiance $\mathbf{c}$. It differs from standard NeRF by replacing the single-scene optimization with a category-level generative process controlled by explicit latent codes, and it distinguishes itself from sibling methods like [Conditional Radiance Field Training](15963-conditional-radiance-field-training.md) by enforcing disentanglement: shape conditioning is applied via volumetric deformation of positional encodings, while appearance conditioning is deferred to the radiance prediction network to prevent mutual interference between shape and color.

#### Conditional Radiance Field
**Definition:** A neural network representation that extends NeRF to model an entire object class (e.g., chairs) rather than a single scene, using latent vectors for shape and appearance to allow editing [S1]. The network $F$ takes a 3D location $\mathbf{x}$, viewing direction $\mathbf{d}$, and instance-specific latent vectors $\mathbf{z}^{(s)}$ and $\mathbf{z}^{(c)}$ to output radiance $\mathbf{c}$ and density $\sigma$ [S1]. The density output $\sigma$ is independent of the viewing direction, while the radiance output $\mathbf{c}$ depends on both position and viewing direction [S1]. The network $F$ is parametrized as a multi-layer perceptron (MLP) [S1].

**Step:** Sample $N_c$ 3D points $\{t_i\}_{i=1}^{N_c}$ along a cast ray $r$ from pixel location, ordered near to far [S1]. Compute radiance $\mathbf{c}_i$ and density $\sigma_i$ at each sampled point $t_i$ using network $F$ [S1].

**Formula:** Alpha compositing value: $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, where $\delta_i = t_{i+1} - t_i$ is the distance between adjacent samples [S1].

**Parameter:** $\mathbf{x}$: 3D location vector $(x, y, z)$; $\mathbf{d}$: viewing direction vector $(\phi, \theta)$; $\mathbf{z}^{(c)}$: latent color (appearance) vector [S1].

#### Continuous Manipulation via Latent Space Scaling
**Definition:** A technique to achieve gradual editing of shape and appearance by scaling the editing direction vectors in the latent space using a scalar factor [S2].

**Formula:** $\mathbf{z}_s = s \times \Delta \mathbf{z}_s + \mathbf{z}_{s0}$, $\mathbf{z}_a = s \times \Delta \mathbf{z}_a + \mathbf{z}_{a0}$, where $s$ is a scalar, $\mathbf{z}_s$ and $\mathbf{z}_a$ are the updated shape and appearance codes, $\mathbf{z}_{s0}$ and $\mathbf{z}_{a0}$ are the original codes, and $\Delta \mathbf{z}_s$ and $\Delta \mathbf{z}_a$ are the manipulation directions provided by the code mappers [S2].

**Parameter:** Scalar $s$: Controls the intensity of the edit. Typical range: 0 to 1.6 with step 0.2. Effect: Manipulation strength increases as $s$ increases [S2].

**Usage:** Used for continuous editing of both shape and appearance given a single text prompt or exemplar, allowing users to adjust the degree of change [S2].

#### CLIP-Driven Manipulation
**Step:** 1. Extract CLIP feature embedding from input text prompt or exemplar image. 2. Feed embedding into two separate feed-forward code mappers (shape mapper and appearance mapper). 3. Mappers output local displacements in the latent space to update $\mathbf{z}_s$ and $\mathbf{z}_a$. 4. Render image with updated codes. 5. Train mappers using a CLIP-based matching loss to enforce CLIP space consistency between input constraint and output renderings [S2].

**Comparison:** Differs from optimization-based editing (e.g., EditNeRF) by using learned feed-forward networks mapping user edits to latent codes, enabling fast inference for interactive editing instead of slow iterative optimization per sample [S2].

#### CLIP Manipulator Training
**Step:** Stage 2: Fix generator weights and train CLIP manipulation parts (shape mapper $M_s$ and appearance mapper $M_a$) [S2]. Sampling for mapper training: Randomly sample $\mathbf{z}_s$ from $Z_s$, $\mathbf{z}_a$ from $Z_a$, $\mathbf{v}$ from $Z_v$, and text prompt $t$ from pre-defined text library $T$. All network weights except mappers are fixed (denoted with hat) [S2].

**Formula:** Shape mapper loss $L_{shape}$: $L_{shape} = f(\hat{D}(\hat{F}_\theta(\mathbf{v}, M_s(\hat{E}_t(t)) + \mathbf{z}_s), \mathbf{z}_a)) + \lambda_c D_{CLIP}(\hat{F}_\theta(\mathbf{v}, M_s(\hat{E}_t(t)) + \mathbf{z}_s), \mathbf{z}_a, t)$ [S2]. Appearance mapper loss $L_{appear}$: $L_{appear} = f(\hat{D}(\hat{F}_\theta(\mathbf{v}, \mathbf{z}_s, M_a(\hat{E}_t(t)) + \mathbf{z}_a))) + \lambda_c D_{CLIP}(\hat{F}_\theta(\mathbf{v}, \mathbf{z}_s, M_a(\hat{E}_t(t)) + \mathbf{z}_a), t)$ [S2].

**Parameter:** $\lambda_c$ is the weight for the CLIP distance term in the mapper losses [S2].

## Background
Conditional NeRF was introduced by Chan et al. in 2021, building upon the original NeRF representation [45] to enable editing over entire shape classes like "chairs" [S1]. The method learns a conditional radiance field model extended with latent vectors over shape and appearance, trained over a set of shapes belonging to a class [S1]. CLIP-NeRF, introduced in 2022, extended this framework by integrating the CLIP model [34] to learn code mappers that map CLIP features to the latent space, enabling text-and-image driven manipulation [S2]. This allowed for fast inference for interactive editing and supported global deformation, differing from previous methods like EditNeRF [21] which relied on user scribbles and iterative optimization [S2]. The approach has been used for high-resolution NeRF manipulation and text-driven 3D content generation [S2].

## Sources
[S1] Editing Conditional Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00572, fetched 2026-10-26
[S2] CLIP-NeRF: Text-and-Image Driven Manipulation of Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00381, fetched 2026-10-29
