# Data Preprocessing


## Overview
Data preprocessing prepares raw volumetric or projection data for optimization by normalizing physical quantities and constructing sparse-view inputs. It solves the problem of inconsistent scales and insufficient angular coverage in sparse-view tomography and inverse rendering. It differs from [Volume Normalization](1823-volume-normalization.md) by handling the initial conversion from raw intensity to physical attenuation and global scaling, and from [Geometric Normalization](12018-geometric-normalization.md) by focusing on intensity and view subsampling rather than coordinate system alignment.

## How it works
**Inputs:** Raw projection images $I_v$, air-scan intensity $I_0$, scan metadata (source-to-detector distance, source-to-object distance, detector pixel pitch, rotation axis), and optionally ground-truth geometry (point clouds or meshes).
**Outputs:** Normalized attenuation projections, sparse-view subsets, and sampled supervision points.

**Data Structures:**
*   **Projection:** 2D array of size $H \times W$ (e.g., $560 \times 560$).
*   **Volume:** 3D grid of size $N^3$ (e.g., $256^3$).
*   **Supervision Points:** List of $(x, y, z)$ coordinates and associated labels (SDF, DDF, or ray hits).

**Steps:**

1.  **Attenuation Conversion:** Convert raw intensity $I_v$ to attenuation $P_v$ using the Beer–Lambert law [S1, unofficial]:
    $$ P_v = -\log\left(\frac{I_v}{I_0}\right) $$
    where $I_0$ is the intensity of the air scan (no object) [S1, unofficial].

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
*   **Time Complexity:** Linear in the number of pixels/views for normalization and subsampling [S1, unofficial]. Sampling supervision points is linear in the number of samples [S2].
*   **Memory Complexity:** Proportional to the number of views and grid resolution ($560 \times 560$ for projections, $256^3$ for volume) [S1, unofficial].
*   **Strengths:** Ensures physical consistency via Beer–Lambert law; stabilizes optimization via global normalization [S1, unofficial].
*   **Limitations:** Sparse-view subsampling ($K < 100$) leads to ill-posed reconstruction problems [S1, unofficial].

## Background
Originated in sparse-view 3D Gaussian tomography [S1, unofficial] and inverse rendering using distance functions [S2]. Used in the FIPS dataset [S1, unofficial] and ShapeNet-based inverse rendering [S2].

## Sources
[S1] Projection-Volume Fidelity Divergence: Diagnosing and Controlling Optimization Drift in Sparse-View 3D Gaussian Tomography, https://doi.org/10.48550/arXiv.2606.22525, fetched 2026-10-27 (unofficial)
[S2] FIRe: Fast Inverse Rendering Using Directional and Signed Distance Functions, https://www.youtube.com/watch?v=Qj0ymDlRYZs, fetched 2026-10-30
