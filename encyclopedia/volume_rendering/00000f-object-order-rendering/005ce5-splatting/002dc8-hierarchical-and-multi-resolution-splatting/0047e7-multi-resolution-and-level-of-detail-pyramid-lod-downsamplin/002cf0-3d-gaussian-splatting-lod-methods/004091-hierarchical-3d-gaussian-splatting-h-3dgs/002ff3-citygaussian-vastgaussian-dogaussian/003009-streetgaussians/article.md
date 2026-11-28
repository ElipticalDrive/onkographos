# StreetGaussians
## Overview
StreetGaussians is a method for reconstructing dynamic urban driving scenes by decomposing the environment into a static background and dynamic actors (vehicles, pedestrians) [S1, unofficial][S3, unofficial]. It solves the problem of efficiently rendering complex, moving traffic in large-scale outdoor environments by associating each dynamic object with a separate 3D Gaussian Splatting (3DGS) sub-model that can be transformed independently [S1, unofficial]. It sets itself apart from monolithic 3DGS approaches by explicitly separating static and dynamic components, allowing for independent pose updates of actors without retraining the background [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Multi-view imagery of a driving scene and tracked trajectories/poses for dynamic objects (vehicles, pedestrians) [S3, unofficial].
- **Outputs:** A static 3DGS representation of the background and a set of dynamic 3DGS sub-models, each associated with a specific actor and its time-varying pose [S1, unofficial][S3, unofficial].

**Data Structures:**
1. **Static 3DGS:** Standard 3DGS representation for the background [S3, unofficial].
2. **Dynamic Actor Sub-models:** For each dynamic object $i$:
   - A separate 3DGS representation [S1, unofficial].
   - Standard 3DGS parameters (means, covariances, opacities, SH coefficients) apply to each sub-model [G].
   - A tracked pose $T_i(t)$ representing the object's transformation over time [S3, unofficial].
   - Time-dependent SH coefficients $c_{lm,i}(t)$ to model appearance changes [S3, unofficial].

**Steps:**
1. Incrementally optimize static 3D Gaussians for the background using input views [S3, unofficial]. This process may be assisted by Segment Anything Model (SAM) masks and LiDAR depth data [S3, unofficial].
2. Identify dynamic objects (vehicles, pedestrians) using tracking or segmentation [S3, unofficial].
3. Train a separate 3DGS model for each dynamic object [S1, unofficial]. The Gaussians are transformed by the tracked vehicle/actor poses $T_i(t)$ [S3, unofficial].
4. Approximate the appearance of dynamic objects using time-related spherical harmonic coefficients $c_{lm}(t)$ [S3, unofficial].
5. Composite the static background and dynamic object renders to produce the final frame [S3, unofficial].

**Formulas:**
- The position of a Gaussian $g$ in object $i$'s local frame is transformed to the world frame at time $t$ via the tracked pose $T_i(t)$ [S3, unofficial].
- The final pixel color is computed by compositing the static background and all dynamic objects, where each dynamic object's contribution uses its transformed Gaussians and time-dependent SH coefficients [S3, unofficial].

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- Not given in the sources.

## Subtopics

### [DrivingGaussian](7348-drivinggaussian.md)
Reconstructs dynamic driving data by compositing surrounding views from multiple cameras into a single dynamic scene. It aligns Gaussian clouds of different vehicles with tracked trajectories to model dynamic urban environments. It builds on StreetGaussians by adopting a similar static/dynamic separation strategy but focuses on composite reconstruction assisted by external priors.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Efficiently handles dynamic scenes by separating static and dynamic components; allows independent updates of dynamic actors; suitable for autonomous driving simulation [S2, unofficial].
- **Limitations:** Requires accurate tracking of dynamic objects to maintain pose alignment [S3, unofficial].
- **When to use:** Dynamic urban driving scenes with distinct, trackable vehicles and pedestrians [S1, unofficial].
- **When not to use:** Static scenes or scenes with untracked, highly dynamic elements [S3, unofficial].

## Background
StreetGaussians was introduced to address the challenges of reconstructing dynamic urban environments for autonomous driving applications [S1, unofficial]. It builds on the success of 3DGS but adapts it to handle motion by decomposing the scene into static and dynamic components [S3, unofficial]. The method is part of a broader trend in 2026 towards per-actor 3DGS for AV simulation, allowing for the insertion of rare-event vehicles into any scene [S2, unofficial]. It is related to other methods like DrivingGaussian, which composites surrounding views from multiple cameras into a single dynamic scene, assisted by external priors [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
