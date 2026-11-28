# StreetGaussians


## Overview
StreetGaussians is a method for reconstructing dynamic urban driving scenes by decomposing the environment into a static background and dynamic actors (vehicles, pedestrians) [S1, unofficial][S3, unofficial]. It solves the problem of efficiently rendering complex, moving traffic in large-scale outdoor environments by associating each dynamic object with a separate 3D Gaussian Splatting (3DGS) sub-model that can be transformed independently [S1, unofficial]. It sets itself apart from monolithic 3DGS approaches by explicitly separating static and dynamic components, allowing for independent pose updates of actors without retraining the background [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Multi-view video/images of a driving scene, LiDAR depth data (optional but used for initialization), and tracked trajectories/poses for dynamic objects (vehicles, pedestrians) [S3, unofficial].
- **Outputs:** A static 3DGS representation of the background and a set of dynamic 3DGS sub-models, each associated with a specific actor and its time-varying pose [S1, unofficial][S3, unofficial].

**Data Structures:**
1.  **Static 3DGS:** Standard 3DGS representation containing 3D mean positions $\mu$, covariance matrices $\Sigma$ (or rotation $R$ and scale $S$), opacity $\alpha$, and spherical harmonic (SH) coefficients $c_{lm}$ for the static background [S3, unofficial].
2.  **Dynamic Actor Sub-models:** For each dynamic object $i$:
    -   A separate 3DGS representation with its own $\mu_i, \Sigma_i, \alpha_i, c_{lm,i}$ [S1, unofficial].
    -   A tracked pose $T_i(t)$ (transformation matrix) for the object at time $t$ [S3, unofficial].
    -   Time-dependent SH coefficients $c_{lm,i}(t)$ to model appearance changes [S3, unofficial].

**Steps:**
1.  **Static Background Reconstruction:** Incrementally optimize static 3D Gaussians for the background using input views [S3, unofficial]. This may be assisted by Segment Anything Model (SAM) masks and LiDAR depth data to separate static elements [S3, unofficial].
2.  **Dynamic Object Isolation:** Identify dynamic objects (vehicles, pedestrians) using tracking or segmentation [S3, unofficial].
3.  **Dynamic Sub-model Training:** For each dynamic object, train a separate 3DGS model. The Gaussians are transformed by the tracked vehicle/actor poses $T_i(t)$ [S3, unofficial].
4.  **Appearance Modeling:** Approximate the appearance of dynamic objects using time-related spherical harmonic coefficients $c_{lm}(t)$ [S3, unofficial]. This allows the model to capture lighting changes or motion blur effects over time.
5.  **Rendering:** To render a frame at time $t$:
    -   Render the static background using the static 3DGS.
    -   For each dynamic object $i$, transform its Gaussians by $T_i(t)$ and render using its time-dependent SH coefficients $c_{lm,i}(t)$.
    -   Composite the results [S1, unofficial][S3, unofficial].

**Formulas:**
-   The position of a Gaussian $g$ in object $i$'s local frame is $\mu_{local}$. Its world position at time $t$ is:
    $$ \mu_{world}(t) = T_i(t) \cdot \mu_{local} $$
    where $T_i(t)$ is the 4x4 transformation matrix for object $i$ at time $t$ [S3, unofficial].
-   The color $C$ of a pixel is computed by compositing the static background and all dynamic objects, where each dynamic object's contribution uses its transformed Gaussians and time-dependent SH coefficients [S3, unofficial].

**Parameters:**
-   Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Tracking Errors:** Inaccurate tracked poses $T_i(t)$ will lead to misaligned Gaussians and artifacts [S3, unofficial].
-   **Occlusion Handling:** Dynamic objects may occlude the static background or other dynamic objects. The compositing order must be handled correctly [S1, unofficial].
-   **Dynamic Object Identification:** Correctly segmenting and tracking all dynamic objects is critical. Missing objects or false positives will degrade quality [S3, unofficial].

## Subtopics

## Operating characteristics
-   **Time Complexity:** Training time is reduced compared to training a single large 3DGS for the entire dynamic scene, as static and dynamic parts are optimized separately [S3, unofficial]. Rendering time depends on the number of dynamic objects and their Gaussians.
-   **Memory Complexity:** Lower memory usage for the background, as it is static and can be optimized independently. Dynamic objects require storing separate Gaussians and pose data [S1, unofficial].
-   **Strengths:** Efficiently handles dynamic scenes; allows independent updates of dynamic actors; suitable for autonomous driving simulation [S2, unofficial].
-   **Limitations:** Requires accurate tracking of dynamic objects; may struggle with highly dynamic or cluttered scenes where object boundaries are unclear [S3, unofficial].
-   **When to use:** Dynamic urban driving scenes with distinct, trackable vehicles and pedestrians [S1, unofficial].
-   **When not to use:** Static scenes or scenes with untracked, highly dynamic elements [S3, unofficial].

## Background
StreetGaussians was introduced to address the challenges of reconstructing dynamic urban environments for autonomous driving applications [S1, unofficial]. It builds on the success of 3DGS but adapts it to handle motion by decomposing the scene into static and dynamic components [S3, unofficial]. The method is part of a broader trend in 2026 towards per-actor 3DGS for AV simulation, allowing for the insertion of rare-event vehicles into any scene [S2, unofficial]. It is related to other methods like DrivingGaussian, which composites surrounding views from multiple cameras into a single dynamic scene, assisted by external priors [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
