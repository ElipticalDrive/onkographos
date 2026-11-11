# Multi-texture Mapping Splatting

Also called: Accelerating splatting algorithm based on multi-texture mapping, Texture-mapped Splatting, Texture-mapped Splatting (Crawfis and Max, Yagel et al.)

## Overview
Multi-texture Mapping Splatting is a hardware-accelerated volume rendering technique that reduces the computational cost of depth sorting by segmenting the volume data into parallel texture slices along the primary viewing direction [S1]. Unlike standard object-order splatting which sorts all voxels globally, this method projects voxels onto local texture planes and blends them using graphics hardware, trading increased memory usage for reduced sorting complexity [S1]. It builds on [Mueller's Splatting Approach](5752-mueller-s-splatting-approach.md) by replacing 3D kernel evaluation with pre-computed 2D footprint lookups and leveraging texture mapping hardware for rasterization [S1].

## How it works
**Inputs:**
* Regular 3D volume dataset with density function $\rho(x,y,z)$ and reconstruction kernel $h_v$ [S1].
* Number of texture slices $n$ [S1].
* Viewing parameters (parallel projection assumed in source [S1]).

**Outputs:**
* Final 2D image composed of blended texture slices [S1].

**Data Structures:**
1. **Footprint Lookup Table:** A 2D table storing values of the footprint function $footprint(x,y)$ [S1].
2. **Texture Slices:** A set of $n$ textures [S1]. Each texture point is in RGBA mode. The original alpha ($\alpha$) value of every point is zero [S1]. All slices are parallel to the view plane and have equivalent dimensions in the x and y directions [S1].

**Steps:**

1. **Preprocessing:**
    * Compute the footprint function by integrating the 3D volume reconstruction kernel $h_v$ over the z-axis [S1]:
        $$footprint(x, y) = \int_{-\infty}^{\infty} h_v(x, y, z) \, dz$$
    * Store the result in a lookup table to avoid excessive arithmetic during rendering [S1].
    * Map voxel values to colors and opacities using a probability statistic method [S1].
    * Transform voxels from object coordinates to the viewing coordinate system using a transformation matrix $M$ [S1]:
        $$\begin{bmatrix} x' \\ y' \\ z' \\ 1 \end{bmatrix} = M \times \begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix}$$

2. **Dividing Volume Data:**
    * Segment the volume data into $n$ regions (texture slices) along the primary viewing direction (z-axis) [S1].
    * Let $n_k$ be the number of samples in slice $k$, such that $\sum n_k = n_{total}$ [S1].

3. **Projecting Voxels:**
    * Traverse voxels in front-to-back order within each segmented region based on depth $z$ [S1].
    * Voxels with the same $z$ belong to the same layer [S1].
    * Project voxels directly onto the texture slice plane (not the screen plane) to avoid popping and splatting overlay artifacts [S1].
    * Calculate the contribution of a voxel $i$ to a point $(x, y)$ in the texture slice using the pre-computed footprint [S1]:
        $$Contribution_i(x, y) = \rho(i) \int_{-\infty}^{\infty} h_v(i_x - x, i_y - y, z) \, dz$$
        This reduces the 3D kernel evaluation to a 2D footprint lookup [S1].

4. **Blending Texture Slices:**
    * Map the texture slices onto polygon surfaces in the scene using hardware texture mapping [S1].
    * Blend the textures in front-to-back order consistent with the voxel projection order [S1].
    * Use the over operator to combine colors and opacities from different layers [S1]:
        $$C_{out} = C_{in} \alpha_{in} + C_{now} \alpha_{now} (1 - \alpha_{in})$$
        $$\alpha_{out} = \alpha_{in} + \alpha_{now} (1 - \alpha_{in})$$
        Where $C$ is color and $\alpha$ is opacity [S1].

**Parameters:**
* $n$: The number of texture slices. Determines the granularity of division and the number of blending passes [S1].

**Edge Cases and Pitfalls:**
* **Popping/Overlay Artifacts:** Occur if voxels are projected onto the screen plane instead of the texture slice plane. The method explicitly projects onto the texture slice plane to maintain correct depth ordering [S1].
* **Precision:** The source notes that lookup tables avoid excessive arithmetic complexity [S1].

## Operating characteristics
* **Time Complexity:** The depth sorting complexity for the divided dataset is $O(A) = \sum n_k \log n_k$, where $n_k$ is the number of samples in slice $k$ [S1]. This is strictly less than the complexity of sorting the whole dataset $O(B) = n \log n$, because $\sum n_k \log n_k < \sum n_k \log n = n \log n$ [S1].
* **Memory Complexity:** Requires storage for $n$ texture slices, each with RGBA components. The original alpha is zero, and dimensions are equivalent across slices [S1].
* **Strengths:** Accelerates rendering by leveraging hardware-assisted texture mapping and reducing sorting complexity [S1]. Scaled volume datasets can be rendered effectively and fast without remarkable degradation in image quality [S1].
* **Limitations:** Requires pre-computation of footprint tables and segmentation of data. The method described assumes parallel projection and regular data sets [S1].
* **When to use:** When hardware-assisted rendering is available and sorting complexity is a bottleneck [S1].

## Background
The method was proposed in 2006 by researchers combining splatting with 2D texture mapping to accelerate footprint-based volume rendering [S1]. It builds on earlier hardware-assisted methods, such as those by Crawfis and Max, and Yagel et al., who used texture mapping hardware to render splats [S2, unofficial]. The approach addresses the computational cost of 3D convolution by reducing it to 2D footprint lookups and hardware blending [S1].

## Sources
[S1] An accelerating splatting algorithm based on multi-texture mapping for volume rendering, https://doi.org/10.1145/1174429.1174464, fetched 2026-11-05
[S2] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
