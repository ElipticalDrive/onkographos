# Curvature-based Transfer Function

Also called: Curvature Based TF, Non-photorealistic Curvature TF, Shape Based TF (Curvature), cbTF

## Overview
The Curvature-based Transfer Function (cbTF) is a classification method that maps scalar data values and local surface curvature to optical properties (color and opacity) to distinguish features based on their geometric shape rather than intensity alone [S1][S3, unofficial]. It solves the problem of distinguishing between structures with similar intensities but different curvatures, such as the borders of small versus large features, and is primarily used for non-photorealistic rendering to highlight contours, ridges, and valleys [S1][S2, unofficial]. Unlike gradient-based methods that detect edges, cbTF emphasizes the "bumpiness" or deviation from a straight aspect of the underlying isosurfaces [S1][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume $V$ where each voxel at position $\mathbf{x}$ has an intensity value $f(\mathbf{x})$.
*   **Output:** A 2D lookup table (LUT) or shader function mapping the pair $(f(\mathbf{x}), \kappa(\mathbf{x}))$ to opacity $\alpha \in [0, 1]$ and color $C \in [0, 1]^3$ [S1][S3, unofficial].

**Data Structures:**
The classification domain is 2-dimensional, consisting of:
1.  **Data Value:** The scalar intensity $v = f(\mathbf{x})$.
2.  **Curvature:** A scalar metric $\kappa$ derived from the second derivatives of the volume data [S1][S4, unofficial].

The transfer function is typically implemented as a 2D texture (LUT) sampled during ray casting [S1].

**Computation of Curvature:**
Curvature measures how much a surface deviates from a straight aspect and is computed via signal convolution [S4, unofficial]. It is derived from the principal curvatures, specifically the maximum curvature $k_1$ and minimum curvature $k_2$ [S4, unofficial]. Common derived metrics include:
*   **Mean Curvature:** $H = \frac{k_1 + k_2}{2}$
*   **Gaussian Curvature:** $K = k_1 \cdot k_2$

The second directional derivative $f''$ is often approximated using the Hessian matrix $H$ or, with lower computational cost but less precision, the Laplacian operator $\nabla^2 f$ [S4, unofficial].

**Algorithm Steps:**
1.  **Smoothing (Optional but Recommended):** Apply a smoothing filter to the input volume $V$ to combat spurious curvature variations caused by noise, particularly in MRI datasets [S2, unofficial]. Note that this sacrifices accuracy and resolution [S2, unofficial].
2.  **Curvature Calculation:** For each voxel $\mathbf{x}$, compute the curvature $\kappa(\mathbf{x})$. This requires a neighborhood of at least 26 points (the 3x3x3 cube excluding the center) to estimate the second derivatives accurately [S1].
3.  **TF Evaluation:** For each sample point $\mathbf{x}$ along a ray:
    a. Retrieve the scalar value $v = f(\mathbf{x})$.
    b. Retrieve the precomputed curvature $\kappa = \kappa(\mathbf{x})$.
    c. Sample the 2D transfer function $T(v, \kappa)$ to obtain opacity $\alpha$ and color $C$.
4.  **Accumulation:** Perform standard volume rendering accumulation (e.g., front-to-back compositing) using the retrieved $\alpha$ and $C$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing: Compute curvature for the entire volume
def compute_curvature(volume):
    """
    Computes curvature for each voxel using a 26-point neighborhood.
    Returns a volume of the same dimensions containing curvature values.
    """
    height, width, depth = volume.shape
    curvature_vol = np.zeros_like(volume)
    
    # Approximation using Laplacian for efficiency, or Hessian for precision
    # Laplacian kernel for 3D (26-point neighborhood approximation)
    # This is a simplified representation; exact kernel weights depend on the specific 
    # derivative approximation method (e.g., central differences).
    
    for z in range(1, depth - 1):
        for y in range(1, width - 1):
            for x in range(1, height - 1):
                # Compute second derivatives or Laplacian
                # Example: Laplacian approximation
                laplacian = (
                    volume[x+1, y, z] + volume[x-1, y, z] +
                    volume[x, y+1, z] + volume[x, y-1, z] +
                    volume[x, y, z+1] + volume[x, y, z-1] -
                    6 * volume[x, y, z]
                )
                # Map Laplacian to curvature metric (sign and magnitude matter)
                # Note: Source [S4, unofficial] notes Laplacian is less precise than Hessian
                curvature_vol[x, y, z] = laplacian 
                
    return curvature_vol

# Rendering: Evaluate Transfer Function
def render_sample(x, y, z, volume, curvature_vol, tf_2d_texture):
    """
    Evaluates the curvature-based transfer function for a single sample point.
    """
    # 1. Get scalar value
    v = volume[x, y, z]
    
    # 2. Get precomputed curvature
    kappa = curvature_vol[x, y, z]
    
    # 3. Sample 2D Transfer Function
    # tf_2d_texture is a 2D texture mapped to [v_min, v_max] x [kappa_min, kappa_max]
    alpha, color = tf_2d_texture.sample(v, kappa)
    
    return alpha, color
```

**Parameters:**
*   **Smoothing Kernel:** Size and sigma of the Gaussian filter used to reduce noise before curvature calculation. Not given in the sources.
*   **Curvature Metric:** Choice between Mean, Gaussian, or Principal curvatures. [S4, unofficial]
*   **Derivative Approximation:** Hessian-based (precise) vs. Laplacian-based (fast). [S4, unofficial]

**Edge Cases and Pitfalls:**
*   **Noise Amplification:** Curvature is highly sensitive to noise. In noisy datasets (e.g., MRI), spurious surface curvature variations occur, leading to incorrect classification [S1][S2, unofficial]. Smoothing is required but degrades resolution [S2, unofficial].
*   **Ambiguity:** Many distinct features share the same curvature value. It is difficult to emphasize a targeted feature without inadvertently emphasizing other features with similar curvature [S1].
*   **Boundary Conditions:** The 26-point neighborhood requirement makes boundary handling complex. [S1]

## Subtopics

### Curvature
A specific metric computed by signal convolution that describes local shape contours and measures surface deviation from a straight aspect [S4, unofficial]. It is derived from maximum ($k_1$) and minimum ($k_2$) principal curvatures, allowing for the generation of Mean and Gaussian curvature variants [S4, unofficial]. This variant is commonly used for non-photorealistic visualizations to highlight contours [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** High. Computing curvature requires a 26-point neighborhood per voxel, making precomputation expensive [S1].
*   **Memory Complexity:** Requires storing an additional volume for curvature values, or computing it on the fly.
*   **Real-time Capability:** Yes, if precomputed. A real-time GPU variant requires a minimum of 27 3D texture lookups per sample (1 for data, 26 for neighborhood curvature estimation) if computed on-the-fly, or 2 lookups (data + curvature) if curvature is precomputed and stored in a 2D LUT [S1].
*   **Strengths:** Effective at distinguishing features with similar intensities but different shapes (e.g., small vs. large blobs); excellent for non-photorealistic contouring and ridge/valley visualization [S1][S3, unofficial].
*   **Limitations:** Highly sensitive to noise; suffers from spurious variations in low-quality data; difficult to isolate specific features due to curvature ambiguity [S1][S2, unofficial].
*   **When to use:** When geometric shape is more discriminative than intensity, or for artistic/non-photorealistic rendering emphasizing contours [S1][S2, unofficial].
*   **When not to use:** In noisy datasets without significant smoothing (which loses detail), or when precise isolation of a single feature type is required without side effects [S1].

## Background
The curvature-based transfer function was introduced by Kindlmann et al. in 2003 [KWTM03], using curvature as a second dimension in the transfer function domain to enable non-photorealistic renderings that highlight volume contours [S3, unofficial]. It is described in reference [16] of [S2, unofficial] as a method to aid classification by specifying isosurfaces and calculating curvature using differential invariants. Real-time GPU implementations were later detailed in *GPU Gems 2* [PF05] and [HSS∗05] for iso-surfaces [S1]. It is categorized alongside 1D, gradient-based, size-based, texture-based, and distance-based transfer functions in comprehensive surveys [S1].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S2] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S3] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S4] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
