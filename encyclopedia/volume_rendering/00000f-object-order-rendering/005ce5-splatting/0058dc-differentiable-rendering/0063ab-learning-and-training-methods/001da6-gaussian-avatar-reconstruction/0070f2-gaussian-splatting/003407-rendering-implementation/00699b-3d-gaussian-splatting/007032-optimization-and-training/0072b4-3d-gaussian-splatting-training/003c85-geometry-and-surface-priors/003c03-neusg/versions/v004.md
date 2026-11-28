# NeuSG

Also called: NeuS-Gaussian Splatting, NeuS-3DGS

## Overview
NeuSG incorporates the NeuS neural surface reconstruction method into the 3D Gaussian Splatting (3DGS) representation to transfer explicit surface properties to the discrete Gaussian primitives [S1]. It solves the problem of vanilla 3DGS producing "floating ellipsoids" rather than coherent surfaces by enforcing geometric consistency with a neural Signed Distance Function (SDF) [S2, unofficial]. Unlike sibling methods that may use monocular depth or post-hoc mesh extraction, NeuSG directly couples a NeuS-style SDF network with the 3DGS rasterizer to align Gaussian centers and normals with the implicit surface [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians $\{G_i\}$, each defined by center $\mathbf{c}_i$, covariance $\Sigma_i$ (or rotation quaternion $\mathbf{q}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and spherical harmonic coefficients for color. A neural SDF network $f_\theta(\mathbf{x})$ initialized or trained alongside the Gaussians. Multi-view images.
*   **Outputs:** Optimized Gaussian attributes and SDF parameters that render photorealistic images with consistent surface geometry.

**Data Structures:**
*   **Gaussian Attributes:** Standard 3DGS layout. Center $\mathbf{c}_i \in \mathbb{R}^3$, Rotation $\mathbf{q}_i \in \mathbb{R}^4$ (normalized), Scale $\mathbf{s}_i \in \mathbb{R}^3$, Opacity $\alpha_i \in [0,1]$, SH coefficients.
*   **SDF Network:** A multi-layer perceptron (MLP) $f_\theta: \mathbb{R}^3 \to \mathbb{R}$ taking 3D coordinates and outputting signed distance values.

**Numbered Steps:**
1.  **Initialization:** Initialize 3D Gaussians from sparse points (e.g., SfM). Initialize the SDF network $f_\theta$.
2.  **Rendering:** Render images using the 3DGS rasterizer. Compute the photometric loss $\mathcal{L}_{rgb}$ between rendered and ground truth images.
3.  **SDF Evaluation:** For each Gaussian $i$, evaluate the SDF at its center: $d_i = f_\theta(\mathbf{c}_i)$.
4.  **Normal Computation:**
    *   Compute the geometric normal $\mathbf{n}_i$ of the Gaussian from its covariance matrix or rotation quaternion.
    *   Compute the surface normal $\mathbf{m}_i$ from the SDF gradient: $\mathbf{m}_i = \frac{\nabla f_\theta(\mathbf{c}_i)}{\|\nabla f_\theta(\mathbf{c}_i)\|}$.
5.  **Loss Calculation:** Compute the SDF consistency loss and normal consistency loss (see formulas below).
6.  **Optimization:** Minimize the total loss $\mathcal{L}_{total}$ with respect to Gaussian attributes and SDF parameters $\theta$.

**Formulas:**
The SDF consistency loss encourages Gaussians to lie on the zero-level set of the SDF [S1]:
$$ \mathcal{L}_{sdf} = \frac{1}{N} \sum_{i=1}^{N} |f_\theta(\mathbf{c}_i)| $$

The normal consistency loss aligns the Gaussian orientation with the SDF surface normal [S1]:
$$ \mathcal{L}_{normal} = \frac{1}{N} \sum_{i=1}^{N} (1 - \mathbf{n}_i \cdot \mathbf{m}_i) $$

The total objective is [S1]:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{sdf} \mathcal{L}_{sdf} + \lambda_{normal} \mathcal{L}_{normal} $$

**Parameters:**
*   $\lambda_{sdf}, \lambda_{normal}$: Weights for the geometric losses. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **SDF Initialization:** The SDF must be initialized to represent the scene volume correctly. Poor initialization may lead to local minima where Gaussians drift away from the surface.
*   **Gradient Stability:** Computing $\nabla f_\theta(\mathbf{c}_i)$ requires stable gradients from the SDF network. [G] Ensure the SDF network uses activation functions (e.g., softplus) that allow stable gradient computation near the zero-level set.
*   **Degenerate Gaussians:** If Gaussians become too small or large, the normal estimation $\mathbf{n}_i$ may become unstable. [G] Implement a minimum scale constraint during optimization.

## Subtopics

### NeuS Integration
The core mechanism of coupling a neural implicit surface representation with explicit splatting. Sets it apart by using a learned SDF rather than fixed depth maps. Choose when high-fidelity surface reconstruction is needed from multi-view inputs. [S1]

### SDF-Gaussian Consistency
The specific loss terms enforcing geometric alignment. Sets it apart from methods using only photometric loss. Choose when surface topology is critical. [S1]

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and the SDF network evaluation cost. SDF evaluation is $O(N \cdot D)$ where $D$ is the depth of the SDF MLP. Rendering is $O(N)$ per pixel.
*   **Memory Complexity:** $O(N)$ for Gaussian attributes plus $O(P)$ for SDF network parameters, where $P$ is the number of SDF weights.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Produces crisp depth and normal maps; surfaces are well-defined. [S2, unofficial]
*   **Limitations:** Requires training an additional neural network (SDF); may be slower than vanilla 3DGS due to SDF evaluations.
*   **When to use:** When surface geometry is required for downstream tasks (e.g., mesh extraction, physics). [S2, unofficial]
*   **When not to use:** When only photorealistic novel view synthesis is needed and surface consistency is not critical.

## Background
NeuSG was proposed to address the lack of explicit surface representation in 3D Gaussian Splatting. It builds on NeuS, a neural surface reconstruction method that uses volume rendering to learn SDFs [S1]. The method combines NeuS-style SDF with a GS rasterizer to transfer surface properties to 3DGS [S2, unofficial]. It is part of a broader trend of integrating geometric priors into 3DGS to enhance reconstruction quality [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
