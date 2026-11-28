# Custom Hardware Rendering
## Overview
Custom Hardware Rendering refers to direct volume rendering techniques implemented via dedicated, non-standard graphics hardware [S2, unofficial]. It emerged to address the computational bottlenecks of software-based rendering when neither CPUs nor GPUs were fast enough to achieve real-time frame rates [S1]. Unlike general-purpose graphics accelerators, these systems utilize highly specialized, fixed-function pipelines optimized for the regular computation patterns of volume data [S1].

## How it works
**Inputs and Outputs:**
* **Input:** Regular 3D volume data (e.g., $256^3$ grids) [S1].
* **Output:** Real-time rendered images (typically 30 fps or higher) [S1].

**Data Structures:**
Specific memory layouts vary by implementation, but generally involve on-chip buffers to store intermediate voxel or sample data to minimize external memory bandwidth [S1].

**Algorithm Steps:**
The implementation varies by hardware generation [S1]. The following describes the hybrid-order approach used in the Cube system, a predecessor to VolumePro [S1]:
1. **Shear Computation:** The custom hardware computes only the shear-step on-board, transforming the volume data to align with the viewing direction based on a Shear-Warp algorithm [S1].
2. **Offloading:** It was planned to let the standard graphics board perform the final warp and render the image [S1].
3. **Warp and Render:** The graphics board handles the warp transformation and rasterization [S1].
Later generations, such as VolumePro, evolved to use separate sample and voxel processing pipelines [S1]. Voxel processors traverse data slice-by-slice in memory order and store results in on-chip buffers [S1]. Sample processors then traverse these buffers for illumination, filtering, and compositing [S1].

**Parameters:**
* **Pipelines:** Systems like Cube utilize eight identical rendering pipelines [S1].
* **Performance:** These pipelines can render a $256^3$ volume at 30 fps [S1].

**Edge Cases and Pitfalls:**
* **Blurring:** Images may blur during zoom-in operations because the re-sampling of the warp matrix is not adaptive [S1]. Enhanced versions exist but increase computational cost [S1].
* **Perspective:** Early custom hardware (like Cube and early VolumePro) did not support perspective rendering [S1].
* **Memory:** Limited on-chip memory restricts the size of the volume that can be processed without external bandwidth bottlenecks [S1].

## Subtopics

### Volume Rendering Unit (VRU)
A dedicated hardware module to render final pixel colors, reducing off-chip memory access by compressing colors and densities of all sample points in a ray into a single pixel color [S3]. It is used in architectures like ICARUS [S3].
**Steps:** The VRU uses a CORDIC module to calculate the exponential of $-\sigma_i \cdot \delta_i$ [S3]. The registered $T_i$ is used in two calculations: 1) multiplication with $\exp(-\sigma_i \cdot \delta_i)$ to get $T_{i+1}$, and 2) subtraction of $T_{i+1}$ from $T_i$ to generate $T_i \cdot (1 - \exp(-\sigma_i \cdot \delta_i))$ [S3]. After these operations, $T_i$ is replaced by $T_{i+1}$ for the next accumulation step [S3]. Once the coefficient is computed, it is multiplied with $c_i$ to get the weighted color of one sample point [S3]. Accumulations proceed until the color of one pixel is obtained [S3].
**Formulas:**
The classic volume rendering equation is:
$$C(r) = \sum_{i=1}^N T_i \cdot (1 - \exp(-\sigma_i \cdot \delta_i)) \cdot c_i, \quad T_i = \exp\left(\sum_{j=1}^{i-1} -\sigma_j \cdot \delta_j\right), \quad \delta_i = x_{i+1} - x_i$$
where $\sigma_i$ is volume density, $c_i$ is sample color after sigmoid activation, and $\delta_i$ is the distance between adjacent samples [S3]. It can be rewritten as:
$$C(r) = \sum_{i=1}^N (T_i - T_{i+1}) \cdot c_i, \quad T_{i+1} = T_i \cdot \exp(-\sigma_i \cdot \delta_i)$$
**Comparison:** VRU vs CPU: VRU reduces data exchange significantly. For original NeRF, CPU requires outputting colors $c=(r,g,b)$ and densities $\sigma$ of 128 sample points per ray. VRU outputs only color $c=(r,g,b)$ of one pixel, which is $3/(128 \times 4) \approx 0.586\%$ of the data [S3].

## Operating characteristics
* **Performance:** Achieves real-time frame rates suitable for interactive rendering due to highly regular computation [S1].
* **Scalability:** Scalability was enhanced in later generations (e.g., VolumePro), but perspective rendering remained a limitation in early designs [S1].
* **Limitations:**
  * **Cost and Flexibility:** Changing or extending custom hardware is tedious and costly [S1].
  * **Memory Constraints:** Like GPUs, these systems suffer from limited memory [S1].
  * **Bandwidth:** Out-of-core solutions are generally not viable due to the high bandwidth requirements [S1].

## Background
The need for custom graphics hardware arose from the demand for real-time volume rendering systems when standard CPUs and GPUs were insufficient [S1]. Notable systems include Cube [KK99], Vizard [KS97], and VolumePro [PHK* 99] [S1]. Cube was based on a hybrid-order algorithm using Shear-Warp and served as the predecessor to the well-known VolumePro board [S1]. While Cube was never commercially realized, VolumePro evolved to include separate sample and voxel processing pipelines, where voxel processors traverse data slice-by-slice and store them in on-chip buffers, which are then traversed by sample processors for illumination, filtering, and compositing [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S3] ICARUS: A Specialized Architecture for Neural Radiance Fields Rendering, https://doi.org/10.48550/arxiv.2203.01414, fetched 2026-10-26
