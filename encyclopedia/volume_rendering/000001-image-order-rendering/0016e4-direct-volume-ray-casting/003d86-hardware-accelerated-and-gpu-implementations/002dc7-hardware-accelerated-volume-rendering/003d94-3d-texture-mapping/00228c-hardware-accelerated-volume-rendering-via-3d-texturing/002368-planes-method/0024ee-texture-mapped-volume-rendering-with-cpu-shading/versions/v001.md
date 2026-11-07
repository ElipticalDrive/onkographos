# Texture-Mapped Volume Rendering with CPU Shading

Also called: Hardware resampling with software shading, Our architecture

## Overview
This method renders regular volumetric grids by leveraging 3D texture mapping hardware to resample density and pre-computed gradients, then transferring the raw sample data to main memory for CPU-based Phong shading and compositing [S1]. It solves the problem of view-dependent shading in texture-mapped rendering by avoiding the need to re-shade and reload textures for every frame change, and by circumventing the infeasibly large lookup tables required by post-texturing gradient indexing [S1]. Unlike [Planes Method](3027-planes-method.md), which may rely on hardware blending for final compositing without explicit CPU shading, this method explicitly offloads the lighting calculation to the CPU to support arbitrary dynamic lights and non-linear transfer functions without hardware interpolation artifacts [S1].

## How it works
**Inputs:**
*   A regular 3D volume grid of density values $D$.
*   Pre-computed gradients $\nabla D$ for each voxel (or computed on-the-fly if hardware supports gradient texture units, though notes imply pre-computation/storage).
*   Camera parameters (view direction, projection type).
*   Lighting parameters (light position, color, material properties for Phong shading).
*   Transfer function parameters (mapping density to opacity $\alpha$).

**Outputs:**
*   A 2D image where each pixel contains the accumulated color and opacity from the volume.

**Data Structures:**
1.  **Texture Memory:** A 3D texture object storing density values and their corresponding gradients. The source notes specify storing "density and pre-computed gradients as a solid texture" [S1].
    *   *Layout:* Not explicitly defined in bits, but typically stored as a 3D texture array or a 3D texture with multiple components (e.g., RGBA where RGB might store gradient components and A stores density, or separate textures). The source mentions "32-bits per voxel" for density storage in the context of memory size [S1].
2.  **Sample Buffer:** A buffer in main memory (CPU-accessible) to store the resampled density and gradient values for each sample point along each ray.
3.  **Frame Buffer:** Standard display buffer for final image output.

**Steps:**
1.  **Load Data:** Store the volume density and pre-computed gradients into the graphics hardware's 3D texture memory [S1]. This is done once per volume dataset, not per frame.
2.  **Resample:** Use hardware texture interpolation to resample the 3D texture. This is achieved by rendering planes parallel to the image plane (or along rays) through the volume [S1]. The hardware interpolates the density and gradient values at the sample points defined by the ray casting geometry.
3.  **Transfer:** Read back the resampled sample data (density and gradient values) from the texture memory (or the rendered output buffer) into a buffer in main memory [S1].
4.  **Shade:** For each sample point in the buffer, compute the shading on the CPU.
    *   Estimate the surface normal $\mathbf{n}$ from the interpolated gradient $\nabla D$.
    *   Apply the full Phong lighting equation using the estimated normal, light direction, view direction, and material properties [S1].
    *   Map the density value to opacity $\alpha$ using the transfer function.
5.  **Composite:** Composite the shaded samples along each ray into a final image color. This can be done in main memory using back-to-front compositing equations, or by sending the shaded samples to the frame buffer blending hardware [S1].

**Formulas:**
*   **Phong Shading:** The intensity $I$ at a sample point is calculated as:
    $$ I = I_a k_a + \sum_{j} I_{l_j} \left( k_d (\mathbf{n} \cdot \mathbf{l}_j) + k_s (\mathbf{r}_j \cdot \mathbf{v})^{\alpha_{spec}} \right) $$
    Where:
    *   $I_a$ is ambient light intensity, $k_a$ is ambient reflectivity.
    *   $I_{l_j}$ is the intensity of the $j$-th light, $\mathbf{l}_j$ is the light direction vector.
    *   $k_d$ is diffuse reflectivity.
    *   $k_s$ is specular reflectivity, $\alpha_{spec}$ is the specular exponent.
    *   $\mathbf{n}$ is the surface normal (derived from gradient).
    *   $\mathbf{r}_j$ is the reflection vector of the light.
    *   $\mathbf{v}$ is the view direction vector.
    *   *Note:* The exact Phong formula is not provided in the source text, but the source explicitly states "full Phong lighting equation" is used [S1]. The implementer must supply the standard Phong equation [G].
