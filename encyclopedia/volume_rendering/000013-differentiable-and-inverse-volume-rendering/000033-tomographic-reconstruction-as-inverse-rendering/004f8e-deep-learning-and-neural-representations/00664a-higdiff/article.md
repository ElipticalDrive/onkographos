# HiGDiff

Also called: Feed-Forward Hierarchical Gaussian Diffusion for Extreme CT Reconstruction

## Overview
HiGDiff is a feed-forward hierarchical Gaussian diffusion framework for constrained Computed Tomography (CT) reconstruction that decomposes the recovery process spatially and hierarchically from global structure to local detail [S1, unofficial]. It addresses the problem of joint degradations—where sparse-view sampling aliasing, limited-angle directional incompleteness, and low-dose photon noise are coupled—by using separate Gaussian banks for attenuation geometry and residual boundaries, conditioned on acquisition metadata [S1, unofficial]. Unlike sibling methods that use a single shared representation for all features, HiGDiff establishes an explicit functional hierarchy where a structure diffusion stage recovers global geometry and conditions a subsequent detail diffusion stage for tissue transitions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Degraded projection data (sinograms) affected by sparse views, limited angles, or low dose; acquisition metadata including view density, angular span, and dose levels [S1, unofficial].
*   **Outputs:** A volumetric attenuation field represented by two banks of learnable 3D Gaussian primitives, refined by a residual module [S1, unofficial].

**Data Structures:**
1.  **Foreground Capacity Field:** A learnable scalar field estimating the utility of allocating Gaussian primitives to specific spatial regions [S1, unofficial].
2.  **Anatomical Anchors:** Landmark-free features used to guide the allocation of primitives to informative regions [S1, unofficial].
3.  **Structure Gaussian Bank:** A set of 3D Gaussian primitives representing global attenuation geometry [S1, unofficial].
4.  **Detail Gaussian Bank:** A set of 3D Gaussian primitives representing residual boundaries and tissue transitions [S1, unofficial].
5.  **Gradient-Isolated Residual Module:** A network module that compensates for residual errors under a fixed primitive budget [S1, unofficial].

