# Multi-Seed Streamline Tracking

Also called: Adaptive Stream Line Tracking, ChomboVis Streamline Integration, ChomboVis backward Euler integration, parallel streamline tracking, volume seeding

## Overview
Multi-Seed Streamline Tracking visualizes vector and tensor fields by integrating streamlines from multiple simultaneous seed points [S2, unofficial]. It overcomes sparse flow representation by seeding a 26-neighbourhood of voxels around a target point or all volume points at a specified interval, rather than relying on isolated seeds [S2, unofficial]. It differs from [Adaptive Stream Line Seeding](5604-adaptive-stream-line-seeding.md) by using fixed spatial intervals or local neighborhoods instead of dynamic divergence-based subdivision, and from [Entropy-based Seeding](14471-entropy-based-seeding.md) by not using information-theoretic metrics for seed density [S2, unofficial].

## How it works
**Inputs:**
*   A vector or tensor field defined on a voxel grid [S1, unofficial].
*   Seed points, either a local 26-neighbourhood or a global volume grid [S2, unofficial].
*   Integration parameters: user-specified step length and step count [S1, unofficial].

**Outputs:**
*   A set of 3D streamlines representing the flow field [S1, unofficial].
*   Arrow heads at the start and end of each streamline for display orientation [S2, unofficial].

**Data Structures:**
*   **Streamline:** An ordered list of 3D coordinates $(x, y, z)$ [S1, unofficial].
*   **Seed Point:** A 3D coordinate in voxel or physical space [S2, unofficial].

**Steps:**
1.  **Seed Generation:** Generate seeds via a 26-neighbourhood of voxels around a target point, or globally at a specified step interval [S2, unofficial]. For interactive cut-views, seed with higher precision on the 2D window to resolve Maximum Intensity Projection (MIP) depth ambiguities [S2, unofficial].
2.  **Integration:** Integrate forward or backward using the specified step length and step count [S1, unofficial]. Sources specify either backward Euler integration [S1, unofficial] or 4th order Runge-Kutta [S3]. For tensor fields, take both forward and backward steps to resolve eigenvector sign ambiguity [S2, unofficial].
3.  **AMR Boundary Handling:** After each step, check if the current position remains inside the current Adaptive Mesh Refinement (AMR) box [S1, unofficial]. If outside, search the AMR hierarchy from finest to coarsest to find a valid box for continuation [S1, unofficial].
4.  **Probabilistic Validation:** Compute the probability $P_{total}$ that the current sample point represents a nerve fibre tract [S3]. $P_{total} = P_{anisotropy} \times P_{mean\_diffusivity}$ [S3]. If $P_{total} < 0.60$, terminate the streamline as the point is likely not in a tract [S3].

**Parameters:**
*   **Step Interval:** Determines global seeding density; exact numerical value is not specified, but the interval is user-defined [S2, unofficial].
*   **Minimum Length Threshold:** 8.2 mm for final fibre tracts [S3].
*   **Maximum Curvature:** Controlled to ensure realistic paths, but the specific value is not specified [S3].
*   **Probability Threshold:** 60% for nerve fibre tract validity [S3].

**Edge Cases and Pitfalls:**
*   **Depth Ambiguity:** Setting seed positions from a MIP alone causes depth ambiguity; precision seeding on a cut-view display window resolves this [S2, unofficial].
*   **Performance:** Seeding all points in the data volume at a specified step interval is computationally slow [S2, unofficial].
*   **Tensor Sign Ambiguity:** Eigenvectors have arbitrary signs; bidirectional stepping from the seed point overcomes this [S2, unofficial].
*   **AMR Boundary Crossing:** Integration crossing box boundaries requires a hierarchy search, adding computational overhead [S1, unofficial].

**Pseudocode:**
# Pseudocode not provided in the sources.

## Subtopics

### Probability-based Seed Point Generation
A variant used in Diffusion Tensor Imaging (DTI) to determine seed points based on tissue classification probabilities [S3]. It computes the probability that a sample point is within a nerve fibre tract using the weighted product of anisotropy and mean diffusivity probabilities ($P_{total} = P_{anisotropy} \times P_{mean\_diffusivity}$) [S3]. It generates potential seed points in the vicinity of a sample point, where the number of points is determined by the computed probability [S3]. It tests at each integration step if the probability exceeds 60% to ensure validity [S3]. This method creates around 9500 start points from 1 million voxels [S3]. It is chosen when visualizing nerve fibre tracts in brain anatomy where distinguishing white matter from grey matter and CSF is critical [S3].

### Stylus Seeding
A technique for exploring vector field datasets where a stylus effector acts as a dynamic particle source [S4]. It positions the stylus within the dataset to define the particle generation location [S4]. It generates a number of particles at the defined location with slight temporal or spatial offsets so their trajectories do not completely overlap [S4]. It animates particles following the vector field to visually identify structures [S4].

## Operating characteristics
*   **Time Complexity:** Global volume seeding is "quite slow" compared to local neighbourhood seeding [S2, unofficial]. Parallel tracking of multiple seeds enables efficient computation on modern hardware [S2, unofficial].
*   **Memory Complexity:** Scales with the number of generated streamlines; global seeding consumes significantly more memory than local seeding [S2, unofficial].
*   **Strengths:** Provides superior flow connectivity visualization compared to static vector displays [S2, unofficial]. Resolves MIP depth ambiguity via precision seeding [S2, unofficial]. Effectively visualizes dense 3D nerve fibre structures in DTI [S3].
*   **Limitations:** Global seeding incurs high computational cost [S2, unofficial]. DTI variants require careful tuning of probability thresholds and geometric constraints [S3].
*   **When to Use:** Comprehensive flow connectivity analysis or complex tensor field visualization (e.g., brain tracts) [S2, unofficial][S3].
*   **When Not to Use:** Real-time applications where global seeding latency is prohibitive, or simple/laminar flows where sparse seeding suffices [S2, unofficial].

## Background
The method originates from the ChomboVis/Amrvis toolkit for AMR data visualization [S1, unofficial] and the Angiotool project for MRI vector and tensor fields [S2, unofficial]. It was developed to address the complexity of multi-dimensional diffusion tensor datasets [S3]. It is used for interactive exploration of vector fields and nerve fibre tract extraction in brain anatomy [S2, unofficial][S3].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc894061/, fetched 2026-10-26 (unofficial)
[S2] Angiotool : a tool for interactive visualization of MRI vector and tensor fields, https://doi.org/10.82444/warw.33642268, fetched 2026-10-26 (unofficial)
[S3] PDF DTI Volume Rendering Techniques for Visualising the Brain Anatomy, https://www.cs.auckland.ac.nz/~burkhard/Publications/CARS05_WuenscheLindenHolmberg.pdf, fetched 2026-10-29
[S4] A portable interface for tangible exploration of volumetric data, https://doi.org/10.1145/2671015.2671130, fetched 2026-10-29
