# NeRF-SH

Also called: NeRF with Spherical Harmonics, PlenOctrees, modified NeRF that predicts spherical harmonic coefficients

## Overview
NeRF-SH is a variant of Neural Radiance Fields that replaces the direct RGB output with spherical harmonic (SH) coefficients, allowing the neural network to predict only view-independent density and appearance weights while deferring view-dependent color calculation to a closed-form basis evaluation [S1, unofficial][S2]. This factorization eliminates the viewing direction as a network input, enabling the representation to be stored in a sparse voxel-based **PlenOctree** structure, which accelerates rendering by more than 3000 times compared to the original NeRF [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x} \in \mathbb{R}^3$. The viewing direction $\mathbf{d}$ is **not** an input to the network during training or storage [S1, unofficial][S2].
*   **Output:** Density $\sigma \in \mathbb{R}$ and a vector of spherical harmonic coefficients $\mathbf{k}$ [S1, unofficial].

**Data Structures:**
The method uses a **PlenOctree**, a sparse octree where each leaf node stores the density $\sigma$ and the SH coefficients $\mathbf{k}$ for the volume contained within that leaf [S1, unofficial]. A sparsity prior is introduced during training to encourage empty or uniform regions to be pruned, improving memory efficiency [S1, unofficial].

**Spherical Harmonics Basis:**
Spherical Harmonics (SH) are used as a standard basis for functions defined on the surface of a sphere $S^2$ [S1, unofficial]. The SH basis functions $Y_\ell^m(\theta, \phi)$ depend on the viewing direction defined by polar angle $\theta$ and azimuthal angle $\phi$ [S1, unofficial].

**Formulas:**
The network $f$ maps position $\mathbf{x}$ to coefficients $\mathbf{k}$ and density $\sigma$:
$$ f(\mathbf{x}) = (\mathbf{k}, \sigma) $$
where $\mathbf{k} = (k_\ell^m)_{\ell: 0 \leq \ell \leq \ell_{\max}}$ [S1, unofficial].

Each coefficient $k_\ell^m \in \mathbb{R}^3$ corresponds to the RGB color components [S1, unofficial]. The view-dependent color $\mathbf{c}$ at point $\mathbf{x}$ for viewing direction $\mathbf{d}$ (with angles $\theta, \phi$) is calculated by summing the weighted SH bases:
$$ \mathbf{c}(\mathbf{x}, \mathbf{d}) = \sum_{\ell=0}^{\ell_{\max}} \sum_{m=-\ell}^{\ell} k_\ell^m \cdot Y_\ell^m(\theta, \phi) $$
where $k_\ell^m$ are the RGB coefficients predicted by the network and $Y_\ell^m$ are the SH basis functions evaluated at the ray direction [S1, unofficial].

**Parameters:**
*   $\ell_{\max}$: The maximum degree of the spherical harmonics. The sources do not specify the exact value used in the primary experiments, but note that higher degrees capture more view-dependent detail [S1, unofficial].
*   Sparsity Prior: A regularization term applied during training to promote sparsity in the octree [S1, unofficial].

**Steps:**
1.  **Network Definition:** Define a multi-layer perceptron (MLP) $f$ that takes spatial coordinates $\mathbf{x}$ as input and outputs density $\sigma$ and SH coefficients $\mathbf{k}$ [S1, unofficial][S2].
2.  **Training:** Train the network using standard NeRF volume rendering optimization against input images. The loss is computed by rendering rays using the SH-based color calculation [S1, unofficial].
    *   For each ray, sample points $\mathbf{x}_i$.
    *   Query $f(\mathbf{x}_i)$ to get $\sigma_i$ and $\mathbf{k}_i$.
    *   Compute color $\mathbf{c}_i$ using the SH formula based on the ray direction [S1, unofficial].
    *   Composite colors and opacities using standard volume rendering [S1, unofficial].
3.  **PlenOctree Construction:** Densely sample the trained network throughout the volume around the target object to tabulate $\sigma$ and $\mathbf{k}$ values into the leaves of an octree structure [S1, unofficial].
4.  **Optional Optimization:** The values stored in the PlenOctree can be further optimized directly using training images to improve quality, as the rendering procedure remains differentiable [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Projection Quality Loss:** Directly projecting a standard trained NeRF (which outputs RGB) into SH coefficients via Monte Carlo estimation (sampling random directions and multiplying by SH component values) is inefficient and lossy. It takes several hours (e.g., ~2 hours for 10,000 samples per point) and results in a quality loss of approximately 2 dB (PSNR 29.21 vs. 31.02 for the direct NeRF-SH method) [S1, unofficial].
*   **Training Time:** Training NeRF-SH remains time-intensive, typically requiring one or two days for convergence [S2].
*   **Basis Limitations:** While Spherical Gaussians (SG) were experimented with, SHs were found to perform better for this specific volumetric use case [S1, unofficial].

## Operating characteristics
*   **Rendering Speed:** Accelerates rendering performance of the original NeRF by more than 3000 times [S2]. Achieves frame rates above 150 fps for 800x800 images [S2].
*   **Training Speed:** Training speed is similar to, but slightly faster than, standard NeRF (by about 10%) [S1, unofficial].
*   **Image Quality:** Produces images equal to or better in quality than the original NeRF after PlenOctree optimization [S1, unofficial].
*   **Memory:** The use of a sparse octree (PlenOctree) with a sparsity prior improves memory efficiency compared to dense voxel grids, though caching radiance values in explicit structures is less compact than the implicit NeRF representation [S1, unofficial][S2].
*   **Strengths:** Real-time rendering capability; view-independent network input simplifies the MLP; efficient storage of view-dependent effects.
*   **Limitations:** Training is still slow (1-2 days); requires post-training conversion to explicit structure; limited by the basis functions' ability to represent high-frequency view-dependent effects (though SHs are generally sufficient for many scenes) [S1, unofficial][S2].

## Background
NeRF-SH was introduced by Yu et al. in the paper "PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs)" [S1, unofficial]. The method addresses the slow rendering speed of original NeRFs by factorizing appearance into view-independent coefficients (stored in a PlenOctree) and view-dependent basis functions (SHs) [S2]. It builds on the original NeRF framework [1185-nerf-neural-radiance-fields.md] but modifies the output layer and introduces a sparse voxel structure [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
