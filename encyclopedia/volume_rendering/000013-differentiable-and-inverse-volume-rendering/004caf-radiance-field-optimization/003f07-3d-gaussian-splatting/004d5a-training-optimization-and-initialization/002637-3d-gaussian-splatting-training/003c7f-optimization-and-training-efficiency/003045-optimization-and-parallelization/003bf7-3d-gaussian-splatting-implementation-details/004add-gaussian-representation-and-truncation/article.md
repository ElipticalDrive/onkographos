# Gaussian Representation and Truncation

Also called: Vertex Pruning, Mixed-Primitive Truncation, Atomized Proliferation, Hard Gaussian Splatting (POG variant)

## Overview
Gaussian Representation and Truncation refers to a set of optimization and initialization strategies in 3D Gaussian Splatting (3DGS) that refine the spatial representation of Gaussians during training. It addresses the problem of redundant or poorly shaped primitives (e.g., overly large, collinear, or noisy Gaussians) that degrade rendering quality, geometric accuracy, and storage efficiency. Unlike standard 3DGS which only performs primitive-level opacity pruning, this method introduces vertex-level pruning for mixed primitives, atomized proliferation for geometric precision, and hard Gaussian growth for challenging areas. It sets itself apart by actively modifying the topology and size distribution of Gaussians beyond simple opacity culling, often integrating geometric constraints or specific growth heuristics.

## How it works
The method encompasses several distinct strategies depending on the specific variant (Mixed-Primitive, AtomGS, or HGS/POG). Below are the implementations for the primary approaches cited in the sources.

### Mixed-Primitive Vertex Pruning
This strategy, used in frameworks like MP-GS [S1, unofficial], operates on primitives with varying numbers of vertices (Gaussian triangles, lines, and ellipses). It triggers vertex-level pruning whenever primitive-level pruning is performed.

**Inputs:**
- A set of Gaussian primitives, each defined by vertices $\mu_i$ and associated covariance matrices.
- Thresholds: $\omega_{dist}$ (default 0.5) and $\omega_{pear}$ (default 0.9) [S1, unofficial].

**Steps:**
1. **Trigger:** Perform vertex-level pruning concurrently with standard primitive-level opacity pruning.
2. **Gaussian Triangle Pruning:**
   - Calculate the maximum pairwise L2 distance between the three vertices $\mu_1, \mu_2, \mu_3$.
   - If $\max(\|\mu_i - \mu_j\|_2) < \omega_{dist}$, the vertices are considered "close."
   - Prune vertices $\mu_2$ and $\mu_3$.
   - Convert the primitive into a Gaussian ellipse centered at $\mu_1$.
3. **Gaussian Triangle to Line Conversion:**
   - If vertices are not close, compute the Pearson correlation coefficient $r$ from the coordinates of the three vertices.
   - If $|r| > \omega_{pear}$, the vertices are considered "nearly collinear."
   - Convert the Gaussian triangle into a Gaussian line.
4. **Gaussian Line Pruning:**
   - For a Gaussian line primitive with vertices $\mu_1, \mu_2$, calculate the L2 distance $\|\mu_1 - \mu_2\|_2$.
   - If $\|\mu_1 - \mu_2\|_2 < \omega_{dist}$, the vertices are "close."
   - Prune vertex $\mu_2$.
   - Reduce the primitive to a Gaussian ellipse centered at $\mu_1$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_pruning(primitive, omega_dist=0.5, omega_pear=0.9):
    """
    Applies vertex-level pruning to a mixed-primitive Gaussian.
    primitive: object with type ('triangle', 'line', 'ellipse') and vertices (list of vectors).
    """
    if primitive.type == 'triangle':
        vertices = primitive.vertices # [mu1, mu2, mu3]
        
        # Check if vertices are close
        max_dist = 0
        for i in range(3):
            for j in range(i + 1, 3):
                dist = l2_distance(vertices[i], vertices[j])
                if dist > max_dist:
                    max_dist = dist
        
        if max_dist < omega_dist:
            # Prune mu2 and mu3, convert to ellipse at mu1
            primitive.type = 'ellipse'
            primitive.vertices = [vertices[0]]
            return

        # Check if vertices are collinear
        # Compute Pearson correlation coefficient from vertex coordinates
        # Flatten vertices to 1D array for correlation calculation
        coords = flatten_vertices(vertices)
        pearson_r = compute_pearson_correlation(coords)
        
        if abs(pearson_r) > omega_pear:
            # Convert triangle to line
            primitive.type = 'line'
            # Keep all vertices for the line, but change type
            return

    elif primitive.type == 'line':
        vertices = primitive.vertices # [mu1, mu2]
        dist = l2_distance(vertices[0], vertices[1])
        
        if dist < omega_dist:
            # Prune mu2, reduce to ellipse at mu1
            primitive.type = 'ellipse'
            primitive.vertices = [vertices[0]]
            return
