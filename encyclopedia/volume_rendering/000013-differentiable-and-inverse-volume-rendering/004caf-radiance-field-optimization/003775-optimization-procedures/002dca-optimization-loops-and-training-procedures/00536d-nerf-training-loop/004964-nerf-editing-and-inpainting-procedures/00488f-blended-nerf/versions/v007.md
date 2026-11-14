# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields

## Overview
Blended-NeRF is a framework for editing a specific region of interest (ROI) in an existing Neural Radiance Field (NeRF) scene based on text prompts and a 3D ROI box [S1]. It solves the problem of local 3D editing by initializing a generator NeRF with the original scene's weights and blending the synthesized content with the original scene using volumetric blending [S1]. Unlike methods that synthesize new objects from scratch or require semantic decomposition/2D masks for localization, Blended-NeRF operates on any region using a 3D box without learning a new feature space [S1].

## How it works
**Inputs:**
*   Pre-trained NeRF model $F_{\theta_O}$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [S1].
*   3D Region of Interest (ROI) box $B$ localized inside the scene [S1].
*   User-provided text prompt $y$ [S1].
*   Language-image model encoders $E_{img}$ and $E_{txt}$ (e.g., CLIP) [S1].

**Outputs:**
*   Edited scene representation $F_{\theta_E}$ (implicitly defined by the blending of $F_{\theta_O}$ and $F_{\theta_G}$) [S1].
*   Rendered views $x_e$ of the edited scene [S1].

**Data Structures:**
*   $F_{\theta_O}$: MLP model for original scene.
*   $F_{\theta_G}$: MLP model for generated content, initialized with weights of $F_{\theta_O}$ [S1].
*   $B$: 3D bounding box defined by center and dimensions [S1].

**Positional Encoding:**
Inputs are embedded into a high-dimensional space using high frequency sinusoidal positional encoding $\gamma(\mathbf{x})$ [S1]:
$$ \gamma(\mathbf{x}) = [\cos(2^l \mathbf{x}), \sin(2^l \mathbf{x})]_{l=0}^{L-1} $$
where $L$ is the number of frequency bands [S1].

**Training Steps:**
1.  **Initialization:** Initialize $F_{\theta_G}$ with the weights of $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Localize the 3D box $B$ inside the scene using a GUI that renders the original NeRF and uses the output depth map for 3D understanding [S1].
3.  **Pose Sampling:** For each training iteration, sample a random camera pose around the scene with azimuth/elevation angles $(\theta, \phi)$ [S1]. The initial distance $d$ from the ROI is calculated as:
    $$ d = \frac{e_{max}}{2 \tan(\gamma/2)} $$
    where $e_{max}$ is the maximum dimension of the box $B$ and $\gamma$ is the camera FOV. Randomly sample radius $r$ around $d$ [S1].
4.  **Ray Casting and Sampling:** Cast rays $r$ from the sampled camera pose. Sample points $\mathbf{x}_i$ along the ray [S1].
5.  **ROI Masking:** For sample points $\mathbf{x}_i$ outside $B$, set density to zero. Render the generated content inside the ROI box $I_{ROI}$ using only points $\mathbf{x}_i \in B$ [S1]. The rendered color $C(r)$ for a ray $r$ is:
    $$ C(r) = \begin{cases} \frac{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i}) c_i}{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i})} & \text{if } \exists \mathbf{x}_i \in r \text{ s.t. } \mathbf{x}_i \in B \\ 0 & \text{otherwise} \end{cases} $$
    where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_j)$, $\delta_i = t_{i+1} - t_i$ is the distance between adjacent samples, $\sigma_i$ is density, and $c_i$ is color [S1].
6.  **Background Augmentation:** Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) during optimization to prevent scene diffusion [S1].
7.  **Directional Dependent Prompts:** Concatenate view-dependent text prompt to the original caption to constrain object description across views [S1].
8.  **Loss Computation:**
    *   **Similarity Loss:** Minimize cosine distance between rendered ROI image and text prompt:
        $$ L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y) $$
        where $I_{ROI}$ is the rendered view of the ROI [S1].
    *   **Transmittance Loss:** Encourage sparse/coherent results:
        $$ L_T = -\min(\tau, \text{mean}(T(P))) $$
        where $T(P)$ is transmittance of rendered view from pose $P$, and $\tau$ is max transmittance [S1].
    *   **Depth Loss:** Encourage volumetric 3D shapes (especially for forward-facing scenes):
        $$ L_D = -\min(\rho, \sigma^2(D(P))) $$
        where $\sigma^2(D(P))$ is variance of disparity map of rendered view from pose $P$, and $\rho$ is max variance allowed [S1].
    *   **Total Objective:**
        $$ L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D $$
        where $\lambda_T, \lambda_D$ are weights [S1].
    *   **Annealing:** Gradually introduce $L_T$ and $L_D$ during training to prevent completely transparent or amorphous scenes [S1].
9.  **Optimization:** Optimize $F_{\theta_G}$ by minimizing $L_{total}$ [S1].

