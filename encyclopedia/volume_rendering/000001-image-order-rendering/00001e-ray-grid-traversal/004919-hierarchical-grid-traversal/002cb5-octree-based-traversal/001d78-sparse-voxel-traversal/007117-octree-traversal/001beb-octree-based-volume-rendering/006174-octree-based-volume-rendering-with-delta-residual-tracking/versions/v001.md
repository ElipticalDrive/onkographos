# Octree-based Volume Rendering with Delta/Residual Tracking

Also called: Delta/residual tracking with octree, Octree-based Volume Traversal, Ratio tracking for volume rendering

## Overview
This method replaces traditional volume meshes and volume stacks with an octree encoding volume topology and extra data, enabling direct traversal and shading during ray marching [S1, unofficial]. It differs from stack-based volume shading by evaluating all volume shaders in the current leaf node simultaneously at each step, eliminating the need to maintain and update a volume stack during traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of objects with volume geometry; volume bounding boxes.
*   **Output:** Rendered image with volume shading, emission, and direct light (note: early implementations had missing emission and direct light) [S1, unofficial].

**Data Structures:**
*   **Octree Leaf Node:**
    *   `volumes`: A list of active volume objects. The source specifies a fixed size of 1024 objects, but notes this consumes excessive memory and suggests a dynamic pointer to a packed list as an alternative [S1, unofficial].
    *   `min_density`: Minimum density value for the volumes in this leaf.
    *   `max_density`: Maximum density value for the volumes in this leaf.
    *   *Note:* The min/max density is used to bound null scattering probability [S1, unofficial].

**Numbered Steps:**
1.  **Build Octree:** Construct an octree from the list of objects possessing volume geometry, using their volume bounding boxes [S1, unofficial].
2.  **Scene Traversal:** Execute the standard scene intersection (`scene_intersect`).
3.  **Octree Traversal:** After `scene_intersect` in `integrator_intersect_closest`, perform octree traversal to determine the active leaf node for the ray segment [S1, unofficial].
4.  **Volume Shading:** In `shade_volume`, traverse the octree to identify the current leaf node. Evaluate all volume shaders in the current leaf node at once [S1, unofficial].
5.  **Null Scattering:** Use the stored `min_density` and `max_density` values to bound the null scattering probability [S1, unofficial].

**Formulas:**
*   Null scattering probability is bounded by the actual volume density using the leaf node's min/max values [S1, unofficial]. Specific formulas are not provided in the sources.

**Parameters:**
*   Fixed list size for volumes per leaf node: 1024 (noted as excessive) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** A fixed list size of 1024 volume objects per leaf node consumes excessive memory [S1, unofficial].
*   **Null Scattering Efficiency:** Computing min/max density over the entire domain yields a minimum
