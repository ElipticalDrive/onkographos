# Neural Network Model for Participating Media Scattering

Also called: Neural Network Model

## Overview
A method for interactive simulation of scattering effects in participating media that uses a neural network to evaluate multiple and double scattering, combined with a single scattering approximation [S1]. It sets itself apart from table-based approaches like [Neural Network Multiple Scattering Representation](7796-neural-network-multiple-scattering-representation.md) by storing the entire parameter space in kilobytes (23.6 KB) rather than megabytes, and from Virtual Ray Lights by avoiding the artifact-prone need for massive light counts under equal time constraints [S1].

## How it works
**Inputs:** Camera ray, scene geometry, participating media properties (homogeneous, Henyey-Greenstein phase function, refractive boundaries), light source positions [S1].
**Outputs:** Radiance value for the pixel [S1].

**Data Structures:**
*   **Neural Network:** A fully connected feed-forward network [S1].
    *   Architecture: Single hidden layer (or multiple layers with uniform size) [S1].
    *   Nodes per layer: 50 [S1].
    *   Activation function: $\tanh$ [S1].
    *   Memory footprint: 23.6 KB for the entire parameter space [S1].

**Steps:**
1.  **Single Scattering Evaluation:** Sample 100 regular points along the camera ray [S1]. For each sample point, connect to all light sources [S1]. Compute single scattering contribution without considering refraction at the medium boundary [S1].
2.  **Multiple and Double Scattering Evaluation (Neural Network):** The neural network is trained to predict **Segment-to-Point** contributions, not Segment-to-Segment contributions [S1]. **Training Constraint:** Camera segment length is constrained to 5 mean-free paths (mfp) during training [S1]. **Inference:** Define the camera segment based on the current ray and the 5 mfp constraint [S1]. Define the light segment (or point) corresponding to the light source interaction [S1]. Pass the segment parameters into the neural network to obtain the contribution [S1]. Accumulate contributions from the Segment-to-Point predictions to approximate the total multiple scattering effect for the camera segment [S1].

**Pitfalls:**
*   **Segment-to-Segment Training:** Training the network directly on Segment-to-Segment contributions overestimates the scattering effects. This is because accumulation amplifies differences between lobes, making the network less accurate and producing images that are lighter than the reference [S1].
*   **Index-Matching Materials:** For materials with no refraction at boundaries (index-matching), the algorithm assumes a single interface and semi-infinite material. This assumption tends to overestimate extinction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_scattering(camera_ray_origin, camera_ray_direction, media_extinction, light_sources):
    # 1. Single Scattering Evaluation
    single_scattering = 0.0
    num_samples = 100
    ray_length = camera_ray_direction.length()

    for i in range(num_samples):
        ray_t = (i + 0.5) / num_samples * ray_length
        sample_point = camera_ray_origin + camera_ray_direction * ray_t

        for light in light_sources:
            light_direction = light.position - sample_point
            distance = light_direction.length()
            light_direction = light_direction.normalize()

            # [G] Standard single scattering computation ignoring boundary refraction
            # L = light.emission * exp(-media_extinction * distance) * phase_function / distance^2
            ss_contribution = light.emission * math.exp(-media_extinction * distance) / (distance ** 2)
            single_scattering += ss_contribution

    # 2. Multiple and Double Scattering Evaluation (Neural Network)
    # Camera segment length constrained to 5 mean-free paths (mfp) during training
    mfp = 1.0 / media_extinction
    camera_segment_length = 5.0 * mfp

    multiple_scattering = 0.0
    num_accumulation_points = 10 # [G] Accumulation step count not specified in source

    for i in range(num_accumulation_points):
        accum_t = (i + 0.5) / num_accumulation_points * camera_segment_length
        segment_point = camera_ray_origin + camera_ray_direction * accum_t

        for light in light_sources:
            # Segment-to-Point geometry: camera segment of length 5 mfp to a point light
            camera_seg_end = segment_point + camera_ray_direction * camera_segment_length
            light_point = light.position

            # [G] Feature vector encoding not specified in source; standard geometric features assumed
            seg_len = camera_segment_length
            light_dist = (light_point - segment_point).length()
            nn_input = [seg_len, light_dist, media_extinction]

            # Neural Network Forward Pass: Input -> Dense(50, tanh) -> Output
            # [G] Weights and biases are trained parameters, not provided in source
            hidden = [tanh(sum(nn_input[j] * W1[j][k] for j in range(len(nn_input))) + b1[k]) for k in range(50)]
            contribution = sum(hidden[k] * W2[k][0] for k in range(50)) + b2[0]

            multiple_scattering += contribution

    # Accumulate Segment-to-Point contributions
    multiple_scattering /= num_accumulation_points

    return single_scattering + multiple_scattering

def tanh(x):
    return math.tanh(x)

# [G] Network weights and biases initialized to zeros; replaced by trained values during inference
W1 = [[0.0] * 50]
b1 = [0.0] * 50
W2 = [[0.0]]
b2 = [0.0]
```

## Operating characteristics
*   **Performance:** 50 ms per frame for multiple scattering only; 150 ms per frame for the full solution (single, double, and multiple scattering) [S1].
*   **Memory:** 23.6 KB for storing parameters for the entire space of participating media [S1]. This is significantly lower than the 100 MB required for a single material in Wang and Holzschuch [4] [S1].
*   **Scaling:** Rendering cost varies with mean-free path (mfp) [S1]. In the Candle scene, cost increases from 26 ms to 65 ms as mfp decreases, while the number of virtual ray lights increases from 40,000 to 2 million [S1].
*   **Grid Hierarchy:** Using a grid hierarchy has minimal impact on quality (around 5%) but prevents significant performance degradation as mfp decreases [S1].
*   **Strengths:** Instantaneous material parameter changes (single network for all homogeneous media); interactive frame rates; better quality than Dipole approximation in thin areas [S1].
*   **Limitations:** Overestimates extinction for index-matching materials due to the single interface/semi-infinite assumption; requires careful training on Segment-to-Point contributions to avoid overestimation [S1].

## Background
The method was proposed by the authors of "Interactive Simulation of Scattering Effects in Participating Media Using a Neural Network Model" [S1]. It builds upon the single scattering approximation by Jensen et al. [3] and compares against Wang and Holzschuch [4] and Virtual Ray Lights (VRL) [S1].

## Sources
[S1] Interactive Simulation of Scattering Effects in Participating Media Using a Neural Network Model, https://doi.org/10.1109/TVCG.2019.2963015, fetched 2026-10-26
