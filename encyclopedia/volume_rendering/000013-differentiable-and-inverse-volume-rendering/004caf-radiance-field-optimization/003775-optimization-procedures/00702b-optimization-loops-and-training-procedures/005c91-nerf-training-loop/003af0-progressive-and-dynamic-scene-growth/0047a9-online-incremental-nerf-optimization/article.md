# Online Incremental NeRF Optimization

Also called: Incremental NeRF Training, Online NeRF

## Overview
Online Incremental NeRF Optimization reconstructs 3D scenes from a continuous stream of incoming camera views rather than a fixed, pre-collected dataset [S2, unofficial]. Unlike offline methods such as DVGO [S1, unofficial] or siblings like [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md) and [Progressive View Synthesis Training](8666-progressive-view-synthesis-training.md), which optimize a scene representation based on a static set of images, this approach updates the representation dynamically as new observations are collected [S1, unofficial]. It enables active exploration by planning next-best views and integrating data incrementally, allowing the model to build 3D representations with limited viewpoints while adapting to real-world navigation scenarios [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A current scene representation (e.g., a NeRF or grid-based model), a training buffer of stored images and poses, and a new incoming image/pose from a sensor or agent [S1, unofficial][S2, unofficial].
*   **Output:** An updated scene representation and a plan for the next viewpoint [S1, unofficial].

**Data Structures:**
*   **Scene Representation:** Typically a neural radiance field (MLP) or a grid-based representation (e.g., HashGrid, Voxel Grid) [S2, unofficial]. Exact field layouts, sizes, and bit packing are not given in the sources.
*   **Training Buffer:** A set of stored images and their corresponding camera poses, which may be bounded to handle large environments [S2, unofficial]. Exact field layouts, sizes, and bit packing are not given in the sources.

**Steps:**
1.  **Perception:** The system captures a new observation (image and pose) based on the current scene representation [S1, unofficial].
2.  **Incremental Update:** The new observation is added to the training buffer [S1, unofficial].
3.  **Optimization:** The scene representation is optimized using the updated training buffer [S1, unofficial].
4.  **Planning:** Based on the current representation, the system computes a metric (e.g., entropy or uncertainty) to determine the next best view for future exploration [S1, unofficial].
5.  **Iteration:** Repeat steps 1–4 as new data arrives [S1, unofficial].

**Formulas and Parameters:**
*   **Entropy-based Exploration:** The method may use entropy to guide viewpoint selection, allowing the agent to explore diverse viewpoints [S1, unofficial].
*   **Thresholds:** Specific thresholds for triggering growth or pruning are not explicitly defined in the provided sources for the general online method, but follow the parent family's principles [S1, unofficial][S2, unofficial]. Typical parameter values are not given in the sources.

**Pitfalls:**
*   **Catastrophic Forgetting:** In vanilla NeRF, global MLP updates can cause previously reconstructed regions to degrade when new, unsupervised areas are optimized [S2, unofficial].
*   **Memory Usage:** Large environments result in a high number of captured images, requiring strategies to discard older frames or manage memory [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plan_next_best_view(R, D):
    # Compute metric (e.g., entropy) over scene representation R
    # using training data D to select next viewpoint
    return next_view

def capture_observation(next_view):
    # Acquire image I and pose P from sensor at next_view
    return I, P

def optimize(R, D):
    # Perform gradient descent on R using all samples in D
    return R_updated

terminated = False
R = initialize_scene_representation()
D = []
P_curr = None

while not terminated:
    next_view = plan_next_best_view(R, D)
    I_new, P_new = capture_observation(next_view)
    D.append((I_new, P_new))
    R = optimize(R, D)
    P_curr = P_new
```

## Subtopics

### Incremental Reconstruction with Frozen Decoder
A strategy for online incremental reconstruction to address catastrophic forgetting, where the model integrates new viewpoints while preserving previously learned information [S2, unofficial]. It employs pre-trained, scene-agnostic decoders that remain frozen during the feature grids optimization [S2, unofficial]. This design enables new grid cells to represent unseen areas while preserving the integrity of already reconstructed regions [S2, unofficial]. Vanilla NeRF suffers from catastrophic forgetting because every optimization step updates shared MLP weights globally, causing previously reconstructed regions not supervised anymore to degrade [S2, unofficial]. Grid-based representations partially mitigate forgetting due to spatially localized feature storage, but forgetting still occurs due to the decoder and hashgrid dictionary [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the optimization frequency and the size of the training buffer [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Scales with the number of stored views and the complexity of the scene representation; large environments may require discarding older frames to bound memory usage [S2, unofficial].
*   **Strengths:** Supports active exploration by planning next-best views; adapts to dynamic or large-scale scenes by integrating data incrementally [S1, unofficial][S2, unofficial].
*   **Limitations:** Susceptible to catastrophic forgetting in vanilla architectures; requires careful management of training data and optimization steps [S2, unofficial].
*   **When to use:** When data arrives continuously (e.g., robot navigation) and incremental updates are required [S1, unofficial][S2, unofficial].
*   **When not to use:** When a fixed, high-quality reconstruction from a static dataset is sufficient and computational resources allow for offline optimization [S1, unofficial].

## Background
The concept of online incremental NeRF optimization arises from the need to adapt NeRFs to real-world scenarios involving imperfect data, large-scale scenes, and incremental reconstruction [S2, unofficial]. Traditional NeRF methods optimize offline on fixed datasets, which is insufficient for active mapping and planning tasks [S1, unofficial]. The method builds on the principles of progressive and dynamic scene growth [8869-progressive-and-dynamic-scene-growth.md].

## Sources
[S1] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
