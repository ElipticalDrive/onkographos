# Neural Network Model for Participating Media Scattering

Also called: Neural Network Model

## Overview
A method for interactive simulation of scattering effects in participating media that uses a neural network to evaluate multiple and double scattering, combined with a single scattering approximation [S1]. It sets itself apart from table-based approaches like [Neural Network Multiple Scattering Representation](7796-neural-network-multiple-scattering-representation.md) by storing the entire parameter space in kilobytes rather than megabytes, and from Virtual Ray Lights by avoiding the artifact-prone need for massive light counts [S1].

## How it works
**Inputs:** Camera ray, scene geometry, participating media properties (homogeneous, Henyey-Greenstein phase function, refractive boundaries), light source positions.
**Outputs:** Radiance value for the pixel.

**Data Structures:**
*   **Neural Network:** A fully connected feed-forward network.
    *   Architecture: Single hidden layer (or multiple layers with uniform size).
    *   Nodes per layer: 50 [S1].
    *   Activation function: $\tanh$ [S1].
    *   Memory footprint: 23.6 KB for the entire parameter space [S1].
    *   [G] Input vector dimensions and output dimensions are not explicitly defined in the source text, but the network maps camera segment properties and light segment properties to scattering contribution.

**Steps:**

1.  **Single Scattering Evaluation:**
    *   Sample 100 regular points along the camera ray [S1].
    *   For each sample point, connect to all light sources [S1].
    *   Compute single scattering contribution without considering refraction at the medium boundary [S1].

2.  **Multiple and Double Scattering Evaluation (Neural Network):**
    *   The neural network is trained to predict **Segment-to-Point** contributions, not Segment-to-Segment contributions [S1].
    *   **Training Constraint:** Camera segment length is constrained to 5 mean-free paths (mfp) during training [S1].
    *   **Inference:**
        *   Define the camera segment based on the current ray and the 5 mfp constraint.
        *   Define the light segment (or point) corresponding to the light source interaction.
        *   Pass the segment parameters into the neural network to obtain the contribution [S1].
        *   Accumulate contributions from the Segment-to-Point predictions to approximate the total multiple scattering effect for the camera segment [S1].

