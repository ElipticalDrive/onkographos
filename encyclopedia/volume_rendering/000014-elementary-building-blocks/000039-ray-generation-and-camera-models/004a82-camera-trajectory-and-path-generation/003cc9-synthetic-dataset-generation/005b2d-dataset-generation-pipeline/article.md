# Dataset Generation Pipeline

Also called: Synthetic & Real Data Preparation

## Overview
The Dataset Generation Pipeline constructs training and test datasets for HDR deblurring and Novel View Synthesis (NVS) by synthesizing or capturing single-exposure Low Dynamic Range (LDR) blurry images paired with corresponding event data. It solves the problem of training neural radiance fields in extreme lighting conditions (highlights and darkness) where standard multi-exposure inputs are unavailable or insufficient. It sets itself apart from sibling pipelines like [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md) by explicitly generating event streams alongside blurry images to model motion blur physics, and from [Replica Dataset Preparation](12080-replica-dataset-preparation.md) by focusing on HDR dynamic range challenges rather than geometric reconstruction of static indoor scans.

## How it works
The pipeline operates in two distinct modes: Synthetic and Real. Both modes generate training views with specific exposure times and camera poses to simulate camera shake and motion blur.

### Inputs and Outputs
*   **Synthetic Inputs:** 8 HDR Blender scenes from the HDR-NeRF dataset ("bathroom", "catroom", "dogroom", "diningroom", "sofa", "sponza", "toyroom", "warmroom") [S1, unofficial].
*   **Real Inputs:** 5 real-world scenes ("bear", "computer", "computerhard", "desktop", "table") captured with a DAVIS 346 Color camera [S1, unofficial].
*   **Synthetic Outputs:** 18 training views per scene, each containing 1 LDR blurry image and corresponding events.
*   **Real Outputs:** 16 training views per scene, each containing 1 LDR blurry image and corresponding events.

### Data Structures
*   **Synthetic Resolution:** $400 \times 400$ pixels [S1, unofficial].
*   **Real Resolution:** $346 \times 260$ pixels [S1, unofficial].
*   **Image Format:** LDR RGB images [S1, unofficial].
*   **Event Format:** Spatial-temporal aligned event streams corresponding to the exposure period [S1, unofficial].

### Parameters
*   **Synthetic Exposure Times:** $\Delta t_0 = 0.001\text{s}$, $\Delta t_1 = 0.004\text{s}$, $\Delta t_2 = 0.016\text{s}$, $\Delta t_3 = 0.064\text{s}$, $\Delta t_4 = 0.256\text{s}$ [S1, unofficial].
*   **Real Exposure Times:** $\Delta t_0 = 0.05\text{s}$ to $\Delta t_4 = 0.25\text{s}$ (specifically $0.05, 0.10, 0.15, 0.20, 0.25\text{s}$) [S1, unofficial].
*   **Target Training Exposure:** $\Delta t_2$ is used for the primary blurry training images in both modes [S1, unofficial].

### Steps

#### Synthetic Data Generation
1.  **Scene Selection:** Load one of the 8 HDR Blender scenes [S1, unofficial].
2.  **View Generation:** For each of the 18 training views:
    a.  Define a camera trajectory with shaking. Generate 17 sharp HDR raw images $\{I_{\text{HDR}}^i\}_{i=0}^{16}$ at times $t_i$ with poses $p(t_i)$, where $t_{\text{start}} \le t_i \le t_{\text{end}}$ [S1, unofficial].
    b.  The pose $p(t_i)$ changes over time to simulate camera shake [S1, unofficial].
    c.  Render 1 LDR blurry image at exposure $\Delta t_2$ using the HDR data and the corresponding event stream generated from the motion between $t_{\text{start}}$ and $t_{\text{end}}$ [S1, unofficial].
3.  **Test Data Generation (Optional):** For NVS tests, transform the sharp HDR image $I_{\text{HDR}}$ into two GT sharp LDR images using Eq. (18) (not provided in source) at novel exposure times $\Delta t_1$ and $\Delta t_3$ [S1, unofficial].

#### Real Data Generation
1.  **Scene Setup:** Position the DAVIS 346 Color camera in one of the 5 real-world scenes [S1, unofficial].
2.  **View Capture:** For each of the 16 training views:
    a.  Capture 1 LDR blurry image at exposure $\Delta t_2$ using a handheld camera [S1, unofficial].
    b.  Simultaneously capture the corresponding event stream within the exposure time $\Delta t_2$ [S1, unofficial].
3.  **Additional Training Data (Optional):** For RGB-based HDR NVS reference training, capture 3 sharp LDR images at exposures $\Delta t_0, \Delta t_2, \Delta t_4$ using a tripod-fixed camera to avoid deblurring effects [S1, unofficial].

### Edge Cases and Pitfalls
*   **Dynamic Range:** Scenes must contain both highlight and darkness areas to ensure rendered/captured images have both overexposure and underexposure [S1, unofficial].
*   **Camera Shake:** In synthetic data, the camera pose must vary continuously over time to generate realistic motion blur; static poses will not produce the required blur [S1, unofficial].
*   **Sensor Alignment:** In real data, the DAVIS camera must provide spatial-temporal aligned images and events; misalignment will corrupt the event-image pairing [S1, unofficial].

## Subtopics

### [Paired Simulation Protocol](20509-paired-simulation-protocol.md)
A standardized evaluation framework that freezes input data, camera poses, and metrics before training to ensure performance differences arise solely from the algorithm. It enforces blind evaluation where clean ground-truth is inaccessible during training.

### [Difix3D Pipeline](20628-difix3d-pipeline.md)
A training data generation method that synthesizes novel views and uses a pretrained diffusion model (SD-Turbo) to remove degradation (blur/artifacts) from rendered images before retraining. It targets HDR deblurring and novel view synthesis with extreme lighting.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The pipeline is designed to train models on scenes with extreme lighting, resulting in images with both overexposure and underexposure areas [S1, unofficial].
*   **Strengths:** Provides paired event-image data for single-exposure HDR deblurring; covers a wide dynamic range.
*   **Limitations:** Synthetic data relies on Blender scenes which may not capture real-world sensor noise characteristics; real data requires specialized hardware (DAVIS camera) [S1, unofficial].
*   **When to use:** When training HDR NeRF or deblurring models that require event data to resolve motion blur from single-exposure inputs.
*   **When not to use:** When multi-exposure sharp images are available as input, or when event data is not required.

## Background
The pipeline originates from the "Seeing through Light and Darkness" paper, which introduces a sensor-physics grounded approach to deblurring HDR NeRF from single-exposure images and events [S1, unofficial]. It builds on the HDR-NeRF dataset [19] for synthetic scenes and utilizes the DAVIS 346 Color camera [45] for real-world data acquisition. The method addresses the limitations of previous NVS methods by incorporating event streams to handle motion blur in extreme lighting conditions [S1, unofficial].

## Sources
[S1] Seeing through Light and Darkness: Sensor-Physics Grounded Deblurring HDR NeRF from Single-Exposure Images and Events, https://arxiv.org/abs/2601.15475v4, fetched 2026-10-26 (unofficial)