```

### Atomized Proliferation (AtomGS)
This strategy enhances geometric precision by constraining Gaussian sizes and guiding optimization with edge-aware losses [S3, unofficial].

**Inputs:**
- Current set of Gaussian ellipsoids.
- 2D edge detector weights.
- Curvature map.

**Steps:**
1. **Atomized Proliferation:**
   - Identify smaller anisotropic Gaussians.
   - Constrain them into uniformly-sized "Atom Gaussians."
   - Apply a progressive split schedule to densify these atoms, emphasizing coverage of fine scene details.
   - Retain larger anisotropic Gaussians to represent background or geometric areas with few features.
2. **Geometry-Guided Optimization:**
   - Compute an Edge-Aware Normal Loss.
   - Integrate weights from a 2D edge detector into the curvature map.
   - Apply stricter constraints on Gaussian positioning for flat surfaces (aligned with normals) while allowing flexibility for irregular shapes.
3. **Pruning and Merging:**
   - During the pruning phase, merge Gaussians covering large, less detailed surfaces.
   - This results in a similar or reduced number of Gaussians compared to original 3DGS while retaining quality.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atomized_proliferation_step(gaussians, edge_weights, curvature_map):
    """
    Implements AtomGS proliferation and optimization.
    """
    # 1. Constrain small Gaussians to Atom Gaussians
    for g in gaussians:
        if is_small_anisotropic(g):
            g.size = atom_size # Uniform size
            g.type = 'atom'
            
    # 2. Progressive Split for Atoms
    for g in gaussians:
        if g.type == 'atom':
            if needs_densification(g, curvature_map):
                split_gaussian_into_atoms(g)

    # 3. Geometry-Guided Optimization
    for g in gaussians:
        # Compute Edge-Aware Normal Loss
        # Integrate edge detector weights into curvature map
        loss = compute_edge_aware_normal_loss(g, edge_weights, curvature_map)
        apply_gradient(g, loss)

    # 4. Pruning/Merging
    for g in gaussians:
        if covers_large_flat_surface(g):
            merge_with_neighbors(g)
```

### Potentially Over-large Gaussians (POG) / Hard Gaussians
This strategy addresses blurring and needle-like artifacts by identifying and growing "hard" Gaussians [S2].

**Inputs:**
- Multi-view significant positional gradients.
- Rendering errors.
- Thresholds for identifying "potentially over-large" vs "hard" Gaussians.

**Steps:**
1. **Identification:**
   - Locate "hard Gaussians" from "potentially over-large Gaussians" based on multi-view gradients and rendering errors.
   - POG (Eq. 8 in source) identifies potentially over-large Gaussians.
   - HG (Eq. 9 in source) identifies hard Gaussians.
