# Textured Mesh Refinement

Also called: Deep Shading, Mesh Rendering Refinement

## Overview
Textured Mesh Refinement is a neural rendering baseline that renders a geometric mesh approximation of a scene and refines the resulting image using a neural network to recover missing details and correct shading errors [S2]. It differs from voxel-based rerendering methods by using a mesh representation to estimate UV maps via projective texturing, though it faces similar limitations regarding GAN artifacts and shadow estimation [S1].

## How it works
**Inputs:**
1. A textured mesh of the scene (vertices, faces, and texture coordinates).
2. Target camera viewpoint parameters.
3. Light position.
4. Target image $T$ (for optimization/training).

**Outputs:**
Refined image $I_{refined}$ that approximates the target image $T$, with missing regions completed and fine details enhanced [S2].

**Data Structures:**
*   **Mesh:** Defined by vertex positions, face connectivity, and UV coordinates.
*   **Buffers:** Diffuse color buffer and depth buffer rendered from the mesh.
*   **Network:** A Pix2Pix network [S1] (or similar image-to-image translation network) takes the rendered buffers and light position as input channels.

**Steps:**
1.  **Estimate UV Map:** Use projective texturing to estimate the UV map of the mesh relative to the target camera viewpoint [S1].
2.  **Position Mesh:** Place the mesh in the desired position corresponding to the target camera view [S1].
3.  **Render Buffers:** Render the scene to generate:
    *   Diffuse color buffer $C_{diffuse}$.
    *   Depth buffer $D$.
4.  **Prepare Input Tensor:** Concatenate $C_{diffuse}$, $D$, and the light position (as an additional channel or encoded feature) to form the input tensor $I_{input}$ [S1].
5.  **Neural Refinement:** Pass $I_{input}$ through a neural network (e.g., Pix2Pix) to predict the refined image $I_{refined}$ [S1, S2].
6.  **Optimization:** Optimize the network parameters to minimize the difference between $I_{refined}$ and the target image $T$ [S1].

**Edge Cases and Pitfalls:**
*   **Missing Regions:** The re-projected textured mesh may have missing regions (e.g., due to occlusion or incomplete reconstruction) which the network must complete [S2].
*   **Artifacts:** The method may produce typical GAN artifacts and fail to estimate shadows properly, similar to Image Translation models [S1].
*   **Detail Loss:** While it completes missing regions, the refined result may have less detail compared to more advanced methods like Neural Point Cloud Rendering [S2].

## Operating characteristics
*   **Performance:** On 'scene0010 00', the PSNR and SSIM of the refined images (re-projected textured mesh) are 22.208 and 0.86, respectively [S2]. This is lower than the proposed Neural Point Cloud Rendering method (PSNR 24.421, SSIM 0.901) [S2].
*   **Strengths:** Can complete missing regions in the re-projected textured mesh [S2].
*   **Limitations:** Faces similar limitations to Image Translation, including GAN artifacts and poor shadow estimation, despite using a mesh representation instead of voxels [S1]. The results may lack fine details compared to other neural rendering approaches [S2].

## Background
Textured Mesh Refinement is used as a comparison method in neural rendering research to evaluate the effectiveness of mesh-based representations versus point cloud or voxel-based representations [S2]. It is also referred to as "Deep Shading" in some contexts, where it is compared against voxel-based rerendering methods [S1].

## Subtopics

### Deep Shading
*   **What it is:** A specific implementation of Textured Mesh Refinement that uses projective texturing to estimate UV maps and a Pix2Pix network for refinement [S1].
*   **Differentiation:** Uses a mesh representation instead of voxels [S1].
*   **When to choose:** When a mesh representation is preferred over voxels, but similar limitations to image translation apply [S1].
*   **Link:** [Textured Mesh Refinement](12627-textured-mesh-refinement.md)

## Sources
[S1] Neural Voxel Renderer: Learning an Accurate and Controllable Rendering Tool, https://doi.org/10.1109/cvpr42600.2020.00546, fetched 2026-10-26
[S2] Neural Point Cloud Rendering via Multi-Plane Projection, https://doi.org/10.1109/cvpr42600.2020.00785, fetched 2026-10-30
