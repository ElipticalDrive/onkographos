# Precomputed Radiance Transfer (PRT)

Also called: Directional surface prelighting, PRT with SH, Precomputed Radiance Transfer (PRT) with SH, SH PRT, SH-based global illumination

## Overview
Precomputed Radiance Transfer (PRT) is a real-time global illumination technique that models light interaction with surfaces using transfer functions rather than operating on actual radiance values [S1, unofficial][S2, unofficial]. It differs from [Hybrid Preshading with Postclassification](2171-hybrid-preshading-with-postclassification.md) and [Pre-computation](2330-pre-computation.md) by explicitly separating the geometry/material response from the lighting environment via transfer functions, and builds on [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md) by allowing arbitrary dynamic lighting setups at runtime, albeit with higher runtime computational costs [S1, unofficial].

## How it works
PRT leverages the linearity of light transport to separate the scene's response from the lighting environment [S1, unofficial]. The method consists of three main phases: precomputing the transfer function, projecting the environment lighting into the same basis, and evaluating the final radiance via a dot product [S3, unofficial].

### Inputs and Outputs
*   **Inputs:** Geometry representation, chosen global illumination algorithm (e.g., radiosity, subsurface scattering) to determine the transfer function, and environment lighting (static or dynamic) [S3, unofficial].
*   **Outputs:** Per-vertex or per-voxel SH coefficients representing the light transfer function, and final radiance $L(p)$ for each point $p$ [S1, unofficial][S3, unofficial].

### Data Structures
*   **Transfer Function:** A vector of real numbers containing the precomputed response of the entity to each SH basis function [S3, unofficial].
*   **Environment Lighting:** A vector of real numbers containing the SH coefficients of the incoming light field [S3, unofficial].
*   **Storage:** Coefficients are stored per-vertex in a mesh or per-voxel in a grid, allowing interpolation across polygons or trilinear interpolation in volumes [S3, unofficial].

### Steps
1.  **Precompute the Transfer Function:**
    a. For each SH basis function $Y_{lm}$, represent it as light approaching the entity from the infinite sphere [S3, unofficial].
    b. Because SH functions contain negative values, compute the lighting contribution twice: once with the positive light intensity and once with the opposite of the negative light intensity [S3, unofficial].
    c. Run the chosen global illumination algorithm for both cases [S3, unofficial].
    d. Subtract the result of the negative light calculation from the positive light calculation to obtain the net transfer response for that basis function [S3, unofficial].
    e. Store the resulting real numbers as the SH coefficients for the transfer function [S3, unofficial].

2.  **Project Environment Lighting into SH:**
    a. For dynamic lighting, this step must be performed at least partly per-frame [S3, unofficial].
    b. Sample the incoming light function $h(S)$ over the sphere $S$ using $n$ uniformly distributed random samples $X = \{x_1, x_2, ..., x_n\}$ [S3, unofficial].
    c. Compute the SH coefficients $\hat{c}^m_l$ for the environment lighting using the approximate integration formula:
    $$ \hat{c}^m_l = \frac{4\pi}{n} \sum_{i=1}^{n} h(x_i) Y_{lm}(x_i) $$
    where $Y_{lm}$ is the SH basis function for degree $l$ and order $m$ [S3, unofficial].
    d. If multiple lights are present, project each separately and sum their coefficients in SH space, as addition is trivial in this basis [S3, unofficial].

3.  **Evaluate Outgoing Light:**
    a. Compute the dot product between the precomputed transfer function SH coefficients and the environment lighting SH coefficients [S3, unofficial].
    b. This calculation is typically performed per-vertex in the model [S3, unofficial].
    c. Interpolate the resulting light values across each polygon [S3, unofficial].

### Formula
The final radiance $L(p)$ for a point $p$ is defined as the sum of the contributions of each light, leveraging the linearity of light transport [S1, unofficial]:

$$ L(p) = \sum_i L_i(p) W_i $$

Where:
*   $L(p)$ is the final radiance for point $p$ [S1, unofficial].
*   $L_i(p)$ is the precomputed unit contribution of light $i$ (the transfer function response) [S1, unofficial].
*   $W_i$ is the current brightness of light $i$ [S1, unofficial].

In the SH basis, this summation is equivalent to the dot product of the transfer vector and the environment lighting vector [S2, unofficial].

### Parameters
*   **SH Bands:** The number of spherical harmonic bands determines the frequency detail captured [S3, unofficial].
*   **Sample Count ($n$):** The number of samples used for projecting the environment lighting into SH [S3, unofficial].

### Edge Cases and Pitfalls
*   **Negative Coefficients:** SH basis functions have negative lobes, requiring the compute-positive-and-negative-separately-then-subtract approach to handle them correctly [S3, unofficial].
*   **Dynamic Lighting Cost:** Projecting an arbitrary spherical environment into SH is expensive, requiring per-frame calculation for dynamic lights [S3, unofficial].
*   **Deformation:** Standard PRT computes the transfer for an entire entity, which becomes inaccurate if geometry deforms significantly. Local PRT (LDPRT) computes transfer for local surface patches but cannot capture effects like shadows from distant parts of the model [S3, unofficial].
*   **Rotation Handling:** For PRT, the transfer function cannot be easily rotated on the GPU; instead, rotate the lighting environment by the inverse model transform on the CPU [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Supports arbitrary dynamic lighting setups; captures complex global illumination effects (indirect lighting, soft shadows) in real-time [S1, unofficial][S3, unofficial].
*   **Limitations:** Higher runtime cost than static full baking [S1, unofficial]; expensive per-frame projection for dynamic environments [S3, unofficial]; inaccurate for highly deformable geometry unless using LDPRT [S3, unofficial].
*   **When to Use:** Real-time applications requiring dynamic global illumination [S3, unofficial].
*   **When Not to Use:** Static scenes where full baking is sufficient [S1, unofficial].

## Background
Precomputed Radiance Transfer was introduced by Sloan et al. in 2002 [S3, unofficial]. While originally formulated using Spherical Harmonics, the concept of using transfer functions is not limited to SH [S1, unofficial]. A later extension, Local Deformable PRT (LDPRT), was introduced by Sloan et al. in 2005 to handle deformable geometry [S3, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Irradiance Volumes for Games, https://chrisoat.com/papers/Oat_GDC2005_IrradianceVolumesForGames.pdf, fetched 2026-10-28 (unofficial)
[S3] A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbc.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