*   **Compositing (Back-to-Front):**
    $$ C_{out} = C_{in} (1 - \alpha_{sample}) + C_{sample} \alpha_{sample} $$
    $$ \alpha_{out} = \alpha_{in} (1 - \alpha_{sample}) + \alpha_{sample} $$
    Where $C_{sample}$ is the shaded color and $\alpha_{sample}$ is the opacity derived from density [S1].

**Parameters:**
*   **Texture Memory Size:** Typical high-end graphics systems have 64 MB of texture memory, which can hold volumes up to $256^3$ with 32-bits per voxel [S1].
*   **Sampling Rate:** Determined by the image size and number of samples along each ray. The source does not specify a fixed number, but notes that 64 sampling planes were used for $512 \times 512$ windows in performance tests [S1].

**Edge Cases and Pitfalls:**
*   **Interpolation Artifacts:** If RGB values are interpolated by texture hardware directly (as in traditional architectures), non-linear mappings from density to RGB produce incorrect interpolated samples [S1]. This method avoids this by interpolating raw density and gradients, then applying non-linear shading in software [S1].
*   **Memory Limits:** The volume data must fit in the texture memory of the graphics system. If the volume is larger than physical texture memory, hardware paging may be used, but performance may degrade [S1].
*   **Gradient Accuracy:** The quality of shading depends on the accuracy of the pre-computed gradients stored in the texture. Poor gradient estimation leads to incorrect normals and shading artifacts.

## Operating characteristics
*   **Time Complexity:** The rendering time is dominated by the texture resampling (hardware accelerated) and the CPU shading/compositing. The source notes that on a four Raster Manager SGI RealityEngine Onyx with one 150MHz CPU, a $512 \times 512 \times 64$ volume rendered into a $512 \times 512$ window in 0.1 seconds [S1].
*   **Memory Complexity:** Requires texture memory proportional to the volume size ($N^3$ voxels). Typical limit is 64 MB for $256^3$ volumes [S1].
*   **Strengths:**
    *   Supports arbitrary and dynamically changing directional lights and viewing parameters without re-loading textures [S1].
    *   Avoids incorrect interpolation artifacts from non-linear transfer functions by shading in software [S1].
    *   Avoids the infeasibly large lookup tables required by post-texturing LUTs [S1].
*   **Limitations:**
    *   Requires transferring sample data from texture memory to main memory, which can be a bottleneck if the bandwidth is low.
    *   CPU shading may be slower than hardware shading for very large numbers of samples, though the source claims high performance on specific hardware [S1].
    *   Limited by the physical texture memory size of the graphics hardware [S1].
*   **When to use:** When high-quality view-dependent shading is required, and the hardware supports 3D texture mapping but lacks sufficient post-texturing LUT capabilities or when dynamic lighting changes make pre-shading impractical. Suitable for SGI Reality Engine and Onyx workstations [S1].

## Background
This method was presented by Frank Dachille, Kevin Kreeger, Baoquan Chen, Ingmar Bitter, and Arie Kaufman from the Center for Visual Computing (CVC) and Department of Computer Science at the State University of New York at Stony Brook [S1]. It builds on the idea of using 3D texture mapping hardware for volume rendering, first mentioned by Akeley on the SGI Reality Engine [S1]. The method addresses limitations of traditional texture-mapped volume rendering architectures, such as the need to re-shade and reload textures for every view change, and the inaccuracies of hardware-interpolating RGB values with non-linear transfer functions [S1]. It is an alternative to Van Gelder and Kim's "Ideal architecture" which uses a post-texturing LUT indexed by gradient, which requires an infeasibly large LUT size [S1]. The method was used on SGI Reality Engine and Onyx workstations with Raster Manager [S1].

## Sources
[S1] High-Quality Volume Rendering Using Texture Mapping Hardware, https://cfcs.pku.edu.cn/baoquan/docs/20180622110034067331.pdf, fetched 2026-10-27