**Blending Steps (Inference):**
After training, render the edited scene by blending sample points from $F_{\theta_O}$ and $F_{\theta_G}$ along each view ray [S1].

1.  **Query Models:** Query both $F_{\theta_O}$ and $F_{\theta_G}$ for the same set of rays [S1].
2.  **Mode Selection:**
    *   **Object Insertion/Replacement:** Consider only radiance field of $F_{\theta_G}$ inside the ROI during training. After training, blend using distance smoothing [S1].
    *   **Object Blending:** Query both models inside the box [S1].
3.  **Object Blending Calculation:**
    *   Compute alpha values for each point $\mathbf{x}_i$ on the ray separately from each model:
        $$ \alpha_O(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_O(\mathbf{x}_i)) \cdot \delta_i) $$
        $$ \alpha_G(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_G(\mathbf{x}_i)) \cdot \delta_i) $$
        where $\phi$ is a non-negative activation function [S1].
    *   **Density Blending Options:**
        *   Option 1 (Sum in activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i) + \sigma_G(\mathbf{x}_i))$. Allows $F_{\theta_G}$ to remove original densities if $\sigma_G < 0$ [S1].
        *   Option 2 (Sum out activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i)) + \phi(\sigma_G(\mathbf{x}_i))$. Allows only addition of new densities [S1].
    *   **Color Blending:**
        $$ c(\mathbf{x}_i) = S\left(\frac{c_O(\mathbf{x}_i) \cdot \alpha_O(\mathbf{x}_i) + c_G(\mathbf{x}_i) \cdot \alpha_G(\mathbf{x}_i)}{\epsilon + \alpha_O(\mathbf{x}_i) + \alpha_G(\mathbf{x}_i)}\right) $$
        where $S$ is sigmoid, $\epsilon$ is a small constant for numerical stability [S1].
4.  **Distance Smoothing (Insertion/Replacement Mode):**
    Blend raw densities and RGB values inside and outside the ROI using a distance smoothing operator [S1]:
    $$ f(\mathbf{x}) = 1 - \exp\left(-\alpha \frac{d(\mathbf{x})}{diag}\right) $$
    $$ \sigma_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \sigma_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \sigma_G(\mathbf{x}) $$
    $$ c_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot c_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot c_G(\mathbf{x}) $$
    where $d(\mathbf{x})$ is Euclidean distance from ROI center, $diag$ is box diagonal, and $\alpha$ is a hyperparameter controlling blending strength [S1].
5.  **Final Rendering:** The final rendered view $x_e$ is obtained by blending the raw densities and RGB values inside and outside the ROI along each ray using standard volume rendering (eq. 1 from parent) [S1].

**Parameters:**
*   $\alpha$: Hyperparameter controlling blending strength. $\alpha=0$ gives all weight to edited scene; higher $\alpha$ gives more attention to original scene points closer to ROI center. Typical values: 0, 0.5, 2, 4, 10 [S1].
*   $\lambda_T, \lambda_D$: Weights for transmittance and depth losses [S1].
*   $\tau, \rho$: Max transmittance and max variance allowed [S1].

**Edge Cases and Pitfalls:**
*   **Scene Diffusion:** Using white or black backgrounds during optimization causes the scene to populate the background. Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) [S1].
*   **Billboard Effect:** In forward-facing scenes with limited viewing intervals, edits may appear as flat billboards. Use Depth Loss $L_D$ to encourage volumetric shapes [S1].
*   **Transparency:** Without regularization, scenes may become completely transparent. Use Transmittance Loss $L_T$ and annealing [S1].
*   **View Consistency:** Without directional prompts, the object may not be consistent across views. Use directional dependent prompts [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of training iterations and the number of samples per ray. Requires optimizing a new MLP $F_{\theta_G}$ initialized from $F_{\theta_O}$ [S1].
*   **Memory Complexity:** Requires storing weights for both $F_{\theta_O}$ and $F_{\theta_G}$ during training and blending [S1].
*   **Strengths:** Can operate on any region of a real-world scene; modifies only the ROI while preserving the rest; generates natural-looking and view-consistent results; not restricted to a specific class or domain; enables complex text-guided manipulations [S1].
*   **Limitations:** Requires a pre-trained NeRF; relies on CLIP similarity which may not always align with human perception; blending artifacts may occur if $\alpha$ is not tuned correctly [S1].
*   **When to use:** When editing a specific object or region in an existing NeRF scene using text prompts, especially when avoiding the need for 2D masks or semantic decomposition [S1].
*   **When not to use:** When synthesizing a completely new object from scratch without an existing scene context (use Dream Fields or similar) [S1].

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds on the principles of Dream Fields [27], utilizing similar priors and augmentations (pose sampling, background augmentation, transmittance/depth losses) to improve the realism of text-guided NeRF generation. Unlike Dream Fields which synthesizes new NeRF objects from scratch, Blended-NeRF edits existing NeRF scenes by initializing the generator with the existing scene's weights and blending the results [S1]. It addresses limitations of prior editing methods like Kobayashi [29] and Benaim et al. [8], which require semantic decomposition or 2D masks for localization [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
