# Data Preprocessing
## Overview
Data preprocessing converts raw X-ray CT projections to attenuation values, normalizes them globally, and subsamples views for sparse-view tomography [S1, unofficial]. In inverse rendering, it samples supervision points for signed and directional distance functions [S2].
## How it works
**Inputs:** Raw projection images $I_v$, air-scan intensity $I_0$, scan metadata, and ground-truth geometry (point clouds or meshes) [S1, unofficial][S2].
**Outputs:** Normalized attenuation projections, sparse-view subsets, and sampled supervision points [S1, unofficial][S2].

**Data Structures:**
*   **Projection:** 2D array of size $H \times W$ (e.g., $560 \times 560$) [S1, unofficial].
*   **Volume:** 3D grid of size $N^3$ (e.g., $256^3$) [S1, unofficial].
*   **Supervision Points:** List of $(x, y, z)$ coordinates and associated labels (SDF, DDF, or ray hits) [S2].

**Steps:**

1.  **Attenuation Conversion:** Convert raw intensity $I_v$ to attenuation $P_v$ using the Beer–Lambert law [S1, unofficial]:
    $$ P_v = -\log\left(\frac{I_v}{I_0}\right) $$
    where $I_0$ is the intensity of the air scan [S1, unofficial].

2.  **Global Normalization:** Compute the global maximum attenuation $P_{\max}$ across all measured views and all voxels/projections [S1, unofficial]. Normalize all attenuation values to $[0, 1]$ by dividing by $P_{\max}$ [S1, unofficial]. This scaling factor is applied consistently to both projections and the reconstructed volume [S1, unofficial].

3.  **Sparse-View Subsampling:** From the original acquisition (several hundred views), uniformly subsample $K$ views to create the sparse input, where $K \in \{10, 20, 25\}$ [S1, unofficial].

4.  **Geometry Setup:** Use scan metadata to define the cone-beam geometry parameters (source-to-detector, source-to-object, pixel pitch, rotation axis) for the forward Radon transform model [S1, unofficial].

5.  **Supervision Sampling (Inverse Rendering Context):**
    *   **SDF Supervision:** Sample $\sim 1$ million points using the DeepSDF pipeline [S2].
    *   **DDF Supervision:** Sample 1 million points on the unit sphere and random directions pointing to the object surface using the object's point cloud [S2].
    *   **Ray Hit Supervision:** Render rays using a trimesh representation (e.g., Trimesh [14]) to obtain ground-truth distances and ray hit labels [S2].

**Parameters:**
*   $K$: Number of sparse views $\{10, 20, 25\}$ [S1, unofficial].
*   Grid Resolution: $256^3$ for volume evaluation [S1, unofficial].
*   Projection Resolution: $560 \times 560$ [S1, unofficial].
*   Sample Count: $\sim 1$ million points for SDF/DDF [S2].

**Edge Cases:**
*   **Precision:** Logarithm requires $I_v > 0$ and $I_0 > 0$. Division by zero in normalization if $P_{\max} = 0$ (empty scan) [G].
*   **Boundaries:** Global maximum must include all views to ensure consistent scaling [S1, unofficial].
## Operating characteristics
## Background
Applied in sparse-view X-ray CT reconstruction using the FIPS dataset [S1, unofficial] and in inverse rendering with the ShapeNet dataset [S2].
## Sources
[S1] Projection-Volume Fidelity Divergence: Diagnosing and Controlling Optimization Drift in Sparse-View 3D Gaussian Tomography, https://doi.org/10.48550/arXiv.2606.22525, fetched 2026-10-27 (unofficial)
[S2] FIRe: Fast Inverse Rendering Using Directional and Signed Distance Functions, https://www.youtube.com/watch?v=Qj0ymDlRYZs, fetched 2026-10-30
