# CoherentGS

Also called: Coherent 3D Gaussians, CoherentGS

## Overview
CoherentGS is a regularized optimization approach for sparse novel view synthesis that introduces coherency to an explicit 3D Gaussian representation by constraining Gaussian movement in 2D image space, initialized with monocular depth estimates [S1]. It addresses the instability and "needle-like" artifacts of standard 3D Gaussian Splatting (3DGS) when trained on extremely sparse inputs (e.g., 3–4 images) by enforcing single-view and multi-view constraints, preventing overfitting and allowing for the identification of occluded regions suitable for inpainting [S1]. Unlike NeRF-based sparse methods that rely on implicit MLP coherency and produce blurry occluded regions, CoherentGS constrains explicit Gaussians in 2D image space, enabling sharper details and structured hallucination via diffusion models [S1].

## How it works
**Inputs:**
*   A sparse set of $N$ posed input images (typically $N=3$ or $4$) [S1].
*   Monocular depth estimates for each input view, predicted using a model such as Depth Anything [S1].
*   Dense flow correspondences between all pairs of input images, predicted using a model such as FlowFormer++ [S1].
*   Binary segmentation masks derived from thresholding the monocular depth estimates to preserve sharp discontinuities [S1].

**Outputs:**
*   A set of 3D Gaussians defined by position $\mathbf{x}$, color $\mathbf{c}$, covariance matrix $\Sigma$, and opacity $\alpha$ [S1].

**Data Structures:**
*   **Gaussian Set:** A collection of $M$ Gaussians, where $M$ equals the total number of pixels across all $N$ input images. Each Gaussian $i$ is associated with a specific pixel $\mathbf{p}_i$ in a specific input view $v_i$.
*   **Implicit Decoder:** A convolutional neural network that takes a 2D image coordinate and camera pose as input and predicts a residual depth value $\Delta D$ [S1].

**Numbered Steps:**

1.  **Structured Initialization:** Assign exactly one Gaussian to every pixel of each input image to create a structured representation [S1].
2.  **Depth-Based Initialization:** Initialize the 3D position $\mathbf{x}_i$ of each Gaussian using the monocular depth estimate $D_{init}$ for its corresponding pixel in its input view. The position is computed by back-projecting the pixel coordinate using the camera intrinsics and the initial depth $D_{init}$ [S1].
3.  **Regularized Optimization:** Minimize the optimization objective by iteratively updating Gaussian parameters. During each optimization step:
    a. **Rendering:** Compute the rendered color $R(\mathbf{p})$ for each pixel using the volumetric rendering equation [S1]:
    $$ R_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}}(\mathbf{p}) = \sum_{i \in \mathcal{N}(\mathbf{p})} \mathbf{c}_i \gamma_i \prod_{j=1}^{i-1}(1-\gamma_j) $$
    where $\mathcal{N}(\mathbf{p})$ is the set of ordered Gaussians overlapping pixel $\mathbf{p}$, and $\gamma_k = f(\Sigma_k, \alpha_k, \mathbf{x}_k, \mathbf{p})$ computes the effective opacity by evaluating the Gaussian at $\mathbf{p}$ and multiplying by global opacity $\alpha$ [S1].
    b. **Single-View Coherency:** For each input view, pass the input view coordinate to the implicit decoder to predict a residual depth value $\Delta D$. The Gaussian positions are constrained such that Gaussians with similar initial depth move together via this implicit decoder. To preserve sharp edges, the decoder output is masked using binary segmentation masks derived from the initial depth [S1].
    c. **Multi-View Coherency:** Enforce multi-view coherency by applying a total variation loss to the rendered depth across all Gaussians. This ensures that the depth field is smooth across the entire scene [S1].
    d. **Flow-Based Coherency:** Apply a flow-based loss to constrain optimization by ensuring corresponding pixels in two images have similar Gaussian positions. This uses the dense flow correspondences to align Gaussians across views [S1].
    e. **Loss Computation:** Compute the total loss $\mathcal{L}$, which is the combination of L1 and SSIM losses between the rendered image and the reference image, plus the regularization losses (single-view, multi-view, and flow-based) [S1].
    f. **Update:** Update $\Sigma, \alpha, \mathbf{x}, \mathbf{c}$ using gradient descent to minimize the total loss [S1].

**Formulas:**
*   **Rendering Equation:**
    $$ R_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}}(\mathbf{p}) = \sum_{i \in \mathcal{N}(\mathbf{p})} \mathbf{c}_i \gamma_i \prod_{j=1}^{i-1}(1-\gamma_j) $$
    where $\gamma_k = f(\Sigma_k, \alpha_k, \mathbf{x}_k, \mathbf{p})$ [S1].
*   **Optimization Objective:**
    $$ \Sigma^*, \alpha^*, \mathbf{x}^*, \mathbf{c}^* = \arg\min_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}} \sum_{\mathbf{p} \in \mathcal{P}} \mathcal{L}(R_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}}(\mathbf{p}), R(\mathbf{p})) $$
    where $\mathcal{P}$ contains all pixels of every input image, $R(\mathbf{p})$ is the reference pixel color, and $\mathcal{L}$ is the combination of L1 and SSIM losses [S1].

**Parameters:**
*   $N$: The number of sparse input images used for reconstruction. Typical value: 3 or 4 [S1].

**Edge Cases and Pitfalls:**
*   Constraining unstructured Gaussians directly in 3D space causes optimization instability; CoherentGS avoids this by mapping constraints to 2D image space [S1].
*   Monocular depth estimates are relative and lack consistency across views; the method relies on the regularized optimization to correct these inconsistencies during training [S1].
*   Occluded regions in all input images are not reconstructed by the Gaussians; this is a feature, not a bug, as it allows for subsequent inpainting using diffusion models [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Prevents overfitting in sparse settings; produces high-quality synthesized views compared to state-of-the-art sparse-view NeRF-based approaches; allows identification of fully occluded areas for inpainting; avoids blurry occluded regions common in NeRF-based methods [S1].
*   **Limitations:** Requires monocular depth and flow prediction models as pre-processing steps; does not reconstruct occluded regions without additional inpainting; limited to static scenes [S1].
*   **When to use:** Sparse novel view synthesis (e.g., 3-4 input images) in static scenes [S1].
*   **When not to use:** Dense view synthesis; dynamic scenes; scenarios where occluded regions must be reconstructed without external inpainting [S1].

## Background
CoherentGS was proposed by A. Paliwal et al. during an internship at Meta Reality Labs [S1]. The work addresses the limitations of 3D Gaussian Splatting (3DGS) when applied to extremely sparse input images, where the representation appears as a jumble of needles from novel views [S1]. It builds on the explicit representation of 3DGS [23] and integrates concepts from implicit convolutional decoders [5, 30, 37] and monocular depth prediction [60] and flow estimation [45]. The method is designed to improve upon sparse NeRF-based approaches like FreeNeRF [59] and SparseNeRF [18] by providing sharper details and structured hallucination capabilities [S1].

## Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
