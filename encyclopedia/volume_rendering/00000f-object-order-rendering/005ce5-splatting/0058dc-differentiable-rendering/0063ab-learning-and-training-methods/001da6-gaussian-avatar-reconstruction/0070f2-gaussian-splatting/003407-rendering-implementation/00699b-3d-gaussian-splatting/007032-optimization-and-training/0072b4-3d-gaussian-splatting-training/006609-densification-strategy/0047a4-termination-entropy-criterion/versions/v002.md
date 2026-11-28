# Termination entropy criterion


## Overview
The termination entropy criterion is a metric used to calculate information gain in active 3D reconstruction [S1, unofficial]. It guides viewpoint selection by quantifying scene uncertainty, distinguishing itself from error-magnitude or density-based strategies by focusing on information saturation rather than reconstruction error [S1, unofficial].

## How it works
**Inputs and Outputs:** Not given in the sources.
**Data Structures:** Not given in the sources.
**Steps:** Not given in the sources. The criterion is applied to calculate information gain for active 3D reconstruction decisions [S1, unofficial].
**Formulas:** Not given in the sources.
**Parameters:** Not given in the sources.
**Edge Cases and Pitfalls:** Not given in the sources.
**Pseudocode:** Not provided in the sources.

## Subtopics

### [SAES (Sliding-Window-Averaged Early Stopping)](17544-saes-sliding-window-averaged-early-stopping.md)
An intrinsic training termination criterion that prevents over-training and "needle-like" primitive accumulation in sparse-view reconstruction by monitoring Gaussian population growth dynamics rather than external volumetric metrics. Choose it when early stopping based on internal structural saturation is preferred over PSNR-based stopping.

## Operating characteristics
*   **Performance:** The method performs consistently better than baselines across all tested scenes, enabling agents to explore more diverse viewpoints and construct higher-quality 3D models with a limited number of observations [S1, unofficial].
*   **Time and Memory Complexity:** Not given in the sources.
*   **Strengths and Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The criterion was introduced in the ActiveRMAP study for active mapping and planning [S1, unofficial]. It was evaluated on the NeRF-Synthetic and Tanks & Temple datasets to guide camera agents in selecting viewpoints that maximize information gain [S1, unofficial].

## Sources
[S1] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