**Pitfalls:**
*   **Segment-to-Segment Training:** Training the network directly on Segment-to-Segment contributions overestimates the scattering effects. This is because accumulation amplifies differences between lobes, making the network less accurate and producing images that are lighter than the reference [S1].
*   **Index-Matching Materials:** For materials with no refraction at boundaries (index-matching), the algorithm assumes a single interface and semi-infinite material. This assumption tends to overestimate extinction [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_scattering(camera_ray, media_properties, light_sources):
    """
    Evaluates scattering for a single camera ray.
    
    Args:
        camera_ray: Object containing ray origin, direction, and length.
        media_properties: Object containing scattering coefficient, absorption coefficient,
                          phase function parameters (g), and refractive index.
        light_sources: List of light source objects.
    
    Returns:
        radiance: Total radiance value.
    """
    
    # 1. Single Scattering Approximation
    single_scattering = 0.0
    num_samples = 100
    ray_length = camera_ray.length
    
    for i in range(num_samples):
        # Regular sampling along the camera ray
        t = (i + 0.5) / num_samples * ray_length
        sample_point = camera_ray.origin + camera_ray.direction * t
        
        for light in light_sources:
            # Connect sample point to light source
            # Note: Refraction at medium boundary is ignored
            light_direction = light.position - sample_point
            distance = light_direction.length()
            light_direction = light_direction.normalize()
            
            # Compute single scattering contribution
            # [G] Standard single scattering formula involving transmittance and phase function
            # would be implemented here based on media_properties.
            # The source specifies the sampling strategy but not the exact radiative transfer
            # equation for the single scattering term, assuming standard implementation.
            ss_contribution = compute_single_scattering(sample_point, light_direction, distance, media_properties, light)
            single_scattering += ss_contribution
            
    # 2. Multiple and Double Scattering via Neural Network
    # The network predicts Segment-to-Point contributions.
    # Camera segment length is constrained to 5 mean-free paths (mfp).
    
    mfp = 1.0 / media_properties.extinction_coefficient
    camera_segment_length = 5.0 * mfp
    
    # [G] The source does not specify the exact input vector format for the NN.
    # We assume the NN takes parameters describing the camera segment and the light segment/point.
    
    multiple_scattering = 0.0
    
    # Sample points along the camera segment for accumulation
    # The source implies accumulation of Segment-to-Point contributions.
    # We use a reasonable number of points for accumulation, e.g., 10.
    num_accumulation_points = 10
    
    for i in range(num_accumulation_points):
        t = (i + 0.5) / num_accumulation_points * camera_segment_length
        segment_point = camera_ray.origin + camera_ray.direction * t
        
        for light in light_sources:
            # Define the light segment/point
            # [G] Assuming point light for simplicity as per "Segment-to-Point"
            light_segment = PointSegment(segment_point, light.position)
            
            # Define the camera segment (fixed length 5 mfp)
            # [G] The source says "constrain the length of the camera segment to 5 mfp"
            # It is unclear if this is a fixed window or relative to the sample.
            # We assume the input to the NN describes the relative geometry.
            camera_segment = LineSegment(segment_point, segment_point + camera_ray.direction * camera_segment_length)
            
            # Query Neural Network
            nn_input = encode_segments(camera_segment, light_segment, media_properties)
            contribution = neural_network_forward(nn_input)
            
            multiple_scattering += contribution
            
    # Scale accumulation if necessary (e.g., divide by num_accumulation_points)
    # [G] The source does not specify the exact accumulation weighting.
    multiple_scattering /= num_accumulation_points
    
    return single_scattering + multiple_scattering

def encode_segments(camera_seg, light_seg, media):
    """
    Encodes segment geometry and media properties into a vector for the NN.
    [G] Specific feature vector dimensions and normalization are not given in the source.
    """
    # Placeholder for feature encoding logic
    return [0.0] * 10 # [G] Placeholder

def neural_network_forward(input_vector):
    """
    Forward pass through the neural network.
    Architecture: Input -> Dense(50, tanh) -> Output
    """
    # [G] Weights and biases are not provided.
    # This function represents the trained model.
    hidden = tanh(dot(input_vector, W1) + b1)
    output = dot(hidden, W2) + b2
    return output

def tanh(x):
    return math.tanh(x)

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))
```

## Operating characteristics
*   **Performance:** 50 ms per frame for multiple scattering only; 150 ms per frame for the full solution (single, double, and multiple scattering) [S1].
*   **Memory:** 23.6 KB for storing parameters for the entire space of participating media [S1]. This is significantly lower than the 100 MB required for a single material in Wang and Holzschuch [4].
*   **Scaling:** Rendering cost varies with mean-free path (mfp). In the Candle scene, cost increases from 26 ms to 65 ms as mfp decreases, while the number of virtual ray lights increases from 40,000 to 2 million [S1].
*   **Grid Hierarchy:** Using a grid hierarchy has minimal impact on quality (around 5%) but prevents significant performance degradation as mfp decreases [S1].
*   **Strengths:** Instantaneous material parameter changes (single network for all homogeneous media); interactive frame rates; better quality than Dipole approximation in thin areas [S1].
*   **Limitations:** Overestimates extinction for index-matching materials due to the single interface/semi-infinite assumption; requires careful training on Segment-to-Point contributions to avoid overestimation [S1].

## Background
The method was proposed by the authors of "Interactive Simulation of Scattering Effects in Participating Media Using a Neural Network Model" [S1]. It builds upon the single scattering approximation by Jensen et al. [3] and compares against Wang and Holzschuch [4] and Virtual Ray Lights (VRL) [S1].

## Sources
[S1] Interactive Simulation of Scattering Effects in Participating Media Using a Neural Network Model, https://doi.org/10.1109/TVCG.2019.2963015, fetched 2026-10-26
