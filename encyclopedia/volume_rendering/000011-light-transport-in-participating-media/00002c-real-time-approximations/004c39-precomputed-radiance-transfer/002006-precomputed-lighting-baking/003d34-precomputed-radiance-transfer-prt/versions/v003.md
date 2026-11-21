# Precomputed Radiance Transfer (PRT)

Also called: Directional surface prelighting, SH-based global illumination

## Overview
Precomputed Radiance Transfer (PRT) is a real-time global illumination technique that models light interaction with surfaces using transfer functions rather than operating on actual radiance values [S1, S2, unofficial]. It allows for arbitrary dynamic lighting setups at runtime by representing the scene's light transfer properties in a basis (typically Spherical Harmonics), offering greater flexibility than static full baking, albeit with higher runtime computational costs [S1, unofficial].

## How it works
PRT leverages the linearity of light transport to separate the geometry/material response from the lighting environment [S1, unofficial]. The method consists of three main phases: precomputing the transfer function, projecting the environment lighting into the same basis, and evaluating the final radiance via a dot product [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Geometry (mesh or voxel representation).
    *   Material properties (albedo, normal, roughness).
    *   Global illumination algorithm (e.g., radiosity, subsurface scattering) to determine the transfer function [S2, unofficial].
    *   Environment lighting (static or dynamic) [S2, unofficial].
*   **Outputs:**
    *   Per-vertex or per-voxel SH coefficients representing the light transfer function [S2, unofficial].
    *   Final radiance $L(p)$ for each point $p$ [S1, unofficial].

### Data Structures
The core data structure is a set of Spherical Harmonic (SH) coefficients.
*   **Transfer Function:** A vector of real numbers $T$ containing the precomputed response of the entity to each SH basis function. The size depends on the number of SH bands used.
*   **Environment Lighting:** A vector of real numbers $E$ containing the SH coefficients of the incoming light field.
*   **Storage:** These coefficients are typically stored per-vertex in a mesh or per-voxel in a grid, allowing for interpolation across polygons or trilinear interpolation in volumes [S2, unofficial].

### Steps

1.  **Precompute the Transfer Function:**
    For each SH basis function $Y_{lm}$:
    a. Represent the SH basis function as a light source approaching the entity from the infinite sphere [S2, unofficial].
    b. Because SH functions contain negative values, the lighting contribution must be computed twice: once with the positive light intensity and once with the opposite of the negative light intensity [S2, unofficial].
    c. Run the chosen global illumination algorithm (e.g., radiosity) for both cases [S2, unofficial].
    d. Subtract the result of the negative light calculation from the positive light calculation to obtain the net transfer response for that basis function [S2, unofficial].
    e. Store the resulting real numbers as the SH coefficients for the transfer function [S2, unofficial].

2.  **Project Environment Lighting into SH:**
    For dynamic lighting, this step must be performed at least partly per-frame [S2, unofficial].
    a. Sample the incoming light function $h(S)$ over the sphere $S$ using $n$ uniformly distributed random samples $X = \{x_1, x_2, ..., x_n\}$ [S2, unofficial].
    b. Compute the SH coefficients $\hat{c}^m_l$ for the environment lighting using the approximate integration formula:
    $$ \hat{c}^m_l = \frac{4\pi}{n} \sum_{i=1}^{n} h(x_i) Y_{lm}(x_i) $$
    where $Y_{lm}$ is the SH basis function for degree $l$ and order $m$ [S2, unofficial].
    c. If multiple lights are present, project each separately and sum their coefficients in SH space, as addition is trivial in this basis [S2, unofficial].

3.  **Evaluate Outgoing Light:**
    a. Compute the dot product between the precomputed transfer function SH coefficients ($T$) and the environment lighting SH coefficients ($E$) [S2, unofficial].
    b. This calculation is typically performed per-vertex in the model [S2, unofficial].
    c. Interpolate the resulting light values across each polygon (or voxel) [S2, unofficial].

### Formula
The final radiance $L(p)$ for a point $p$ is defined as the sum of the contributions of each light, leveraging the linearity of light transport [S1, unofficial]:

$$ L(p) = \sum_i L_i(p) W_i $$

Where:
*   $L(p)$ is the final radiance for point $p$ [S1, unofficial].
*   $L_i(p)$ is the precomputed unit contribution of light $i$ (the transfer function response) [S1, unofficial].
*   $W_i$ is the current brightness of light $i$ (or the SH coefficient corresponding to that light component) [S1, unofficial].

In the SH basis, this summation is equivalent to the dot product of the transfer vector and the environment lighting vector [S2, unofficial].

### Parameters
*   **SH Bands:** The number of spherical harmonic bands determines the frequency detail captured. Higher bands capture more complex lighting effects (e.g., sharp shadows, specularities) but increase storage and computation [S1, S2, unofficial].
*   **Sample Count ($n$):** The number of samples used for projecting the environment lighting into SH. More samples yield higher accuracy but increase per-frame cost [S2, unofficial].

### Edge Cases and Pitfalls
*   **Negative Coefficients:** SH basis functions have negative lobes. Standard lighting algorithms assume positive light intensities. The "compute positive and negative separately, then subtract" approach is required to handle this correctly [S2, unofficial].
*   **Dynamic Lighting Cost:** Projecting an arbitrary spherical environment into SH is expensive. For dynamic lights, this projection must be done per-frame, which can be a bottleneck [S2, unofficial].
*   **Deformation:** Standard PRT computes the transfer for an entire entity. If the geometry deforms significantly, the precomputed transfer may become inaccurate. Local PRT (LDPRT) addresses this by computing transfer for local surface patches, but it cannot capture effects like shadows from distant parts of the model [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Precomputation is offline and depends on the complexity of the global illumination algorithm and the number of SH bands. Runtime evaluation is $O(N)$ where $N$ is the number of SH bands (dot product). However, projecting dynamic lighting into SH is $O(n \cdot N)$ per frame, where $n$ is the number of samples [S2, unofficial].
*   **Memory Complexity:** Storage is proportional to the number of vertices/voxels times the number of SH bands [S2, unofficial].
*   **Strengths:** Supports arbitrary dynamic lighting setups; captures complex global illumination effects (indirect lighting, soft shadows) in real-time [S1, S2, unofficial].
*   **Limitations:** Higher runtime cost than static baking; inaccurate for highly deformable geometry unless using LDPRT; expensive per-frame projection for dynamic environments [S1, S2, unofficial].
*   **When to Use:** Real-time applications requiring dynamic global illumination with static or slowly changing geometry [S2, unofficial].
*   **When Not to Use:** Static scenes where full baking is sufficient; scenes with highly dynamic geometry where LDPRT is insufficient [S2, unofficial].

## Background
Precomputed Radiance Transfer was introduced by Sloan et al. in 2002 [S1, S2, unofficial]. While originally formulated using Spherical Harmonics, the concept of using transfer functions is not limited to SH [S1, unofficial]. A later extension, Local Deformable PRT (LDPRT), was introduced by Sloan et al. in 2005 to handle deformable geometry [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbc.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
