# DrivingGaussian

Also called: A method that composites surrounding views from multiple cameras into a single dynamic scene, aligning the Gaussian clouds of different vehicles with their tracked trajectories

## Overview
DrivingGaussian is a method for reconstructing dynamic driving data by compositing surrounding views from multiple cameras into a single dynamic scene [S1, unofficial]. It specifically addresses the challenge of aligning the Gaussian clouds of different vehicles with their tracked trajectories to model dynamic urban environments [S1, unofficial]. It builds on [StreetGaussians](6161-streetgaussians.md) by adopting a similar strategy of separating static and dynamic components, but focuses on the composite reconstruction process assisted by external priors [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-camera image sequences, input LiDAR depth data, and tracked trajectories for dynamic objects (vehicles/pedestrians) [S1, unofficial][S2, unofficial].
*   **Outputs:** A single dynamic scene representation composed of static 3D Gaussians and dynamic object Gaussians aligned with trajectories [S1, unofficial].

**Data Structures:**
The method maintains two distinct sets of 3D Gaussians:
1.  **Static Gaussians:** Representing the fixed urban background (roads, buildings, etc.).
2.  **Dynamic Gaussians:** Representing individual vehicles or pedestrians. Each dynamic object is associated with a separate 3DGS sub-model [S1, unofficial].

**Steps:**
1.  **Static Optimization:** Incrementally optimize the static 3D Gaussians using the multi-camera inputs [S2, unofficial].
2.  **Dynamic Composition:** Compose the static scene with dynamic objects' 3D Gaussians [S2, unofficial].
3.  **Trajectory Alignment:** Align the Gaussian clouds of different vehicles with their tracked trajectories [S1, unofficial].
4.  **Assisted Reconstruction:** Utilize the Segment Anything Model (SAM) and input LiDAR depth data to assist in the segmentation and depth estimation of dynamic objects [S2, unofficial].

**Parameters:**
*   **Tracked Trajectories:** Required for each dynamic object to define its motion and pose over time [S1, unofficial].
*   **LiDAR Depth:** Used as a geometric prior [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Object Segmentation:** Accurate separation of dynamic objects from the static background is critical; errors here lead to misalignment. SAM is used to mitigate this [S2, unofficial].
*   **Trajectory Accuracy:** Misaligned trajectories result in incorrect Gaussian placement and motion artifacts [S1, unofficial].

## Subtopics

### Time-Dependent Gaussian Parameters (Life Peak)
A method for urban scene data that makes Gaussians' mean and opacity values time-dependent functions centered at the corresponding Gaussian's life peak [S2, unofficial]. The **Life peak** is defined as the maximum prominence over time [S2, unofficial]. This approach allows for modeling the temporal evolution of dynamic objects within the scene [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables reconstruction of complex dynamic driving scenes by separating static and dynamic elements, leveraging external priors (LiDAR, SAM) for robustness [S1, unofficial][S2, unofficial].
*   **Limitations:** Relies on accurate tracked trajectories and external models (SAM, LiDAR), which may not be available in all scenarios [S1, unofficial][S2, unofficial].
*   **When to use:** When reconstructing dynamic urban driving scenes with multi-camera setups and available trajectory/LiDAR data [S1, unofficial][S2, unofficial].
*   **When not to use:** For static scene reconstruction or scenes without reliable dynamic object tracking [S1, unofficial].

## Background
DrivingGaussian was introduced as a method to reconstruct dynamic driving data by compositing surrounding views from multiple cameras into a single dynamic scene [S1, unofficial][S2, unofficial]. It aligns the Gaussian clouds of different vehicles with their tracked trajectories, building upon the concepts of modeling dynamic urban environments with separate sub-models for each vehicle or pedestrian [S1, unofficial]. The method is assisted by the Segment Anything Model and input LiDAR depth data to improve reconstruction quality [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
