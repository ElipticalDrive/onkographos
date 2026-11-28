# Fourier PlenOctrees


## Overview
Fourier PlenOctrees (FPO) is a representation for the real-time rendering of dynamic neural radiance fields (NeRF) that combines the static sparse octree structure of PlenOctrees with discrete Fourier transform (DFT) compression to compactly store time-dependent information [S1]. It solves the problem of rendering temporally evolving scenes by unifying multiple static PlenOctrees into a single structure where leaf entries are compressed in the frequency domain, enabling efficient reconstruction of density and color at any time step $t$ [S1]. Unlike static PlenOctrees which store fixed values, FPO stores Fourier coefficients, allowing for continuous time interpolation and significant memory reduction, though it introduces low-frequency approximation artifacts [S1].

## How it works
**Inputs:** A set of $T$ static PlenOctrees, one for each time step in a dynamic sequence. Each static PlenOctree contains leaf nodes with density $\sigma$ and spherical harmonic (SH) coefficients $z$ for color.
**Outputs:** A single Fourier PlenOctree structure containing DFT-compressed leaf entries, and a renderer capable of generating images for any time step $t \in \{0, \dots, T-1\}$.

**Data Structures:**
The FPO is a sparse octree. Each leaf node $i$ stores:
1.  **Fourier Coefficients for Density:** A vector of complex numbers representing the DFT of the density sequence over time. Let $K_\sigma$ be the number of frequency components stored.
2.  **Fourier Coefficients for SH Coefficients:** For each color channel ($R, G, B$) and each SH coefficient index $k \in \{0, \dots, Z-1\}$, a vector of complex numbers representing the DFT of that coefficient's temporal evolution. $Z$ is the total number of SH coefficients per color channel.

**Construction Steps:**
1.  **Structure Unification:** Merge the $T$ static PlenOctrees into a single sparse octree structure. This involves aligning the spatial hierarchy so that corresponding leaves across time steps occupy the same positions in the unified tree [S1].
2.  **DFT Compression:** For each leaf node $i$ in the unified structure:
    a. Extract the sequence of density values $\sigma_i(t)$ for $t=0 \dots T-1$.
    b. Extract the sequence of SH coefficient vectors $z_i(t) \in \mathbb{R}^{Z \times 3}$ for $t=0 \dots T-1$.
    c. Apply the Discrete Fourier Transform (DFT) to these sequences to obtain the compressed Fourier coefficients stored in the leaf [S1].

**Rendering Steps:**
To render an image at time step $t$:
1.  **Ray Casting:** For each pixel, cast a ray $r(\tau) = o + \tau \cdot d$, where $o$ is the camera origin and $d$ is the viewing direction [S1].
2.  **Leaf Traversal:** Traverse the FPO to find the $N$ leaf nodes hit by the ray.
3.  **Reconstruction:** For each hit leaf node $i$:
    a. **Density:** Reconstruct the time-dependent density $\sigma_i(t)$ by applying the Inverse Discrete Fourier Transform (IDFT) to the stored density Fourier coefficients at time $t$ [S1].
    b. **Color:** Reconstruct the time-dependent SH coefficients $z_i(t) \in \mathbb{R}^{Z \times 3}$ by applying the IDFT to the stored SH Fourier coefficients at time $t$. Then, evaluate the SH basis functions for the viewing direction $d$ to get the raw color vector. Apply the sigmoid function to normalize the color $c_i(t)$ [S1].
4.  **Volume Rendering:** Accumulate the color $C(r, t)$ along the ray using standard volume rendering integration [S1]:
    $$ \hat{C}(r, t) = \sum_{i=1}^{N} T_{i-1}(t) \cdot (1 - \exp(-\sigma_i(t)\delta_i)) \cdot c_i(t) $$
    where $\delta_i$ is the distance between voxel borders for leaf $i$, and $T_{i-1}(t)$ is the accumulated transmittance from the camera up to leaf node $i$ [S1].
5.  **Transmittance:** Compute accumulated transmittance for leaf $i$ as [S1]:
    $$ T_{i-1}(t) = \exp\left(-\sum_{j=1}^{i-1} \sigma_j(t)\delta_j\right) $$

**Parameters:**
*   $T$: Total number of frames in the dynamic time sequence [S1].
*   $t$: Current time step, integer in range $\{0, \dots, T-1\}$ [S1].
*   $Z$: Number of spherical harmonic (SH) coefficients per color channel [S1].

**Pitfalls:**
*   **Artifacts:** Fourier-based compression results in a low-frequency approximation of the original data. This is susceptible to artifacts in both reconstructed geometry and color, which often persist even after fine-tuning [S1].
*   **Negative Densities:** The IDFT reconstruction may produce negative density values, which cannot be meaningfully interpreted in volume rendering [S1].
*   **High Density Errors:** Standard L2 minimization during fine-treatment treats approximation errors of all values equally, which can lead to large positive errors in high-density areas that are less perceptually significant than errors in low-density areas [S1].

## Subtopics

### Logarithmic Density Encoding
An encoding applied to density values before DFT compression to map larger values closer together and smaller values apart, focusing approximation error on low-density areas [S1]. During FPO construction, density $\sigma$ is encoded using $e_{log}(\sigma) = \log(\sigma + 1)$ before applying DFT [S1]. During rendering, the inverse logarithmic function is applied after IDFT reconstruction to project densities back to the original range [S1]. This allows for higher approximation error of high densities compared to linear encoding, improving reconstruction of low-density areas like empty space [S1].

## Operating characteristics
*   **Time Complexity:** Rendering requires IDFT computation per leaf per frame. The complexity depends on the number of frequency components $K$ and the number of SH coefficients $Z$. The integration step is linear in the number of leaves hit by the ray.
*   **Memory Complexity:** Significantly reduced compared to storing $T$ full PlenOctrees. Memory usage is proportional to the number of leaves times the number of stored frequency components $K$ and SH coefficients $Z$, rather than $T \times Z$.
*   **Performance:** Enables real-time rendering of dynamic scenes [S1].
*   **Strengths:** Compact storage of temporal data; real-time rendering capability; differentiable, allowing for fine-tuning [S1].
*   **Limitations:** Susceptible to low-frequency artifacts; potential for negative densities; approximation errors in high-density regions if not encoded [S1].
*   **When to use:** Real-time dynamic NeRF rendering where memory is constrained and some artifact tolerance is acceptable.
*   **When not to use:** When high-fidelity reconstruction of sharp temporal details or high-density boundaries is critical without post-processing.

## Background
Fourier PlenOctrees were introduced as an extension of the static PlenOctree representation [72] to handle dynamic scenes. The method was analyzed and enhanced in "FPO++: efficient encoding and rendering of dynamic neural radiance fields by analyzing and enhancing Fourier PlenOctrees" by Saskia Rabich, Patrick Stotko, and Reinhard Klein, published in *The Visual Computer* (2024) [S1]. The work builds upon earlier applications of NeRF to dynamic scenarios, which often suffered from slow rendering speeds, by combining the advantages of static explicit representations with frequency-domain compression [S1].

## Sources
[S1] FPO++: efficient encoding and rendering of dynamic neural radiance fields by analyzing and enhancing Fourier PlenOctrees, https://doi.org/10.1007/s00371-024-03475-3, fetched 2026-10-26
