# Neural Rendering of Variable Scenes

Also called: Variable Scene Methods

## Overview
Neural rendering of variable scenes is a method that explicitly incorporates scene variation into the training phase by encoding predefined ranges of dynamic scene elements into a structured vector representation [S1]. Unlike methods that constrain learning to a single static manifold, these approaches learn a surface radiance field capable of handling all possible permutations of the specified scene variables [S1]. This distinguishes them from sibling methods like [DyNeRF](7274-dynerf.md), which typically model deformation or motion through explicit geometric transformations or temporal bases, whereas variable scene methods treat dynamic parameters as direct inputs to the neural field to capture higher-order interactions between moving objects and the static environment [S1].

## How it works
The method operates by mapping a set of dynamic scene parameters to radiance values, allowing the neural network to generalize across different configurations of the scene without retraining for every specific instance.

**Inputs and Outputs**
*   **Inputs:** A ray $\mathbf{r}$ (origin $\mathbf{o}$, direction $\mathbf{d}$) and a vector of scene variables $\mathbf{v}$ representing the current state of dynamic elements (e.g., object positions, light angles) [S1].
*   **Outputs:** The estimated color $\hat{C}(\mathbf{r})$ and optionally density $\sigma$ for volume rendering or surface shading [S1].

**Data Structures**
*   **Scene Variable Vector ($\mathbf{v}$):** A structured representation encoding the predefined range of each variable scene element. The dimensionality of $\mathbf{v}$ corresponds to the number of dynamic parameters being modeled [S1].
*   **Neural Radiance Field Model:** A neural network (e.g., MLP or hash-based encoding) that takes the ray coordinates and the encoded scene variables as input [S1].

**Steps**
1.  **Encode Scene Variables:** The dynamic scene parameters are encoded into a vector $\mathbf{v}$. This can be done directly or processed through a learned encoding scheme [S1].
2.  **Construct Input Tensor:** Combine the ray sampling position (or surface point) with the encoded scene variables $\mathbf{v}$ to form the input to the neural network [S1].
3.  **Evaluate Radiance Field:** The neural network predicts the outgoing radiance (and density, if applicable) for the given ray and scene configuration [S1].
4.  **Render:** Compute the final pixel color using the standard volume rendering equation or surface shading model, integrating the predicted radiance along the ray [S1].

**Formulas**
The core prediction is a function $f$ parameterized by weights $\theta$:
$$
(\sigma, \mathbf{c}) = f_\theta(\mathbf{x}, \mathbf{v})
$$
where $\mathbf{x}$ is the spatial coordinate along the ray, and $\mathbf{v}$ is the scene variable vector [S1]. The final color is computed via the volume rendering equation defined in the parent article [44-radiance-field-optimization.md].

**Parameters**
*   **Number of Variables:** Typically limited to 1-2 variables to maintain performance and avoid visible reconstruction error at fixed representational capacity [S1].
*   **Encoding Method:** Frequency encoding is often used for parameter changes to add little inference overhead [S1].

**Edge Cases and Pitfalls**
*   **Retraining Requirement:** The model requires retraining if the set of scene parameters changes (e.g., objects are added, removed, or modified) [S1].
*   **Performance Scaling:** Inference time scales with the number of dynamic objects, limiting the use of primitives with many variables like animated meshes [S1].
*   **Capacity Limits:** Scaling to more than two scene variables at the same representational capacity causes visible reconstruction error [S1].

## Subtopics

### Higher-dimensional Irradiance Fields
Extends a pre-computed neural irradiance volume by adding inputs for known runtime-changing scene parameters (e.g., emitter positions) to avoid real-time representation updates and temporal artifacts [S1]. It trains the irradiance field while randomly sampling the variable scene parameter during the training phase [S1]. This approach is suitable for simulating phenomena like time-of-day cycles with rotating directional lights or moving occluders [S1]. To ensure artifact-free results, the hash table size encoding scene position may need to be doubled (e.g., to ~10MB) [S1]. However, scaling to more than two scene variables at the same capacity causes visible error, which might be resolved with learned encodings but introduces other disadvantages [S1].

## Operating characteristics
*   **Time Complexity:** Inference times exceed 100 ms in reported experiments, failing to achieve real-time performance [S1]. The inference time scales with the number of dynamic objects present in the scene [S1].
*   **Memory Complexity:** Not explicitly given, but hash table sizes may need to be increased (e.g., ~10MB) to handle variable parameters artifact-free [S1].
*   **Strengths:** Can produce shading effects resembling path-traced global illumination, including higher-order interactions between variable parameters and the scene, potentially dropping below the theoretical lower bound of irradiance volumes as capacity grows [S1].
*   **Limitations:** Requires retraining for changes in the set of scene parameters; poor real-time performance; scaling issues with many dynamic objects [S1].
*   **Comparison:** For a fixed model size, Neural Irradiance Volumes (NIV) often achieve comparable or lower rendering error than variable-scene methods as the number of scene variables increases, because variable-scene methods must devote capacity to learning interactions on top of outgoing radiance [S1].

## Background
This method is discussed in the context of comparing different approaches to dynamic scene rendering, specifically contrasting with Neural Irradiance Volumes (NIV) [S1]. The work highlights the trade-offs between explicit variable encoding and implicit generalization via irradiance volumes [S1].

## Sources
[S1] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
