# Appearance Feature Initialization and Training Schedule

Also called: Appearance feature initialization, Training schedule with appearance features

## Overview
This method defines the temporal schedule for introducing and initializing appearance features (view-dependent color) in 3D Gaussian Splatting, specifically within the context of optimized minimal implementations. It addresses the problem of stable convergence for view-dependent rendering by decoupling the initialization of static and view-dependent components. It sets apart from standard 3D Gaussian Splatting by delaying the introduction of appearance features until iteration 15K and initializing them using trained Spherical Harmonics (SH) DC coefficients, rather than random or RGB-based initialization [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D Gaussian scene model with positions, covariances, opacities, and SH coefficients (trained up to iteration 15K).
- Output: A Gaussian model with initialized appearance feature vectors (static and view-dependent) ready for continued training.

**Data Structures:**
- **Appearance Feature Vector:** A vector of dimension $D$. In the cited implementation, $D=3$ [S1].
- **Static Appearance Features:** Stored per Gaussian. Initialized from SH DC coefficients.
- **View-Dependent Appearance Features:** Stored per Gaussian. Initialized as zero vectors.

**Numbered Steps:**
1. **Initial Training (Iteration 0–14,999):** Train scale and rotation parameters from the initial iteration. Do not train appearance features. Train SH coefficients (typically up to order 3 or as configured) [S1].
2. **Appearance Feature Introduction (Iteration 15,000):**
   - Initialize **static appearance features** for each Gaussian using the current SH DC coefficients (zeroth-order term) trained up to iteration 15K [S1].
   - Initialize **view-dependent appearance features** for each Gaussian as zero vectors [S1].
3. **Simplification (Iteration 20,000):** Run the simplification process (pruning/densification) incorporating local distinctiveness (LD) scoring with factor $\lambda=2$ [S1].
4. **Sub-Vector Quantization (Iteration 29,000–30,000):** Apply Sub-Vector Quantization (SVQ) to per-Gaussian features. K-means clustering is performed once at the start of this phase. The assigned indices are fixed, and only the codebooks are optimized for the remaining 1K iterations [S1].

**Formulas and Parameters:**
- **Local Distinctiveness Scoring Factor:** $\lambda = 2$ [S1].
- **Appearance Feature Dimension:** $D = 3$ [S1].
- **SVQ Bit Allocations:**
  - Scale: length 1, 26 codes per sub-vector [S1].
  - Rotation: length 2, 29 codes per sub-vector [S1].
  - Appearance features: length 2, 210 codes per sub-vector [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_appearance_features(gaussians, sh_coeffs, iteration):
    """
    Initialize appearance features at iteration 15K.
    gaussians: List of Gaussian objects with attributes .static_appearance, .view_dependent_appearance
    sh_coeffs: Trained SH coefficients up to iteration 15K
    """
    if iteration == 15000:
        for gaussian in gaussians:
            # Extract DC coefficient (zeroth-order) from SH
            # Assuming SH coefficients are stored in a format where index 0 is DC
            dc_color = sh_coeffs.get_dc_color(gaussian.id)
            
            # Initialize static appearance features with DC coefficients
            # Dimension D=3 as per [S1]
            gaussian.static_appearance = dc_color.copy()
            
            # Initialize view-dependent appearance features as zero vectors
            # Dimension D=3 as per [S1]
            gaussian.view_dependent_appearance = np.zeros(3)
            
    return gaussians

def apply_svq(gaussians, iteration, kmeans_indices, codebooks):
    """
    Apply Sub-Vector Quantization during the last 1K iterations (29K-30K).
    """
    if 29000 <= iteration <= 30000:
        # K-means indices are fixed from previous step
        # Only optimize codebooks
        for gaussian in gaussians:
            # Quantize scale (length 1)
            gaussian.scale_quantized = quantize_subvector(gaussian.scale, codebooks['scale'], kmeans_indices['scale'])
            
            # Quantize rotation (length 2)
            gaussian.rotation_quantized = quantize_subvector(gaussian.rotation, codebooks['rotation'], kmeans_indices['rotation'])
            
            # Quantize appearance features (length 2)
            gaussian.appearance_quantized = quantize_subvector(gaussian.appearance, codebooks['appearance'], kmeans_indices['appearance'])
            
    return gaussians
```

**Edge Cases and Pitfalls:**
- **Precision:** SVQ codebooks are optimized while indices are fixed. Ensure gradient flow is maintained only through codebooks during iterations 29K–30K [S1].
- **Boundaries:** The transition at iteration 15K must ensure that SH coefficients are fully trained up to that point to provide a meaningful DC initialization [S1].

## Subtopics

### Appearance Embedding
Associates each training image with a unique embedding vector to model appearance changes such as lighting and shadows [S2, unofficial]. The embedding dimension is set to 8 [S2, unofficial]. During validation, the mean of training appearance embeddings is used for all validation images to ensure consistent representation [S2, unofficial]. This variant is used in NeRF-based methods (e.g., SCALAR-NeRF) rather than Gaussian Splatting [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The method adds negligible overhead to the standard 3D Gaussian Splatting training loop, except for the one-time K-means clustering at iteration 29K [S1].
- **Memory Complexity:** Introduces storage for appearance feature vectors (dimension $D=3$) per Gaussian [S1]. SVQ reduces memory footprint by storing codebooks and indices instead of full feature vectors [S1].
- **Measured Performance:** Experiments conducted on NVIDIA RTX 4090 using the Mini-Splatting framework [S1]. Total training time is 30K iterations [S1].
- **Strengths:** Stabilizes view-dependent rendering by delaying appearance feature training; reduces memory usage via SVQ [S1].
- **Limitations:** Requires careful scheduling of hyperparameters (e.g., $\lambda=2$, iteration counts) [S1].

## Background
Originated in the context of optimized minimal 3D Gaussian Splatting implementations [S1]. The approach builds on [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md) by leveraging trained SH coefficients for static appearance initialization [S1]. The appearance embedding variant originates from NeRF-W and SCALAR-NeRF [S2, unofficial].

## Sources
[S1] Optimized Minimal 3D Gaussian Splatting, https://doi.org/10.52202/085713-4534, fetched 2026-10-30
[S2] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
