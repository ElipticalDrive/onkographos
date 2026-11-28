# Differentiable Volumetric Rendering


## Overview
Differentiable Volumetric Rendering is a method that utilizes surface rendering primitives to compute gradients analytically, performing inference via a CNN encoder [S1, unofficial]. It distinguishes itself from linear or additive lighting models by accounting for complex non-linear lighting effects, specifically absorption and self-shadowing, which are critical for accurately capturing the visual appearance of real-world phenomena such as fluid markers [S2].

## How it works
The method operates within a fully differentiable volumetric rendering pipeline that integrates physical constraints with image-space observations [S2]. Unlike prior works that rely on linear image formation models suitable for thin marker densities with little absorption under uniform lighting, this approach handles non-linear attenuation [S2].

**Inputs and Outputs:**
*   **Inputs:** A small number of input views (potentially unposed or posed depending on the specific instantiation like SinGRAF, though the core [NG20] reference implies a general differentiable pipeline) and physical priors (e.g., Navier-Stokes equations for fluid reconstruction) [S2].
*   **Outputs:** Reconstructed volumetric fields (density $\rho$, velocity $\mathbf{u}$) or 3D radiance fields that match the input views while adhering to physical transport constraints [S1, unofficial][S2].

**Data Structures:**
*   **Volumetric Field:** Represented as a density field $\rho$ and potentially a velocity field $\mathbf{u}$ for dynamic scenes [S2].
*   **CNN Encoder:** Used for inference to map input images to latent codes or scene parameters [S1, unofficial].
*   **Surface Representation:** The method uses surface rendering primitives, implying a representation of the volume boundaries or isosurfaces rather than pure voxel marching for the gradient computation [S1, unofficial].

**Steps:**
1.  **Scene Representation:** The scene is represented using a structure that supports surface rendering (e.g., a neural radiance field or a latent code optimized via a hypernetwork) [S1, unofficial].
2.  **Differentiable Rendering:** The rendering process computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ using a model that accounts for absorption and self-shadowing. This involves non-linear attenuation of light as it passes through the medium [S2].
3.  **Analytic Gradient Computation:** Gradients are computed analytically with respect to the scene parameters (e.g., density, velocity, latent codes) rather than relying solely on reverse-mode automatic differentiation through the entire volumetric integral [S1, unofficial]. This reduces memory overhead compared to storing $O(kn)$ intermediate states .
4.  **Inference/Optimization:**
    *   For generative tasks (e.g., SinGRAF), a CNN encoder infers latent codes from unposed 2D images, which are then decoded into 3D radiance fields [S1, unofficial].
    *   For reconstruction tasks (e.g., fluid), the method optimizes initial density $\rho_0$ and velocity sequence $\{\mathbf{u}_0, \dots, \mathbf{u}_{t-1}\}$ to minimize a composite loss $\mathcal{L}_\rho$ that enforces correct transport over the fluid flow trajectory [S2].

**Formulas:**
The rendering model accounts for non-linear attenuation. While the standard emission-absorption model is:
$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$
This method extends the model to include self-shadowing and absorption effects, moving beyond the linear additive model $\hat{C} = \sum \mathbf{c}_i \delta_i$ [S2]. The exact closed-form gradient expressions are derived analytically [S1, unofficial].

**Parameters:**
*   **Loss Function:** A composite loss $\mathcal{L}_\rho$ constrains the density fields [S2].
*   **Physics Priors:** Navier-Stokes equations are used as physical priors for fluid reconstruction [S2].

**Edge Cases and Pitfalls:**
*   **Thin Densities:** Linear models fail for thick densities with significant absorption; this method is required in such cases [S2].
*   **Small Input Views:** Reconstruction from a very small number of input views is challenging and requires physical priors to reduce the solution space [S2].

## Subtopics

### [Differentiable Volumetric Rendering — Uses surface rendering, computes gradients analytically, and performs inference via a CNN encoder.](19661-differentiable-volumetric-rendering-uses-surface-rendering-c.md)
A 3D-aware generative model that uses surface rendering, computes gradients analytically, and performs inference via a CNN encoder. It distinguishes itself by employing a progressive-scale patch discrimination strategy and operating on unposed inputs with a tri-plane representation.

## Operating characteristics
*   **Memory:** Analytic gradient computation reduces memory consumption compared to standard reverse-mode AD, which requires storing $O(kn)$ intermediate states .
*   **Accuracy:** Handles non-linear attenuation and self-shadowing, providing higher fidelity for real-world marker observations and fluids compared to linear models [S2].
*   **Data Requirements:** Can operate on a very small number of input views if physical priors (like Navier-Stokes) are introduced [S2]. SinGRAF variant operates on unposed 2D images of a single scene [S1, unofficial].
*   **Complexity:** The use of surface rendering and analytic gradients simplifies the optimization landscape compared to full volumetric integration backpropagation [S1, unofficial].

## Background
The method is attributed to [NG20] (Neural Graphics Primitives or similar, cited as Differentiable Volumetric Rendering [NG20] in [S1, unofficial]). It was developed to address the limitations of linear lighting models in volumetric reconstruction. It is used in 3D-aware generative modeling (e.g., SinGRAF) and fluid reconstruction (e.g., Global Transport for Fluid Reconstruction) [S1, unofficial][S2].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
