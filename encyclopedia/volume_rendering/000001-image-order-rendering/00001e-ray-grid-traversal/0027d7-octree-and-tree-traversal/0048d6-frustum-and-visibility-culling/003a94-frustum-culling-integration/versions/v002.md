# Frustum Culling Integration

Also called: Base+Frustum culling

## Overview
Frustum Culling Integration is a technique used in progressive volume rendering (specifically for Gaussian Splatting) to prioritize the loading and rendering of splats visible to the user by applying frustum culling [S1]. It is applied after global ordering is determined to refine the selection of data based on the current viewpoint [S1]. It sets itself apart from the base octree-only approach by leveraging user-specific view information to improve foreground quality, whereas the octree-only method provides a good scene representation without requiring such knowledge [S1]. It builds on [Frustum and Visibility Culling](12355-frustum-and-visibility-culling.md) by adding view-dependent filtering to the ordering pipeline.

## How it works
The method integrates with an existing global ordering (e.g., based on contribution or octree depth) to filter or prioritize splats based on their visibility within the viewing frustum [S1].

**Inputs and Outputs:**
*   **Input:** A list of splats (Gaussians) with their attributes, the current camera view, and the existing global ordering [S1, G].
*   **Output:** A filtered or prioritized subset of splats for the current frame [S1].

**Data Structures:**
*   **Splat List:** An array of splat objects containing position, covariance, opacity, and color attributes [G]. Exact field layouts and bit packing are Not given in the sources.
*   **Frustum State:** The current viewing frustum boundaries and the applied margin percentage [S1].

**Steps:**
1.  **Projection:** Project the mean position of each splat onto the image plane [S1].
2.  **Margin Expansion:** Increase the frustum boundaries by a specified margin percentage to account for Gaussian area coverage [S1].
3.  **Filtering:** Filter out splats whose projected positions fall outside the expanded frustum boundaries [S1].
4.  **Selection:** Select the remaining splats for high-priority loading or rendering [S1].

**Formulas:**
Let the original frustum boundaries in projected space be $[x_{min}, x_{max}] \times [y_{min}, y_{max}]$ and the margin percentage be $m$. The expanded boundaries are calculated as:
$$x'_{min} = x_{min} - m \cdot (x_{max} - x_{min}), \quad x'_{max} = x_{max} + m \cdot (x_{max} - x_{min})$$
$$y'_{min} = y_{min} - m \cdot (y_{max} - y_{min}), \quad y'_{max} = y_{max} + m \cdot (y_{max} - y_{min})$$
A splat is kept if its projected center $(u, v)$ satisfies $x'_{min} \le u \le x'_{max}$ and $y'_{min} \le v \le y'_{max}$ [S1].

**Parameters:**
*   **Frustum Margin:** The percentage by which frustum boundaries are increased. Typical value: 30% [S1].

**Edge Cases and Pitfalls:**
*   **Late Application:** Frustum culling is applied after global ordering is determined, so it does not account for occlusions from the current viewpoint [S1].
*   **Visibility Threshold:** If more than 10% of splats are visible within the current frustum, small background splats may still not be selected [S1].
*   **Area Coverage:** Since Gaussians cover areas rather than points, a splat whose mean falls outside the frustum might still contribute to the image; the margin mitigates this but strict point-based culling can miss contributing splats [S1].

## Subtopics

### In-frustum prioritization
A technique to fine-tune ordering based on the user's initial pose, selecting a percentage of splats from within the viewing frustum to allow rapid movement [S1]. It projects all splats onto the image plane and filters those falling outside the frustum [S1]. It uses a frustum margin (typically 30%) to account for Gaussian area coverage [S1]. This variant is chosen when rapid movement and focusing on visible content are prioritized over global scene completeness [S1].

## Operating characteristics
*   **Performance:** Improves foreground quality and perceived responsiveness when user-specific view information is available [S1].
*   **Limitations:** Does not account for occlusions because it operates after global ordering [S1]. If the visible splat count exceeds 10% of the total, background splats may be neglected [S1].
*   **Comparison:** The complete approach (octree + frustum culling) performs better than the octree-only approach when view information is accessible, but the octree-only approach is robust without it [S1].

## Background
This technique is part of the PRoGS (Progressive Rendering of Gaussian Splats) framework, which brings progressive rendering of 3D Gaussian Splats into an academic context [S1]. It builds upon global ordering strategies (such as those based on contribution or octree depth) and refines them using view-dependent visibility [S1].

## Sources
[S1] PRoGS: Progressive Rendering of Gaussian Splats, https://doi.org/10.1109/wacv61041.2025.00308, fetched 2026-10-26
