# Neural Rendering of Variable Scenes

Also called: Variable Scene Methods

## Overview
Neural Rendering of Variable Scenes is a method for dynamic scene representation that explicitly incorporates scene variation into the training phase by encoding predefined scene parameters into a structured vector [S1]. Unlike methods that constrain learning to a single manifold or assume static geometry, these approaches learn a surface radiance field capable of handling all possible permutations of the specified scene variables [S1]. This distinguishes them from NeRF-based Inverse Volume (NIV) methods, which do not explicitly model higher-order interactions between dynamic objects and the surrounding scene [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of predefined scene variables (e.g., emitter positions, object transforms) encoded into a structured vector, and static scene geometry or appearance data.
*   **Outputs:** A neural model (typically a neural radiance field or irradiance field) that predicts radiance or irradiance values conditioned on the variable scene parameters.

**Data Structures:**
*   **Scene Variable Vector:** A structured representation (usually a vector) encoding the range of each variable scene element [S1].
*   **Neural Model:** A neural network (e.g., MLP or hash-based encoding) that takes spatial coordinates and the scene variable vector as input [S1].

**Steps:**
1.  **Encode Scene Variables:** Define the predefined range for each variable scene element and encode them into a structured vector [S1]. This vector can be used directly as input to the neural model or processed through a learned encoding layer [S1].
2.  **Train Radiance Field:** Train the neural model to learn a surface radiance field for all possible permutations of the scene variables [S1]. During training, the variable scene parameters are randomly sampled to ensure the model generalizes across the defined range [S1].

**Parameters:**
*   **Number of Variables:** Typically limited to 1-2 variables modeling dynamic phenomena to maintain performance and avoid visible reconstruction error at fixed representational capacity [S1].
*   **Encoding Method:** Frequency encoding can be applied to parameter changes to add little inference overhead [S1].

**Pitfalls:**
*   **Retraining Requirement:** The model requires retraining whenever the set of scene parameters changes (e.g., objects added, removed, or modified) [S1].
*   **Performance:** Inference times exceed 100 ms, failing to achieve real-time performance [S1].
*   **Scaling:** Inference time scales with the number of dynamic objects, limiting the use of primitives with many scene variables, such as animated meshes [S1].
*   **Capacity Limits:** Scaling to more than two scene variables at the same representational capacity causes visible reconstruction error [S1].

## Subtopics

### Higher-dimensional Irradiance Fields
Extends a pre-computed neural irradiance volume by adding inputs for known runtime-changing scene parameters (e.g., emitter positions) to avoid real-time representation updates and temporal artifacts [S1]. The irradiance field is trained while randomly sampling the variable scene parameter [S1]. This approach targets learning 1-2 variables, using frequency encoding [MST* 21] for parameter changes to minimize inference overhead [S1]. To ensure artifact-free results, the hash table size encoding scene position is doubled to ~10MB [S1]. Implemented use cases include a rotating directional light source simulating a time-of-day cycle and a moving occluding object [S1]. Scaling to more than two variables causes visible reconstruction error, which could be resolved with learned encodings but introduces disadvantages noted in the main method [S1].

## Operating characteristics
*   **Time Complexity:** Inference times exceed 100 ms, failing to achieve real-time performance [S1]. Inference time scales with the number of dynamic objects present in the scene [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** For a fixed model size, these methods often achieve comparable or lower rendering error than NIV as the number of scene variables increases, because they devote capacity to learning higher-order interactions between dynamic objects and the scene [S1]. As model capacity grows, rendering error can drop below the theoretical lower bound of irradiance volumes (the error level of a path-traced static irradiance field), producing shading effects resembling path-traced GI including higher-order interactions [S1].
*   **Strengths:** Can produce shading effects resembling path-traced global illumination, including higher-order interactions between variable parameters and the scene [S1].
*   **Limitations:** Requires retraining for parameter changes; inference times exceed 100 ms; inference time scales with the number of dynamic objects; scaling to more than two variables causes visible reconstruction error [S1].
*   **When to use:** When higher-order interactions between dynamic objects and the scene are critical and real-time performance is not required [S1].
*   **When not to use:** When real-time performance is required, or when the number of dynamic objects/variables is large [S1].

## Background
Originated in the context of neural rendering methods that explicitly take scene variation into account during training, assuming a predefined range for each variable scene element [S1]. The method is discussed in contrast to NeRF-based Inverse Volume (NIV) approaches and other techniques that constrain learning to a single manifold [S1].

## Sources
[S1] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
