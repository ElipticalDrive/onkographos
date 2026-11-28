# H-2DGS (Height-Field 2D Gaussian Splatting)
Also called: Height-Field Gaussian Splatting

## Overview
H-2DGS converts a volumetric 3D Gaussian Splatting reconstruction into a thin, geometrically sharp, and deformable cortical surface using curved height-field Gaussian primitives initialized on a shared Truncated Signed Distance Field (TSDF) zero level set [S1, unofficial]. It solves the problem of depth ambiguity and thick, geometrically inconsistent opacity bands in standard 3DGS, particularly in medical imaging where explicit surface geometry is required for navigation and analysis [S1, unofficial]. Unlike its siblings, which either expand from sparse seeds or parameterize onto a UV atlas, H-2DGS explicitly constructs a continuous surface representation by concentrating volumetric density into a TSDF and then mapping curved 2D Gaussians to that surface [S1, unofficial].

## How it works
H-2DGS operates as a three-stage pipeline: photometric bootstrap, ray-wise density concentration, and TSDF-guided surface conversion [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Multi-view images $I_v$ from training cameras.
*   Camera poses and intrinsics.
*   (Optional) Postoperative MRI-derived depth geometry for deformation [S1, unofficial].

**Outputs:**
*   A set of curved height-field Gaussian primitives $\{G_j\}$ representing the surface.
*   Each primitive contains height, color, and opacity bases [S1, unofficial].

### Data Structures
**Height-Field Primitive:**
Unlike standard 3D Gaussians defined by mean $\mu$, covariance $\Sigma$, opacity $\alpha$, and spherical harmonics $c$, H-2DGS primitives are defined on a 2D manifold (the TSDF zero level set).
*   **Position:** 3D coordinates on the TSDF surface.
*   **Orientation:** Normal vector derived from the TSDF gradient.
*   **Bases:** Separate parameters for height (geometry deviation from the TSDF), color (appearance), and opacity [S1, unofficial].

### Numbered Steps

**Stage I: Photometric Bootstrap**
1.  Train a conventional 3DGS model using adaptive densification [S1, unofficial].
2.  Optimize using the standard photometric loss $\mathcal{L}_{\text{photo}}$ to establish appearance and scene support without surface constraints [S1, unofficial].
    $$ \mathcal{L}_{\text{photo}} = (1 - \lambda) \|\hat{I}_v - I_v\|_1 + \lambda(1 - \text{SSIM}(\hat{I}_v, I_v)) $$
    where $\lambda = 0.2$, $\hat{I}_v$ is the rendered image, and $I_v$ is the input image [S1, unofficial].

**Stage II: Ray-wise Density Concentration**
1.  Initialize with the Stage I model after a 5,000 iteration warm start [S1, unofficial].
2.  Optimize for 1,000 iterations using a combined loss $\mathcal{L}_{\text{stage2}}$ to concentrate density along rays [S1, unofficial].
3.  For each camera ray $r(t) = o + td$, compute 1D Gaussian contributions:
    $$ \sigma_i(t) = \beta_i \exp\left[-\frac{1}{2}a_i(t - m_i)^2\right] $$
    where $\beta_i$ is amplitude, $a_i$ is inverse variance, and $m_i$ is mean position along the ray [S1, unofficial].
4.  Compute normalized density $p_m$ and cumulative distribution $F_m$ at sample $t_m$:
    $$ p_m = \frac{\sigma(t_m)}{\sum_n \sigma(t_n) + \epsilon}, \quad F_m = \sum_{n=1}^m p_n $$
5.  Approximate quantile $q$ using differentiable quantile approximation:
    $$ w_m(q) = \frac{\exp(-\tau_q|F_m - q|)}{\sum_n \exp(-\tau_q|F_n - q|)}, \quad t_q = \sum_m w_m(q)t_m $$
    where $\tau_q$ is the temperature parameter [S1, unofficial].
6.  Calculate effective thickness $\Delta t = t_{q_{\text{high}}} - t_{q_{\text{low}}}$ with $q_{\text{low}} = 0.01$ and $q_{\text{high}} = 0.99$ [S1, unofficial].
7.  Compute thickness loss:
    $$ \mathcal{L}_{\text{thick}}(\mathbf{r}) = \frac{\max(0, \Delta t - \Delta t^*)}{E} $$
    where $\Delta t^*$ is the target thickness (set to zero normalized thickness) and $E$ is the scene extent [S1, unofficial].
8.  Total loss:
    $$ \mathcal{L}_{\text{stage2}} = \mathcal{L}_{\text{photo}} + \lambda_{\text{thick}} \mathbb{E}_{\mathbf{r}}[\mathcal{L}_{\text{thick}}] $$
    where $\lambda_{\text{thick}} = 5.0$ [S1, unofficial].

**Stage III: TSDF-Guided Surface Conversion**
1.  Render depth maps from all training cameras using the Stage II model [S1, unofficial].
2.  Fuse these depth observations into a Truncated Signed Distance Field (TSDF) [S1, unofficial].
3.  For a voxel center $\mathbf{x}$ and view $v$, compute the signed distance observation:
    $$ \phi_v(\mathbf{x}) = \text{clip}\left(\frac{\hat{D}_v(\pi_v(\mathbf{x})) - d_v(\mathbf{x})}{\mu}, -1, 1\right) $$
    where $\hat{D}_v$ is the rendered depth, $d_v(\mathbf{x})$ is the projected depth, and $\mu$ is the truncation margin [S1, unofficial].
4.  Fuse observations into the global TSDF $\Phi(\mathbf{x})$ using view weights $w_v(\mathbf{x})$ [S1, unofficial].
5.  Initialize curved height-field primitives on the TSDF zero level set $\Phi(\mathbf{x}) = 0$ [S1, unofficial].
6.  Apply neighbor-transition optimization to coordinate overlapping patches and ensure locally continuous geometry [S1, unofficial].

**Postoperative Deformation (Optional)**
1.  Deform the explicit height-field surface toward postoperative MRI-derived depth geometry [S1, unofficial].
2.  Use tracked-color supervision and alternating geometry-appearance updates to preserve photorealistic texture [S1, unofficial].

### Parameters
*   $\lambda$ (SSIM weight): 0.2 [S1, unofficial].
*   $\lambda_{\text{thick}}$ (Thickness loss weight): 5.0 [S1, unofficial].
*   $q_{\text{low}}$: 0.01 [S1, unofficial].
*   $q_{\text{high}}$: 0.99 [S1, unofficial].
*   $\Delta t^*$ (Target thickness): Zero normalized thickness [S1, unofficial].
*   Stage II iterations: 1,000 [S1, unofficial].
*   Warm start iterations: 5,000 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Depth Ambiguity:** Standard 3DGS may distribute opacity through a thick band under limited view coverage; Stage II is required to resolve this [S1, unofficial].
*   **Floating Density:** Stage II may leave residual floating density; TSDF fusion in Stage III aggregates evidence to create a clean surface [S1, unofficial].
*   **Postoperative Changes:** Brain motion and tissue removal alter anatomy; the deformation stage must preserve texture while aligning to new geometry [S1, unofficial].

## Operating characteristics
*   **Representation Size:** Compact, achieving 7.97 MB across five real neurosurgical cases [S1, unofficial].
*   **Performance:** Interactive rendering at 29–42 ms per frame [S1, unofficial].
*   **Strengths:** Produces sharp, geometrically consistent surfaces; supports postoperative deformation; integrates with tractography and MRI anatomy [S1, unofficial].
*   **Limitations:** Requires multi-stage training; dependent on TSDF fusion quality; specific to surface reconstruction tasks [S1, unofficial].

## Background
H-2DGS was introduced to address the limitations of conventional 3D Gaussian Splatting in neurosurgical visualization, where explicit surface geometry is critical for navigation and analysis [S1, unofficial]. It builds on [3D Gaussian Splatting](3410-3d-gaussian-splatting-mvp.md) by incorporating geometric supervision and representation modification [21351-geometry-and-representation-adaptation.md]. The method was developed for real neurosurgical cases to provide low multi-view depth inconsistency and a compact representation [S1, unofficial].

## Sources
[S1] H-2DGS: Height-Field Gaussian Splatting for Sharp and Deformable Neurosurgical Surface Reconstruction, https://arxiv.org/abs/2610.30910, fetched 2026-10-26 (unofficial)
