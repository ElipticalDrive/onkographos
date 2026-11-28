# Progressive Coarse-to-Fine Training Strategy

Also called: Training Strategy

## Overview
A training methodology that decouples the recovery of global geometry from the refinement of fine details and appearance, typically applied in dynamic or high-fidelity volume rendering. It sets itself apart from monolithic training by first establishing a coherent low-resolution structure (coarse stage) before optimizing high-resolution features and directional dependencies (fine stage), thereby improving convergence stability and final reconstruction quality [S1][S2, unofficial].

## How it works
The strategy consists of two distinct phases: a coarse stage to recover global shape and a fine stage to refine details and appearance.

**Inputs and Outputs:**
*   **Inputs:** Training images (potentially with time indices for dynamic scenes), camera poses, and a proxy geometry or bounding box estimate.
*   **Outputs:** A final neural field model (e.g., SDF and color fields) capable of rendering high-quality views.

**Data Structures:**
*   **Voxel Grid:** A 3D grid storing density or SDF values. Resolution varies by stage (e.g., $128^3$ for coarse, $256^3$ for fine) [S2, unofficial].
*   **Color Features:**
    *   Coarse Stage: Scalar color values (no directional dependency) [S1].
    *   Fine Stage: High-dimensional feature vectors allowing for view-dependent shading [S1].
*   **Bounding Box:** A tight bounding box computed from proxy geometry to filter empty space during the fine stage [S1].

**Numbered Steps:**

1.  **Coarse Stage Initialization:**
    *   Set the voxel grid resolution to a low value (e.g., $128^3$) [S2, unofficial].
    *   Configure the color representation to be view-independent (color itself) to simplify the optimization landscape [S1].
    *   If the scene is dynamic, employ **Progressive Training**: Start by training only on images close to the canonical time step. Gradually add images with larger time offsets until all training data is included [S1].
    *   Optimize the model to recover a coherent global shape.
    *   *Optional:* Apply a 3D convolution with a Gaussian kernel during this stage or as post-processing to smooth the geometry. This reduces Chamfer Distance (CD) error from 0.74 to 0.72 [S2, unofficial].

2.  **Transition:**
    *   Compute a tighter bounding box using the proxy geometry learned during the coarse stage. This filters out empty space, reducing the computational load for the fine stage [S1].

3.  **Fine Stage Optimization:**
    *   Increase the voxel grid resolution to a high value (default $256^3$) [S2, unofficial].
    *   Switch the color representation to high-dimensional features to model ray direction dependency (e.g., via a network component $F_{\theta4}$) [S1].
    *   Train the model using the tight bounding box to recover fine geometric details and view-dependent appearance while maintaining the structural consistency established in Stage 1 [S2, unofficial].

**Formulas and Parameters:**
*   **Loss Function (Dynamic Scenes):** The total loss combines multiple terms [S1]:
    $$ \mathcal{L} = w_0 \mathcal{L}_{rgb} + w_1 \mathcal{L}_{ptc} + w_2 \mathcal{L}_{bg} + w_3 \mathcal{L}_{flow} + w_4 \mathcal{L}_{vdiff} + w_5 \mathcal{L}_{tv}(\mathbf{V}_{grid}^{\sigma_{\text{Can}}}) + w_6 \mathcal{L}_{tv}(\mathbf{V}_{grid}^{\text{flow}_t}) + w_7 \mathcal{L}_{tv}(\mathbf{D}) $$
    Where $w_i$ are weights, $\mathcal{L}_{rgb}$ is the image reconstruction loss, $\mathcal{L}_{ptc}$ is point cloud consistency, $\mathcal{L}_{flow}$ is flow consistency, $\mathcal{L}_{vdiff}$ is video diffusion loss, and $\mathcal{L}_{tv}$ are total variation regularizations on density, flow, and displacement fields.
*   **Resolution Scaling:** Training time scales with voxel grid resolution. Measured times for a specific implementation: $128^3$ takes 11 mins, $192^3$ takes 12 mins, $256^3$ takes 15 mins, and $320^3$ takes 17 mins [S2, unofficial]. The default resolution is $256^3$ as it offers the best trade-off between CD error and cost [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gaussian Kernel Removal:** Removing the Gaussian kernel during coarse training increases CD error to 0.74; removing it during post-processing slightly increases it to 0.715 from 0.720 [S2, unofficial].
*   **Resolution Diminishing Returns:** Increasing resolution from $256^3$ to $320^3$ yields similar CD performance but at a higher training cost [S2, unofficial].

## Subtopics

### Two-Stage Training Pipeline
A specific implementation of the coarse-to-fine strategy where the process is strictly divided into Stage 1 (coarse training for global shape) and Stage 2 (fine training for details). It is chosen when computational resources allow for sequential optimization rather than joint training. It sets itself apart by explicitly monitoring metrics like Chamfer Distance at the end of each stage to verify structural coherence before proceeding [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time increases with voxel resolution. For voxel-based methods, time scales roughly linearly with resolution steps (e.g., 11–17 minutes for resolutions $128^3$ to $320^3$ in reported benchmarks) [S2, unofficial].
*   **Memory Complexity:** Proportional to the voxel grid resolution. The fine stage requires significantly more memory due to the higher resolution grid ($256^3$ vs $128^3$) [S2, unofficial].
*   **Performance:** Improves reconstruction quality (lower Chamfer Distance) and visual fidelity compared to single-stage training. The coarse stage ensures global shape coherence, while the fine stage captures details [S2, unofficial].
*   **Strengths:** Stabilizes training for complex geometries; prevents local minima associated with high-frequency details; allows for efficient bounding box culling in the fine stage [S1][S2, unofficial].
*   **Limitations:** Requires two distinct optimization phases; careful tuning of the transition (bounding box computation) is required; Gaussian smoothing may sacrifice minor quantitative metrics for visual quality [S2, unofficial].
*   **When to use:** High-fidelity surface reconstruction or dynamic scenes where global shape is difficult to converge directly [S1][S2, unofficial].
*   **When not to use:** Real-time inference applications where training time is prohibitive, or simple scenes where a single resolution suffices.

## Background
The coarse-to-fine strategy originates from the need to stabilize the optimization of neural radiance fields and implicit surfaces, which are prone to local minima when optimizing high-frequency details from the start. The "Two-Stage Training Pipeline" was explicitly detailed in Voxurf for efficient neural surface reconstruction [S2, unofficial]. The "Progressive Training" aspect (adding time steps gradually) was introduced in Forward Flow for dynamic scenes to handle temporal consistency [S1].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
