# Custom Hardware Rendering


## Overview
Custom Hardware Rendering is a direct volume rendering technique implemented via dedicated, application-specific integrated circuits (ASICs) or field-programmable gate arrays (FPGAs) rather than general-purpose processors [S2, unofficial]. It addresses the historical need for real-time volume rendering systems when neither CPUs nor GPUs possessed sufficient computational throughput [S1]. Unlike software-based ray casting or texture-mapped approaches, these systems offload the entire computationally intensive pipeline to fixed-function hardware, achieving interactive frame rates for regular data sets through highly parallel, regular computation [S1].

## How it works
Custom hardware rendering systems are specialized accelerators designed to execute volume rendering algorithms with minimal latency. The implementation varies by specific architecture (e.g., Cube, Vizard, VolumePro), but generally follows a pipeline approach optimized for regular grid data.

**Inputs and Outputs:**
*   **Input:** 3D scalar volume data (typically regular grids), transfer functions, and camera parameters.
*   **Output:** 2D color image frame buffer.

**Data Structures:**
*   **Volume Data:** Stored in on-chip memory or high-bandwidth external memory. The data is accessed in memory order to maximize bandwidth efficiency [S1].
*   **On-chip Buffers:** Used to store intermediate voxel slices or sample results between processing stages (e.g., between voxel processing and sample processing) [S1].

**Algorithm Steps:**
1.  **Pipeline Configuration:** The hardware is configured with a specific number of identical rendering pipelines. For example, the Cube architecture utilized eight identical pipelines [S1].
2.  **Voxel Processing (Traversal):** Voxel processors traverse the volume data slice-by-slice in memory order. This step involves fetching voxel values and performing initial classification or filtering [S1].
3.  **Sample Processing (Compositing):** Sample processors traverse the on-chip buffers to perform illumination, filtering, and front-to-back compositing. This stage accumulates the final color and opacity for each pixel [S1].
4.  **Image Warping/Rendering:** In hybrid-order algorithms like Cube, the hardware may compute only the shear step on-board, while a graphics board handles the warp and final image rendering [S1].

**Formulas:**
The compositing step follows the standard front-to-back volume rendering equation [S1]:
$$
C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
$$
$$
\alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
$$
Where $C_i$ and $\alpha_i$ are the color and opacity of the current sample, and $C'_{i}$ and $\alpha'_{i}$ are the accumulated color and opacity.

**Parameters:**
*   **Pipeline Count:** The number of parallel pipelines determines throughput. Cube used 8 pipelines [S1].
*   **Resolution Support:** Systems were designed for specific resolutions, e.g., Cube rendered a $256^3$ volume [S1].

**Edge Cases and Pitfalls:**
*   **Memory Limitations:** Custom hardware shares the limited memory constraint of GPUs. Out-of-core solutions are generally not viable due to the high memory bandwidth required for real-time rendering [S1].
*   **Rigidity:** Changing or extending the hardware logic is tedious and costly, as it requires physical redesign or reconfiguration of the hardware [S1].
*   **Perspective Limitations:** Early systems like Cube and VolumePro did not support perspective rendering, limiting their applicability to orthographic views [S1].

## Operating characteristics
*   **Performance:** Achieved real-time frame rates (e.g., 30 fps for a $256^3$ volume on Cube) due to highly regular computation and parallel pipelines [S1].
*   **Scalability:** Scalability was enhanced in later generations (VolumePro), but perspective rendering remained unsupported in early custom hardware [S1].
*   **Memory:** Limited by on-chip or dedicated high-bandwidth memory; not suitable for out-of-core rendering [S1].
*   **Flexibility:** Low. Hardware is fixed-function, making updates expensive and difficult [S1].
*   **Use Case:** Best suited for static, regular grid data where maximum throughput is required and perspective viewing is not critical. Not suitable for irregular data or dynamic algorithm changes.

## Background
Custom hardware rendering emerged in the late 1990s when CPUs and GPUs could not meet the real-time requirements for volume rendering [S1]. Key systems included Cube [KK99], Vizard [KS97], and VolumePro [PHK* 99]. Cube, based on a Shear-Warp hybrid algorithm, was the predecessor to VolumePro [S1]. VolumePro evolved to include separate sample and voxel processing pipelines, improving scalability but still lacking perspective support in its earlier generations [S1]. These systems were primarily developed for medical imaging and scientific visualization of regular data sets [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
