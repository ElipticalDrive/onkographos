# Dwivedi Sampling

Also called: Biasing Random Walk

## Overview
Dwivedi sampling is a variance reduction technique for Monte Carlo light transport in participating media, specifically designed for high-scattering and isotropic scenarios [S1]. It improves convergence by biasing the random walk probability distributions to exit the medium as quickly as possible, operating on the premise that paths close to the boundary contribute more significantly to the final image than paths deep within the volume [S1][S2, unofficial]. Unlike standard distance sampling which treats the medium uniformly, Dwivedi sampling approximates the local geometry as a slab to analytically estimate the likelihood of exit, thereby reducing the number of "lost" paths that wander uselessly inside the object [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current scattering vertex position $\mathbf{x}$, incoming direction $\omega_i$, medium properties (extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$), and the geometric boundary of the participating object.
*   **Outputs:** A sampled distance to the next interaction $s$ and/or a sampled outgoing direction $\omega_o$, along with the associated probability density function (PDF) value $p(s)$ or $p(\omega_o)$ for importance weighting.

**Data Structures:**
*   **Slab Approximation:** A local geometric representation defined by the entry point $\mathbf{x}_{in}$ and exit point $\mathbf{x}_{out}$ of the probe ray passing through $\mathbf{x}$. The thickness $T$ is the distance $|\mathbf{x}_{out} - \mathbf{x}_{in}|$.
*   **Sampling Weights:** A set of weights $w_k$ for each sampling method $k$ used in Multiple Importance Sampling (MIS).

**Steps:**
1.  **Probe Ray Casting:** Shoot a ray from the current vertex $\mathbf{x}$ in the direction of the incoming light or the primary ray direction to intersect the object boundary. This yields the entry point $\mathbf{x}_{in}$ and exit point $\mathbf{x}_{out}$ [S2, unofficial].
2.  **Slab Thickness Estimation:** Calculate the local slab thickness $T = |\mathbf{x}_{out} - \mathbf{x}_{in}|$ [S2, unofficial]. This crudely approximates the object as a planar slab [S1][S2, unofficial].
3.  **Analytic Probability Calculation:** Compute the probability of exiting the slab within a distance $s$ using the analytic solution for a slab with extinction coefficient $\sigma_t$. The transmission through a slab of thickness $T$ is $e^{-\sigma_t T}$ [S1]. The PDF for distance sampling biased towards exit is derived from the derivative of the transmission function relative to the slab geometry [S1].
4.  **Sampling Mix:** Combine the Dwivedi-biased sampling with classical distance/direction sampling methods. Specifically, mix the slab-based sampling with standard exponential distance sampling $p_{classical}(s) = \sigma_t e^{-\sigma_t s}$ and potentially direction sampling methods [S2, unofficial].
5.  **Multiple Importance Sampling (MIS):** Apply MIS to combine the samples. The sources specify mixing with classical methods and using modified heuristic weights from [Kd14] for distance and direction sampling [S2, unofficial]. For RGB renders, this involves weighting between 6 methods (2 directions $\times$ 3 channels or similar combinatorial mix depending on implementation) [S2, unofficial].
6.  **Weight Adjustment:** The MIS weights are modified from the standard heuristic in [Kd14] to be slightly different for distance and direction sampling to better balance the bias [S2, unofficial].

**Formulas:**
*   Slab Transmission: $T(s) = e^{-\sigma_t s}$ for a path of length $s$ within the slab [S1].
*   Biased PDF: The exact analytic form of the Dwivedi PDF is not explicitly provided in the notes, but it is derived from the slab approximation to favor shorter paths that exit the boundary [S1].
*   MIS Weight: $w_k = \frac{\alpha_k p_k(x)}{\sum_j \alpha_j p_j(x)}$, where $\alpha_k$ are the heuristic weights modified from [Kd14] [S2, unofficial].

**Parameters:**
*   **Slab Approximation:** Used as a local surrogate for complex geometry [S1][S2, unofficial].
*   **MIS Heuristic Weights:** Modified from [Kd14]; specific values are "Not given in the sources" [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Thin Geometries:** The method is unsuitable for thin geometries (such as ears) with strong backlighting [S1]. The slab approximation fails to capture the complex light transport in thin regions, leading to high variance or incorrect biasing [S1].
*   **Precision:** The probe ray must accurately intersect the boundary; errors in $\mathbf{x}_{in}$ and $\mathbf{x}_{out}$ directly corrupt the slab thickness $T$ and the resulting bias [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dwivedi_sampling(vertex, medium, geometry):
    # 1. Probe ray to estimate thickness
    # Shoot ray from vertex in relevant direction (e.g., incoming light direction)
    hit_in, hit_out = geometry.intersect_through(vertex)
    
    # 2. Calculate slab thickness
    T = distance(hit_in, hit_out)
    
    # 3. Compute slab-based PDF for distance
    # Analytic approximation based on slab transmission
    # Note: Exact formula from Dwivedi 1982 is not fully detailed in sources
    # Assuming standard exponential decay biased by slab exit probability
    sigma_t = medium.extinction
    p_slab = compute_slab_pdf(vertex, T, sigma_t) 
    
    # 4. Classical sampling
    p_classical = sigma_t * exp(-sigma_t * random_distance())
    
    # 5. Direction sampling (if applicable)
    # Mix with direction sampling methods from [Kd14]
    p_dir = compute_direction_pdf(vertex, medium.phase)
    
    # 6. MIS Combination
    # Modified heuristic weights from [Kd14]
    # Specific weights for distance and direction are modified [S2, unofficial]
    weights = get_modified_heuristic_weights(p_slab, p_classical, p_dir)
    
    # Select sample based on weights
    sample = select_mis_sample([p_slab, p_classical, p_dir], weights)
    
    return sample, weights
```

## Subtopics

### Meng et al. Biasing Sampling
Meng et al. [S1] proposed variants to address Dwivedi sampling's limitations with thin geometries. **Closest points sampling** searches for the closest point to the boundary at every scattering vertex to increase the chances of escaping thin geometry [S1]. **Incident illumination sampling** chooses light vertices proportional to their emission using importance sampling, specifically improving variance for backlit cases [S1]. These methods are only suitable for high-order scattering and isotropic participating media [S1].

### Adjoint-guided Tracking
Adjoint-guided tracking is a future work technique where the probabilities $P_a$ (absorption), $P_s$ (scattering), and $P_n$ (null scattering) in the tracker are biased based on scene information [S3]. This includes increasing absorption rates in emissive parts or scattering more in brightly lit regions to reduce variance [S3]. It cites Dwivedi (1982) for advanced zero-variance schemes [S3].

## Operating characteristics
*   **Time Complexity:** Adds overhead for probe ray casting and slab intersection tests [S2, unofficial]. The MIS combination adds computational cost for evaluating multiple PDFs and weights [S2, unofficial].
*   **Memory Complexity:** Minimal additional memory for storing slab parameters and MIS weights [S2, unofficial].
*   **Performance:** Leads to lower variance in high-scattering, isotropic media [S1]. Significantly improves convergence by reducing lost paths [S2, unofficial].
*   **Strengths:** Effective for thick, highly scattering volumes where paths deep inside contribute little [S1].
*   **Limitations:** Fails on thin geometries with strong backlighting [S1]. Limited to isotropic media [S1].
*   **When to use:** High-scattering, isotropic participating media with significant thickness [S1][S2, unofficial].
*   **When not to use:** Thin geometries (e.g., ears, leaves) or anisotropic media [S1].

## Background
Dwivedi sampling originates from Dwivedi (1982), who proposed advanced zero-variance schemes for radiation transport [S3]. The technique was later adapted for computer graphics by Dwivedi et al. (cited as [40] in [S1] and [D’E16] in [S2, unofficial]) to improve subsurface scattering and volumetric path tracing [S1][S2, unofficial]. It builds on earlier biasing techniques such as those by Spanier and Gelbard (1969) and Steen (1966) for biasing probabilities towards increasing absorption rates [S3].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
[S3] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
