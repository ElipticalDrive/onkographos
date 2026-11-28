# Pixel Intensity Evaluation with Scene Aggregates

Also called: AIV/ABV Integration, Aggregated Boundary Visibility, Aggregated Visibility Functions Integration

## Overview
Pixel Intensity Evaluation with Scene Aggregates is a method for evaluating pixel intensity by integrating radiance over a pixel footprint, using aggregated visibility functions to handle inter-voxel correlation and background blending without order-dependent ray marching [S1, unofficial]. It solves the problem of correctly blending scene aggregate contributions with external background objects by defining visibility on the 2D boundary enclosing a 3D scene aggregate [S1, unofficial]. It sets itself apart from its siblings in [Ray Traversal Schemes](6890-ray-traversal-schemes.md) by replacing sequential sample accumulation with a precomputed visibility integral that accounts for the geometric coverage of primitives within the pixel footprint [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A discretized 3D scene represented as voxels; a pixel footprint $P$ in world space; a viewing direction $\boldsymbol{\nu}_p$.
- **Output**: The final pixel intensity $I$.

**Data Structures**
- **Boundary Faces**: A list of 2D faces on the boundary of the voxel grid, determined by flood fill iteration [S1, unofficial].
- **ABV Map**: A 2D visibility map stored in the Haar wavelet basis, parameterizing the inward-facing hemisphere of each boundary face [S1, unofficial].
- **Primitive Coverage**: For each voxel, a truncated ellipsoid primitive $B$ defining its geometric extent [S1, unofficial].

**Steps**
1. **Scene Discretization**: Represent the scene as a voxel grid. Identify the boundary faces of the entire scene aggregate using a flood fill iteration on the discretized scene [S1, unofficial].
2. **ABV Precomputation**: For each boundary face:
   a. Consider all directions in its inward-facing hemisphere.
   b. Use concentric mapping to parameterize the hemispherical domain into a 2D space [S1, unofficial].
   c. Cast visibility rays with origins uniformly sampled on the face for each direction to estimate average visibility [S1, unofficial].
   d. Project the resulting visibility map to the Haar wavelet basis [S1, unofficial].
   e. Store the wavelet coefficients. The angular resolution for this precomputation is typically $64^2$ to accurately reconstruct the high-frequency silhouette signal [S1, unofficial].
3. **Pixel Intensity Integration**: For a given pixel with footprint $P$ and direction $\boldsymbol{\nu}_p$:
   a. Determine the subset $P^+ \subseteq P$ such that rays originating from points in $P^+$ intersect the scene aggregate primitives (specifically, the union of subsets $P_k^+$ covered by each voxel's primitive $B_k$) [S1, unofficial].
   b. The remaining subset is $P^- = P \setminus P^+$ [S1, unofficial].
   c. Evaluate the incoming radiance $L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p)$ for each point $\boldsymbol{x}_p \in P$:
      - If $\boldsymbol{x}_p \in P^-$, $L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p) = L_b(-\boldsymbol{\nu}_p)$, where $L_b$ is the background radiance (assumed constant due to the far-field assumption) [S1, unofficial].
      - If $\boldsymbol{x}_p \in P^+$, $L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p)$ is evaluated based on the scene aggregate's outgoing radiance, utilizing the precomputed Aggregated Interior Visibility (AIV) and Aggregated Boundary Visibility (ABV) terms to handle inter-voxel correlations and transparency [S1, unofficial].
   d. Compute the pixel intensity $I$ by integrating the incoming radiance over the entire footprint $P$:
      $$ I = \frac{1}{|P|} \int_P L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p) d\boldsymbol{x}_p $$
      where $|P|$ is the area of the pixel footprint [S1, unofficial].

**Formulas**
- **Aggregated Boundary Visibility (ABV)**:
  $$ \hat{V}_b(\boldsymbol{\nu}; P^+) = \frac{1}{|P|} \int_{P^+} V(\boldsymbol{x}, \boldsymbol{\nu}) d\boldsymbol{x} $$
  where $V(\boldsymbol{x}, \boldsymbol{\nu})$ is the visibility at point $\boldsymbol{x}$ in direction $\boldsymbol{\nu}$, and the integral is restricted to $P^+$, the subset of the pixel footprint covered by scene primitives [S1, unofficial].
- **Pixel Intensity**:
  $$ I = \frac{1}{|P|} \int_P L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p) d\boldsymbol{x}_p $$
  where $L_i$ is the incoming radiance [S1, unofficial].

**Parameters**
- **Angular Resolution**: $64^2$ samples per hemisphere for ABV precomputation [S1, unofficial].
- **Mapping**: Concentric mapping for hemisphere parameterization [S1, unofficial].
- **Basis**: Haar wavelet basis for storing visibility maps [S1, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: The ABV term contains high-frequency signals responsible for reconstructing the scene silhouette; insufficient angular resolution (e.g., lower than $64^2$) may lead to inaccurate silhouette reconstruction [S1, unofficial].
- **Boundaries**: The definition of $P^+$ relies on the intersection of rays with truncated ellipsoid primitives; points outside $P^+$ are guaranteed not to be covered by the underlying geometry, ensuring correct background blending [S1, unofficial].
- **Far-Field Assumption**: The background radiance $L_b$ is assumed constant across the pixel footprint; this may not hold for near-field backgrounds or complex lighting [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Precomputation is offline and depends on the number of boundary faces and the angular resolution ($64^2$). Runtime evaluation involves integrating radiance over the pixel footprint, which is replaced by querying precomputed visibility terms, avoiding order-dependent ray marching [S1, unofficial].
- **Memory Complexity**: Storage of ABV and AIV terms in Haar wavelet basis for all boundary faces and voxels. Nonlinear approximation and truncation of wavelet coefficients can be used for compression [S1, unofficial].
- **Strengths**: Handles inter-voxel correlation correctly; blends with background without order-dependency; accurate silhouette reconstruction with high angular resolution [S1, unofficial].
- **Limitations**: Requires offline precomputation; relies on far-field background assumption; complexity in determining $P^+$ for complex geometries [S1, unofficial].
- **When to use**: Level-of-detail rendering where accurate appearance aggregation and background blending are critical, and offline precomputation is acceptable [S1, unofficial].
- **When not to use**: Real-time applications requiring dynamic scene changes without precomputation; scenes with near-field backgrounds where the far-field assumption fails [S1, unofficial].

## Background
The method was introduced in "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" [S1, unofficial]. It addresses the limitations of traditional volumetric models that cannot distinguish between different types of inter-voxel correlation, leading to incorrect accumulation results [S1, unofficial]. The approach uses aggregated visibility functions (AIV and ABV) to preserve the notion of semi-transparency after geometry abstraction [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
