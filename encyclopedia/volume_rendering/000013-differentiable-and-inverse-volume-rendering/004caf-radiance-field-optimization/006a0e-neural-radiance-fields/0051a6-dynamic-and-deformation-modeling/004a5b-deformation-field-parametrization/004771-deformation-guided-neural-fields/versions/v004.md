# Deformation-Guided Neural Fields

Also called: Deformation Field NeRF, Morphable Model Guided NeRF, RigNeRF

## Overview
Deformation-Guided Neural Fields address the problem of modeling dynamic, articulated subjects (specifically human faces) in neural radiance fields by decoupling identity from pose and expression. Unlike static NeRFs, this method constructs a fully 3D face portrait where geometry and appearance are controllable via pose and expression parameters. It sets itself apart from sibling methods by explicitly guiding the deformation field MLP with a parametric 3D morphable model (3DMM), ensuring that the learned deformations adhere to anatomical priors rather than relying solely on unconstrained neural predictions [S1].

## How it works
The method operates by mapping 3D spatial coordinates from the canonical (neutral) space to the deformed (current pose/expression) space using a deformation field, which is then queried by a standard appearance field.

**Inputs:**
*   Training images with known camera poses and 3DMM parameters (pose, expression coefficients).
*   A pre-trained 3D Morphable Model (3DMM) providing the mean face geometry and basis vectors for shape and expression variations [S1].

**Outputs:**
*   A trained neural field capable of rendering novel views of the face with arbitrary pose and expression.
*   Controllable 3D face portrait [S1].

**Data Structures:**
1.  **3DMM Model:** Defines the canonical mesh $\mathbf{V}_{canon}$ and basis shapes $\mathbf{S}_{shape}$ and expressions $\mathbf{S}_{exp}$.
2.  **Deformation Field MLP ($\mathcal{M}_{def}$):** A neural network that takes canonical coordinates and expression/pose parameters as input and outputs a displacement vector.
3.  **Appearance Field MLP ($\mathcal{M}_{app}$):** A standard NeRF MLP that takes deformed 3D coordinates and viewing direction as input and outputs density $\sigma$ and color $\mathbf{c}$.

**Steps:**
1.  **Canonical Space Definition:** Define the canonical space as the neutral expression state of the subject. The 3DMM provides the mean shape $\mathbf{V}_{mean}$ [S1].
2.  **Deformation Field Evaluation:** For a query point $\mathbf{x}_{canon}$ in canonical space, compute the deformation $\Delta \mathbf{x}$ using the deformation field MLP $\mathcal{M}_{def}$. The input to $\mathcal{M}_{def}$ includes $\mathbf{x}_{canon}$ and the current expression/pose parameters $\theta$ derived from the 3DMM fit [S1].
    $$ \Delta \mathbf{x} = \mathcal{M}_{def}(\mathbf{x}_{canon}, \theta) $$
3.  **Coordinate Transformation:** Map the canonical coordinate to the deformed space $\mathbf{x}_{def}$:
    $$ \mathbf{x}_{def} = \mathbf{x}_{canon} + \Delta \mathbf{x} $$
4.  **Appearance Query:** Query the appearance field $\mathcal{M}_{app}$ with the deformed coordinate $\mathbf{x}_{def}$ and the viewing direction $\mathbf{d}$ to obtain density $\sigma$ and color $\mathbf{c}$:
    $$ (\sigma, \mathbf{c}) = \mathcal{M}_{app}(\mathbf{x}_{def}, \mathbf{d}) $$
5.  **Volume Rendering:** Render the image using the standard emission-absorption model (see [44-radiance-field-optimization.md](44-radiance-field-optimization.md)) with the sampled $(\sigma, \mathbf{c})$ values along rays.
6.  **Optimization:** Minimize the photometric loss between rendered and ground-truth images. The 3DMM guidance ensures that the deformation field $\mathcal{M}_{def}$ learns physically plausible facial movements [S1].

**Parameters:**
*   3DMM coefficients (pose, expression): Typically 3DMM-specific dimensions (e.g., 100-300 coefficients) [S1].
*   MLP hidden layers: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Fitting Errors:** If the 3DMM fit to the input images is inaccurate, the canonical space definition may be misaligned, leading to artifacts. [G] A common mitigation is to include a mask supervision loss to ensure the rendered alpha channel aligns with the subject's silhouette.
*   **Degenerate Input:** Extreme expressions outside the 3DMM basis may cause extrapolation errors in the deformation field.

## Subtopics

### [Canonical Face Space Transformation](13405-canonical-face-space-transformation.md)
Canonical Face Space Transformation normalizes world coordinates to a canonical face space using a rigid transform to address face shape variations between training data and real-world inputs, making the NeRF robust to geometry and pose variations. It sets itself apart from methods that rely on detailed mesh priors or non-rigid deformation fields by using only a rigid alignment derived from a mean geometry, avoiding the need for mesh details during view synthesis.

### Mesh-anchored Hash Table Blendshapes
Mesh-anchored Hash Table Blendshapes is a 3D neural avatar system achieving real-time rendering with high fidelity and fine-grained control through mesh-anchored hash table blendshapes [S1]. Each vertex of a 3DMM mesh is linked to a local hash table, allowing for expression-dependent embeddings and localized facial deformations [S1]. Local blendshapes are combined using per-vertex weights predicted in UV space from driving signals [S1]. It uses a hash encoding with decoding MLP in a hybrid neural implicit field to predict color and density from 3D queries using volume rendering [S1].

### HiDe-NeRF
HiDe-NeRF is a neural radiation field technique for detailed, unobstructed dynamic head synthesis [S2]. It depicts a three-dimensional dynamic environment as a standard appearance field encompassing the standard source face, and a subtle deformation field that mimics the primary pose and expression [S2]. HiDe-NeRF is equipped with a multi-scale generalized appearance module (MGA) and a streamlined expression-aware deformation module (LED) to enhance expression precision while preserving identity data [S2].

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the evaluation of two MLPs (deformation and appearance) per sample point, plus the 3DMM projection if computed per sample. This is generally slower than static NeRFs but comparable to other deformation-based methods like Neural Body [S1].
*   **Memory Complexity:** Requires storing the parameters for both the deformation and appearance MLPs, plus the 3DMM model data. The 3DMM itself is lightweight compared to the neural fields.
*   **Strengths:** Provides explicit control over pose and expression; robust to identity variations; leverages strong anatomical priors from the 3DMM [S1].
*   **Limitations:** Dependent on the quality of the 3DMM fit; may struggle with non-facial regions or occlusions not covered by the face model; limited to the expression range of the 3DMM basis.
*   **When to use:** When controllable, high-fidelity 3D face avatars are required from video or image collections.
*   **When not to use:** For non-rigid, non-anatomical dynamic scenes or when 3DMM fitting is unreliable (e.g., heavy occlusion, extreme angles).

## Background
Deformation-Guided Neural Fields emerged as a solution to the limitations of standard NeRFs in handling moving subjects. While standard NeRFs struggle with moving bodies, mesh deformation approaches like Neural Body demonstrated the ability to interpolate between frames and poses [S1]. RigNeRF, introduced in June 2022, innovated on this topic by using deformation field MLPs guided by a morphable 3D face model, creating a fully 3D face portrait with controllable pose and expression [S1]. This approach built upon the popular paradigm of animating articulated bodies using a baseline skeleton equipped with an MLP-based deformation field, a paradigm established by works such as A-NeRF (Feb 2021) and Animatable NeRF (May 2021) [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