**Method Steps:**
1.  **Metadata Conditioning:** The reconstruction process is conditioned on the input view density, angular span, and dose metadata to adapt to the specific degradation type (sparse, limited, low-dose, or joint) [S1, unofficial].
2.  **Structure Diffusion:** A diffusion process operates on the Structure Gaussian Bank to recover the global attenuation geometry [S1, unofficial].
3.  **Capacity Allocation:** The Foreground Capacity Field and Anatomical Anchors are used to allocate learnable Gaussian primitives to anatomically informative regions, modulating their denoising features [S1, unofficial].
4.  **Detail Diffusion:** The learned representation from the Structure Gaussian Bank conditions a second diffusion stage on the Detail Gaussian Bank to recover residual boundaries and tissue transitions [S1, unofficial]. This instantiates multi-stage diffusion dependency directly in the space of explicit CT Gaussian parameters [S1, unofficial].
5.  **Rendering:** The resulting Gaussian banks are rendered into attenuation fields [S1, unofficial].
6.  **Residual Refinement:** A gradient-isolated residual module refines the rendered fields to compensate for residual errors and preserve the Gaussian reconstruction as the output backbone [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def HiGDiff(degraded_projections, metadata):
    """
    metadata: dict containing 'view_density', 'angular_span', 'dose'
    """
    # Step 1: Condition on acquisition metadata
    # [G] Metadata is typically embedded and added to the diffusion timestep embeddings
    conditioning = embed_metadata(metadata)
    
    # Step 2: Structure Diffusion Stage
    # Initialize structure bank with noise or prior
    structure_bank = initialize_gaussian_bank()
    
    # Run diffusion to recover global attenuation geometry
    # [G] Standard DDPM/DDIM sampling loop conditioned on structure prior
    structure_bank = diffusion_step_structure(structure_bank, conditioning)
    
    # Step 3: Foreground Capacity and Allocation
    # Estimate utility of allocating primitives
    capacity_field = compute_foreground_capacity(structure_bank)
    anchors = compute_anatomical_anchors(structure_bank)
    
    # Allocate primitives to informative regions
    # [G] This modulates the features of the primitives for the next stage
    allocated_features = modulate_features(structure_bank, capacity_field, anchors)
    
    # Step 4: Detail Diffusion Stage
    # Initialize detail bank
    detail_bank = initialize_gaussian_bank()
    
    # Condition detail diffusion on the learned structure representation
    # This establishes the explicit functional hierarchy
    detail_conditioning = combine(structure_bank, allocated_features)
    
    # Run diffusion to recover residual boundaries and tissue transitions
    detail_bank = diffusion_step_detail(detail_bank, detail_conditioning)
    
    # Step 5: Rendering
    # Render both banks into attenuation fields
    structure_field = render_gaussians(structure_bank)
    detail_field = render_gaussians(detail_bank)
    
    # Step 6: Residual Refinement
    # Refine the combined field using a gradient-isolated module
    # [G] The module likely operates on the gradient space to isolate artifacts
    refined_field = gradient_isolated_refinement(structure_field, detail_field)
    
    return refined_field
```

**Parameters:**
*   **Primitive Budget:** Fixed number of Gaussian primitives per bank [S1, unofficial].
*   **Diffusion Steps:** Number of denoising steps for structure and detail stages (Not given in the sources).
*   **Learning Rate:** For training the Gaussian parameters and capacity field (Not given in the sources).

**Edge Cases and Pitfalls:**
*   **Joint Degradation:** The method is designed to handle the coupling of sampling aliasing, directional incompleteness, and photon noise, which obscures both long-range anatomy and local tissue transitions [S1, unofficial].
*   **Primitive Budget Constraints:** The gradient-isolated module is necessary to compensate for residual errors under a fixed primitive budget, preventing the loss of detail when the bank size is limited [S1, unofficial].

## Operating characteristics
*   **Performance:** On the LDCT-PD dataset, HiGDiff improves macro-average PSNR by 5.81 dB and SSIM by 0.113 over the strongest metric-specific baselines [S1, unofficial]. It achieves the best macro-average SSIM on all three tested datasets and the best macro-average PSNR on both 3D datasets [S1, unofficial].
*   **Complexity:** The method is feed-forward after training, implying inference speed is determined by the diffusion sampling steps and Gaussian rendering cost [S1, unofficial].
*   **Strengths:** Explicitly handles joint degradations; separates global structure from local detail to prevent ambiguity; uses explicit Gaussian parameters for multi-stage diffusion dependency [S1, unofficial].
*   **Limitations:** Requires training of two diffusion stages and a capacity field; performance depends on the accuracy of the foreground capacity field in allocating primitives [S1, unofficial].
*   **When to use:** Severe constrained CT acquisition with sparse views, limited angles, or low dose, particularly when these degradations are present jointly [S1, unofficial].

## Background
HiGDiff was proposed by Yang et al. in a preprint titled "Feed-Forward Hierarchical Gaussian Diffusion for Extreme CT Reconstruction" [S1, unofficial]. It addresses gaps in existing Gaussian CT methods, which typically use one bank for both global geometry and residual detail, leaving the functional hierarchy implicit under joint degradation [S1, unofficial]. The work builds on the trend of using explicit Gaussian representations for CT and multi-stage diffusion models, but instantiates the dependency directly in the space of explicit CT Gaussian parameters rather than dense image/volume states [S1, unofficial]. Code and experimental configurations are openly available at github.com/Bean-Young/HiGDiff [S1, unofficial].

## Sources
[S1] Feed-Forward Hierarchical Gaussian Diffusion for Extreme CT Reconstruction, https://doi.org/10.48550/arxiv.2608.15343, fetched 2026-10-26 (unofficial)