2. **Growth:**
   - Grow and optimize these identified Gaussians to resolve artifacts in challenging areas.
   - Note: POG can overgrow already learned areas, leading to redundant Gaussians and PSNR degradation, whereas HG avoids this [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_gaussian_growth(gaussians, gradients, errors):
    """
    Implements Hard Gaussian Splatting growth.
    """
    hard_gaussians = []
    for g in gaussians:
        # Check if Gaussian is a 'hard' Gaussian based on gradients and errors
        if is_hard_gaussian(g, gradients, errors):
            hard_gaussians.append(g)
            
    # Grow and optimize hard Gaussians
    for g in hard_gaussians:
        grow_gaussian(g)
        optimize_gaussian(g)
```

## Subtopics

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Replaces the standard Gaussian radial basis function with an inverse multiquadric function ($1/(ellipse + 1)$) to model long-tailed distributions. Distinguished by its specific algebraic form and a fixed correction factor applied to the covariance matrix. Choose when standard Gaussians fail to capture heavy-tailed radiance field features.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Decouples spatial truncation threshold from opacity. By checking spatial response before multiplying by opacity, it allows aggressive truncation (e.g., 1σ or 2σ) and enables gradient flow for low-opacity Gaussians that would otherwise be culled. Choose when low-opacity Gaussians are not receiving gradients or contributing to geometry.

### [Atomized Proliferation](12924-atomized-proliferation.md)
A training optimization that constrains fine-detail Gaussians into isotropic "Atom Gaussians" and prioritizes their proliferation. Uses a warm-up phase for aggressive splitting followed by merging of large surfaces. Choose when superior mesh quality and geometric precision are required over raw rendering speed.

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
*Note: This is a minor variant described in the notes, not a separate article.*
POG identifies potentially over-large Gaussians to grow them. However, it can overgrow learned areas, causing redundant Gaussians and PSNR degradation (Tanks&Temples: PSNR 26.26, SSIM 0.879, LPIPS 0.121) compared to Hard Gaussians (HG) which avoid this issue (PSNR 26.36, SSIM 0.880, LPIPS 0.126).

### [Atomized Proliferation](12924-atomized-proliferation.md)
*Note: This is a minor variant described in the notes, not a separate article.*
AtomGS uses Atomized Proliferation and Geometry-Guided Optimization. It constrains small anisotropic Gaussians into uniform Atom Gaussians and retains large ones for backgrounds. It integrates Edge-Aware Normal Loss using 2D edge detector weights into the curvature map. Unlike standard 3DGS, it provides precise guidance for 3D geometry optimization, balancing visual and geometric accuracy.

## Operating characteristics
- **Storage Efficiency:** Vertex pruning in mixed-primitive frameworks reduces average storage size for vertex coordinates by 24% compared to variants without pruning [S1, unofficial].
- **Rendering Quality:**
  - AtomGS achieves state-of-the-art rendering quality and competitive geometry reconstruction accuracy with improved training speed [S3, unofficial].
  - POG degrades PSNR significantly compared to HG due to redundant Gaussians in learned areas [S2].
  - HG (Hard Gaussians) improves LPIPS and PSNR over POG by avoiding overgrowth [S2].
- **Geometric Accuracy:** AtomGS provides better 3D geometry optimization than standard 3DGS by constraining Gaussian placement [S3, unofficial]. Mixed-primitive methods with vertex pruning show better Chamfer distance (0.46 with initialization vs 0.56 without) [S1, unofficial].
- **Limitations:** POG is susceptible to redundant Gaussians and PSNR degradation [S2]. AtomGS requires additional computational steps for edge detection and curvature map integration [S3, unofficial].

## Background
- **Mixed-Primitive Vertex Pruning:** Introduced in "A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction" [S1, unofficial]. It extends standard 3DGS by allowing primitives with varying numbers of vertices (triangles, lines) and introduces vertex-level pruning to refine the representation.
- **AtomGS:** Introduced in "AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field" [S3, unofficial]. It aims to enhance rendering quality and geometric precision by atomizing Gaussians and guiding optimization with edge-aware losses.
- **Hard Gaussian Splatting (HGS/POG):** Introduced in "Pushing Rendering Boundaries: Hard Gaussian Splatting" [S2]. It addresses blurring and needle-like artifacts by growing hard Gaussians identified from multi-view gradients and rendering errors.

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S2] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S3] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
