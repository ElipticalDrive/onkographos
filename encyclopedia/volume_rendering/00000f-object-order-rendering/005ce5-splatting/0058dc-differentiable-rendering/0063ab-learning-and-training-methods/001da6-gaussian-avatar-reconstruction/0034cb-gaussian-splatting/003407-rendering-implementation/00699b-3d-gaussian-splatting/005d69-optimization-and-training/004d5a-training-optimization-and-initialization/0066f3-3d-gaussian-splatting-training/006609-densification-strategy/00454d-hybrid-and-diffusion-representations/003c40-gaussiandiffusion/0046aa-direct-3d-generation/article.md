# Direct 3D Generation

Also called: Feedforward 3D Gaussian Splatting Generation, Amortized 3DGS Synthesis

## Overview
Direct 3D Generation refers to methods that learn a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set [S1]. It solves the problem of high reconstruction latency and multi-view inconsistency inherent in iterative optimization by amortizing the cost over a trained network [S1]. Unlike iterative reconstruction, it naturally promotes multi-view coherence and enables near-real-time sampling at test time [S1].

## How it works
**Inputs:** Text prompts, input images, or partial 3DGS representations [S1, S2].
**Outputs:** A set of 3D Gaussian primitives defined by position, covariance (transform), color, and opacity [S1, S2].

**Data Structures:**
The method typically disentangles the 3D Gaussian Splatting (3DGS) representation into three distinct functions or latent spaces to handle the high-dimensional output:
1.  **GauPF (Gaussian Probability Function):** Models the 3D locations/geometry of the Gaussians [S2].
2.  **GauCF (Gaussian Color Function):** Models the color attributes [S2].
3.  **GauTF (Gaussian Transform Function):** Models the covariance/transform attributes [S2].

In diffusion-based variants (e.g., DiffGS), a Gaussian Variational Autoencoder (VAE) maps these functions into a regularized latent space [S2]. The latent vector $z$ represents the compressed 3DGS shape [S2].

**Steps:**
1.  **Training (VAE):** Train a Gaussian VAE to map 3DGS functions into a latent vector $z$ [S2]. The VAE learns a regularized latent space [S2].
2.  **Training (Diffusion):** Train a Latent Diffusion Model (LDM) in the latent space of the Gaussian VAE [S2].
    *   The LDM is trained to predict noise or reverse the diffusion process on latent vectors $z_k$ [S2].
    *   Conditioning inputs (text, image, partial GS) are injected into the LDM [S2].
3.  **Generation:**
    *   Start from noise $N(0,1)$ [S2].
    *   Iteratively denoise from Step $K$ to Step $0$ to obtain the clean latent vector $z_0$ [S2].
    *   Decode $z_0$ using the VAE decoder to recover GauPF, GauCF, and GauTF [S2].
4.  **Geometry Extraction:**
    *   Extract 3D Gaussian geometry from GauPF by sampling 3D locations in the 3D space with the highest Gaussian probabilities [S2].
5.  **Attribute Prediction:**
    *   Predict the Gaussian attributes (color and transform) using GauCF and GauTF respectively [S2].
6.  **Discretization (Optional):**
    *   Apply an octree-guided sampling and optimization algorithm to extract Gaussians at arbitrary numbers [S2].

**Formulas:**
*   Latent Diffusion Process: The model predicts noise for latent vectors $z_k$ derived from the VAE [S2].
*   Generation: $z_0 = \text{Denoise}(z_K, \text{Condition})$ where $z_K \sim N(0,1)$ [S2].

**Parameters:**
*   Diffusion Steps $K$: Not given in the sources.
*   Latent Dimension: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Imperfect Cross-View Consistency:** Can lead to noisy surface geometry, duplicated structures, and degraded textures, especially in occluded or textureless regions [S1].
*   **High-Dimensional Output:** The structured output space (positions, anisotropic covariance, colors, opacity) presents significant modeling challenges [S1].
*   **Data Requirements:** Requires large, curated 3D training corpora [S1].

## Subtopics

### Latent Diffusion Model (LDM) for 3DGS
A diffusion model trained in the latent space of the Gaussian VAE to generate novel 3DGS shapes conditionally or unconditionally [S2]. It differs from direct pixel-space diffusion by operating on the compressed latent representation of the Gaussian functions, enabling efficient generation of diverse 3DGS [S2].
*   **Training:** The LDM is trained to predict noise or reverse the diffusion process on latent vectors $z_k$ derived from the VAE [S2].
*   **Generation:** Starts from noise $N(0,1)$, iteratively denoises to $z_0$ (Step K to Step 0), conditioned on text, image, or partial GS if applicable [S2].

## Operating characteristics
*   **Time Complexity:** Near-real-time sampling at test time [S1].
*   **Memory Complexity:** Requires storing the trained feedforward or diffusion model and the VAE [S1, S2].
*   **Strengths:** Amortizes reconstruction cost; naturally promotes multi-view coherence; enables near-real-time sampling [S1].
*   **Limitations:** Requires large, curated 3D training corpora; challenges in maintaining controllability (e.g., disentangling geometry from appearance) while scaling [S1]; imperfect cross-view consistency can lead to noisy geometry in occluded regions [S1].
*   **When to use:** When rapid generation from text/images is required and iterative optimization is too slow [S1].
*   **When not to use:** When exact reconstruction of a specific input is required without generative artifacts [S1].

## Background
Direct 3D generation methods aim to learn a feedforward or diffusion model that outputs Gaussian fields directly from text or images, without an explicit intermediate image set [S1]. Trained on large collections of synthetic or reconstructed data, these models amortize the reconstruction cost [S1]. The approach addresses the challenges of the high-dimensional and highly structured output space of 3DGS [S1]. DiffGS introduces a Gaussian VAE and LDM framework to generate diverse 3DGS both conditionally and unconditionally [S2].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] DiffGS: Functional Gaussian Splatting Diffusion, https://arxiv.org/abs/2410.19657v2, fetched 2026-10-26
