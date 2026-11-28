# Neural Emitter Field

Also called: Emission Field, Emitter Semantic Field, GlossyNEF

## Overview
The Neural Emitter Field (NEF) is a neural representation that pre-integrates the emission of a light source around its geometry to enable real-time rendering of complex luminaires [S2, unofficial]. It solves the problem of expensive direct illumination computation by learning a continuous field that approximates the radiance distribution, avoiding the linear scaling cost of analytical methods like Linearly Transformed Cosines (LTC) with respect to emitter vertex count [S2, unofficial]. It is distinct from sibling surface-SDF methods (e.g., [NeuS](6060-neus.md), [VolSDF](10715-volsdf.md)) which model scene geometry and albedo; NEF specifically models the *emission* properties of light sources, often integrated into a larger SDF-based reconstruction pipeline [S1].

## How it works
The method consists of two primary variants: the **Semantic NEF** (used for intrinsic scene reconstruction and emitter segmentation) [S1] and the **GlossyNEF** (used for real-time pre-integrated lighting) [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   3D position $\mathbf{x}$.
    *   For Semantic NEF: Latent code $\mathbf{z}(\mathbf{x})$ from the SDF field $F_d$ [S1].
    *   For GlossyNEF: Position $\mathbf{x}$, outgoing direction $\boldsymbol{\omega}_o$, and reflected view direction $\boldsymbol{\omega}_r$ [S2, unofficial].
*   **Outputs:**
    *   Semantic NEF: Emitter mask value $\hat{m}(\mathbf{x})$ and binary classification (emitter/non-emitter) [S1].
    *   GlossyNEF: Pre-integrated radiance/emission values for diffuse and glossy components [S2, unofficial].

### Data Structures

#### Semantic NEF Data Structures
1.  **SDF Field $F_d$**: A standard neural SDF network.
2.  **Latent Code $\mathbf{z}(\mathbf{x})$**: Output of $F_d$ before the SDF regression head, providing semantic/latent scene information [S1].
3.  **Emitter Network $F_e$**: A neural network taking $\mathbf{z}(\mathbf{x})$ as input [S1].
4.  **Emission Array $L[\cdot]$**: An array of size $K$ storing learnable HDR emission values for $K$ clustered emitters [S1].

#### GlossyNEF Data Structures
1.  **Multiresolution Hash Grid**:
    *   **Entries**: $2^{19}$ [S2, unofficial].
    *   **Levels**: 8 levels [S2, unofficial].
    *   **Scaling**: Each level increases in size by a factor of $\sqrt{2}$ [S2, unofficial].
    *   **Latent Vector**: 4 floats per entry [S2, unofficial].
    *   **Memory**: 15 MB for standard NEF, 30 MB for GlossyNEF (half-precision) per emitter [S2, unofficial].
2.  **Network Heads**: Two output heads (diffuse and glossy) [S2, unofficial].

### Steps and Formulas

#### Semantic NEF (Emitter Segmentation)

1.  **Compute Latent Code**: For a 3D point $\mathbf{x}$, compute the latent code $\mathbf{z}(\mathbf{x})$ using the SDF field $F_d$ [S1].
2.  **Evaluate Emitter Mask**: Pass $\mathbf{z}(\mathbf{x})$ through the emitter network $F_e$ to get the local mask value:
    $$ \hat{m}(\mathbf{x}) = F_e(\mathbf{z}(\mathbf{x})) $$
    [S1]
3.  **Volume Accumulation**: For a ray $\mathbf{r}$ with samples $i=1 \dots M$, compute the accumulated emitter mask $\hat{M}_e(\mathbf{r})$:
    $$ \hat{M}_e(\mathbf{r}) = \sum_{i=1}^{M} T_i \alpha_i r_i \hat{m}_i $$
    where $T_i$ is transmittance, $\alpha_i$ is opacity, $r_i$ is the distance step, and $\hat{m}_i$ is the emitter mask value at sample $i$ [S1].
4.  **Optimization**: Train $F_e$ using binary cross-entropy loss against ground truth emitter masks $M_e(\mathbf{r})$:
    $$ L_{emi} = \sum_{\mathbf{r} \in R} \left[ M_e(\mathbf{r}) \log \hat{M}_e(\mathbf{r}) + (1 - M_e(\mathbf{r})) \log (1 - \hat{M}_e(\mathbf{r})) \right] $$
    [S1]
5.  **Clustering and Storage**: After training, cluster emitter points using K-Means with $K$ clusters. Store the emission values in the array $L[\cdot]$ indexed by cluster ID [S1].

#### GlossyNEF (Real-time Pre-integration)

1.  **Encoding**: Apply multiresolution hash encoding to position $\mathbf{x}$ and outgoing direction $\boldsymbol{\omega}_o$. **Note**: Encoding $\boldsymbol{\omega}_o$ is critical; encoding the reflected direction $\boldsymbol{\omega}_r$ reduces quality across all roughness values [S2, unofficial].
2.  **Network Forward Pass**:
    *   The GlossyNEF architecture uses two output heads: diffuse and glossy [S2, unofficial].
    *   Both heads use hash-grid encodings on $\mathbf{x}$ and $\boldsymbol{\omega}_r$ (for the glossy head) or $\boldsymbol{\omega}_o$ (for the diffuse head, implied by standard practice and the ablation study noting shared positional encoding) [S2, unofficial].
    *   *Pitfall*: Decoupling the hash table for diffuse and glossy heads adds runtime capacity but yields no quality improvement [S2, unofficial].
3.  **Rendering**: The network outputs pre-integrated radiance values, allowing constant-time shading regardless of emitter tessellation density [S2, unofficial].

### Parameters
*   **Hash Table Size**: $2^{19}$ entries [S2, unofficial].
*   **Hash Grid Levels**: 8 [S2, unofficial].
*   **Latent Vector Size**: 4 floats [S2, unofficial].
*   **Scaling Factor**: $\sqrt{2}$ per level [S2, unofficial].
*   **Memory Budget**: 15 MB (standard) or 30 MB (GlossyNEF) per emitter [S2, unofficial].

### Edge Cases and Pitfalls
*   **Direction Encoding**: Using hash encoding on the reflected direction $\boldsymbol{\omega}_r$ instead of the outgoing direction $\boldsymbol{\omega}_o$ significantly reduces quality across all roughness values [S2, unofficial].
*   **Hash Table Decoupling**: Using separate hash tables for diffuse and glossy heads increases capacity but does not improve quality [S2, unofficial].
*   **Under-estimation of Emitter Area**: In the Semantic NEF context, if the emitter area is under-estimated, it causes over-dark estimation in the re-render stage [S1].

## Operating characteristics
*   **Time Complexity**:
    *   **Inference**: Constant time with respect to emitter tessellation density [S2, unofficial].
    *   **Shading Time**: Maintains a constant 2.5 ms shading time at 1920x1080 resolution, even for complex emitters like the Stanford Dragon (436k vertices) [S2, unofficial].
    *   **Comparison**: Outperforms LTC (Linearly Transformed Cosines) at 267 emitter vertices for glossy receivers; LTC scales linearly with vertex count [S2, unofficial].
*   **Memory Complexity**: 15 MB to 30 MB per emitter [S2, unofficial].
*   **Training Time**: Ranges from ~1 minute for simple planar emitters to 3 hours for emitters with internal interreflections, depending on the cost of generating path-traced samples [S2, unofficial].
*   **Strengths**: Real-time rendering capability for complex, high-polygon emitters; amortizable training cost across multiple scenes [S2, unofficial].
*   **Limitations**: Requires offline training; memory overhead per emitter is fixed and may be excessive for very simple emitters [S2, unofficial].

## Background
The Neural Emitter Field was introduced to address the computational bottleneck of real-time rendering with complex, high-polygon light sources. Traditional analytical methods like LTC scale linearly with the number of emitter vertices, making them prohibitive for detailed meshes [S2, unofficial]. NEF pre-integrates the emission field, allowing for constant-time inference [S2, unofficial]. The concept of using a neural field for emitter semantics ($F_e$) was also explored in the context of intrinsic scene reconstruction to separate geometry from lighting [S1].

## Sources
[S1] I 2 -SDF: Intrinsic Indoor Scene Reconstruction and Editing via Raytracing in Neural SDFs, https://doi.org/10.1109/cvpr52729.2023.01202, fetched 2026-11-01
[S2] Real-time Rendering of Pre-integrated Neural Emitters, https://arxiv.org/abs/2610.06762, fetched 2026-10-30 (unofficial)
