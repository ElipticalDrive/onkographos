# Multi-Seed Streamline Tracking

Also called: parallel streamline tracking, volume seeding

## Overview
Multi-Seed Streamline Tracking is a method for visualizing vector and tensor fields by integrating streamlines from multiple simultaneous seed points. It solves the problem of sparse or ambiguous flow representation by seeding a neighborhood of voxels (e.g., a 26-neighbourhood) or a regular grid across the volume, rather than relying on single isolated seeds. It sets itself apart from [Adaptive Stream Line Seeding](5604-adaptive-stream-line-seeding.md) by using fixed spatial intervals or local neighborhoods for seeding rather than dynamic subdivision based on flow divergence, and from [Entropy-based Seeding](14471-entropy-based-seeding.md) by not using information-theoretic metrics to determine seed density.

## How it works
**Inputs:**
*   A vector or tensor field defined on a voxel grid.
*   A seed point (or set of seed points).
*   Integration parameters (step size, order of integration method).
*   Display parameters (arrow head rendering, cut-view precision).

**Outputs:**
*   A set of 3D streamlines representing the flow field.
*   Visual markers (arrow heads) at the start and end of each streamline.

**Data Structures:**
*   **Streamline:** A list of 3D coordinates $(x, y, z)$ representing the path.
*   **Seed Point:** A 3D coordinate $(x, y, z)$ in voxel space or physical space.
*   **Neighborhood:** A 3x3x3 grid of voxels centered on a seed point (26 neighbours).

**Steps:**
1.  **Seed Generation:**
    *   *Option A (Local):* For a given seed point, generate seed points for all 26 neighbouring voxels. This captures local flow variations.
    *   *Option B (Global):* Seed all points in the data volume at a specified step interval. This provides a denser representation of flow connectivity but is computationally slower [S1, unofficial].
    *   *Option C (Precision):* For interactive cut-view displays, seed start points with greater precision on the 2D window to overcome depth ambiguities inherent in Maximum Intensity Projections (MIP) [S1, unofficial].
2.  **Integration:**
    *   Integrate the vector/tensor field from each seed point simultaneously.
    *   Use a numerical integration method, such as 4th order Runge-Kutta [S2].
    *   For tensor fields, handle the sign ambiguity of eigenvectors by taking both forward and backward steps from the seed point [S1, unofficial].
3.  **Validation (Probabilistic Variant):**
    *   In diffusion tensor imaging (DTI) contexts, compute the probability $P_{total}$ that the current sample point represents a nerve fibre tract.
    *   $P_{total} = P_{anisotropy} \times P_{mean\_diffusivity}$ [S2].
    *   If $P_{total} < 0.60$ (60%), the integration is considered invalid or the point is not in a tract, and the streamline may be terminated [S2].
4.  **Post-Processing:**
    *   Add arrow heads to the start and end of individual stream lines for display orientation [S1, unofficial].
    *   Filter streamlines based on length and curvature constraints.

**Parameters:**
*   **Step Interval:** Determines the density of global seeding. Not given in the sources for a specific value, but described as "specified" [S1, unofficial].
*   **Minimum Length Threshold:** For DTI fibre tracts, the minimum length is 8.2 mm [S2].
*   **Maximum Curvature:** Controlled to ensure realistic fibre paths, but the specific value is not specified [S2].
*   **Probability Threshold:** 60% for nerve fibre tract validity [S2].

**Edge Cases and Pitfalls:**
*   **Depth Ambiguity:** When setting seed positions from a MIP alone, depth ambiguity can occur. Using precision seeding on a cut-view display window overcomes this [S1, unofficial].
*   **Performance:** Seeding all points in the data volume at a specified step interval is noted as "quite slow" [S1, unofficial].
*   **Tensor Sign Ambiguity:** Eigenvectors have arbitrary signs. Taking both forward and backward steps from the seed point overcomes this ambiguity [S1, unofficial].

## Subtopics

### Probability-based Seed Point Generation
A variant used in Diffusion Tensor Imaging (DTI) to determine seed points based on tissue classification probabilities. It computes the probability that a sample point is within a nerve fibre tract using the weighted product of anisotropy and mean diffusivity probabilities ($P_{total} = P_{anisotropy} \times P_{mean\_diffusivity}$) [S2]. It generates potential seed points in the vicinity of a sample point, where the number of points is determined by the computed probability. It tests at each integration step if the probability exceeds 60% to ensure validity [S2]. This method creates around 9500 start points from 1 million voxels [S2]. It is chosen when visualizing nerve fibre tracts in brain anatomy where distinguishing white matter from grey matter and CSF is critical [S2].

## Operating characteristics
*   **Time Complexity:** Seeding all points in the volume is "quite slow" compared to local neighbourhood seeding [S1, unofficial]. Parallel tracking of multiple seeds allows for efficient computation on modern hardware.
*   **Memory Complexity:** Depends on the number of streamlines generated. Global seeding generates significantly more data than local seeding.
*   **Strengths:** Provides a better impression of flow connectivity than static vector displays alone [S1, unofficial]. Overcomes depth ambiguities in MIPs when using precision seeding [S1, unofficial]. Effective for dense 3D visualizations of nerve fibre structures [S2].
*   **Limitations:** Global seeding is computationally expensive [S1, unofficial]. Requires careful parameter tuning for probability thresholds and length/curvature constraints in DTI [S2].
*   **When to Use:** When a comprehensive view of flow connectivity is needed, or when visualizing complex tensor fields like brain fibre tracts where tissue probability is a strong indicator of valid paths.
*   **When Not to Use:** When real-time performance is critical and global seeding is too slow, or when the flow field is simple and sparse seeding is sufficient.

## Background
The method originates from work on interactive visualization of MRI vector and tensor fields, specifically within the Angiotool project [S1, unofficial]. It was developed to address the difficulties in understanding the size and multi-dimensional nature of diffusion tensor data sets [S2]. The probabilistic variant was introduced to improve the perception of 3D arrangements of fibre tracts in brain anatomy [S2].

## Sources
[S1] Angiotool : a tool for interactive visualization of MRI vector and tensor fields, https://doi.org/10.82444/warw.33642268, fetched 2026-10-26 (unofficial)
[S2] PDF DTI Volume Rendering Techniques for Visualising the Brain Anatomy, https://www.cs.auckland.ac.nz/~burkhard/Publications/CARS05_WuenscheLindenHolmberg.pdf, fetched 2026-10-29
