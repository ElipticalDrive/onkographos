# DONeRF Oracle Input Construction

Also called: Depth oracle inputs, Logarithmic input transformation

## Overview
This method defines the input vector construction for the depth oracle network in DONeRF, which predicts sample locations for the subsequent shading network. It solves the problem of providing sufficient spatial context to the oracle so it can accurately classify depth ranges, thereby improving the quality of the final rendered image. It sets itself apart from standard NeRF training by explicitly feeding discretized 3D positions and applying a logarithmic transformation to align with the shading network's sampling strategy, rather than relying on positional encoding or single-depth predictions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $I$ 3D positions along the ray.
*   **Output:** The input vector passed to the first layer of the depth oracle network.

**Data Structures:**
*   **Positions:** A list or array of $I$ 3D coordinates. Each coordinate corresponds to the center of a discretized depth range.
*   **Encoding:** No positional or Fourier encoding is applied to these inputs [S1].

**Steps:**
1.  **Discretize Depth Ranges:** Divide the ray's depth interval into $N$ discrete bins (classification steps).
2.  **Calculate Centers:** For each bin $i$ (where $i \in [0, N-1]$), calculate the 3D position $p_i$ at the center of the bin.
3.  **Logarithmic Transformation:** Apply a logarithmic transformation to the depth values of these centers. This aligns the input distribution with the logarithmic sampling strategy used by the shading network [S1].
    *   Let $d_i$ be the depth of the center of bin $i$.
    *   The transformed depth is $d'_i = \log(d_i)$ (or equivalent logarithmic scaling consistent with the shading network's implementation) [S1].
    *   The 3D position $p_i$ is constructed using $d'_i$ and the ray direction.
4.  **Construct Input Vector:** Concatenate the $I$ transformed 3D positions into a single input vector.
    *   The number of positions $I$ is a hyperparameter. The final configuration uses $I=128$ [S1].
    *   The input vector size is $I \times 3$ (assuming 3D coordinates) [S1].

**Parameters:**
*   **$I$ (Number of input positions):** Tested values range from 1 to 128. The optimal value used in the final model is 128 [S1].
*   **$N$ (Number of samples per ray):** Used for discretization. Tested values: 2, 4, 8, 16 [S1].

**Edge Cases and Pitfalls:**
*   **Positional Encoding:** Experiments showed that using positional or Fourier encoding for depth oracle inputs did not improve results and was therefore omitted [S1].
*   **Filter Size:** While not part of the input construction itself, note that using an excessively large filter size for neighborhood smoothing can reduce quality by placing samples in empty space [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_oracle_inputs(ray_origin, ray_direction, near, far, num_bins, num_inputs):
    """
    Constructs the input vector for the DONeRF depth oracle.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray (normalized).
        near: float, near plane distance.
        far: float, far plane distance.
        num_bins: int, number of discretized depth ranges (N).
        num_inputs: int, number of 3D positions to provide (I).
        
    Returns:
        numpy array of shape (num_inputs, 3), the input vector.
    """
    # Step 1: Discretize depth ranges
    # Create num_bins evenly spaced depth values in log space to match shading network
    # Note: The source mentions logarithmic transformation on classification steps.
    # We assume the bins are defined in log-depth space for uniformity with the shading network.
    
    # Calculate log-depth bounds
    log_near = np.log(near)
    log_far = np.log(far)
    
    # Create num_bins intervals in log space
    log_depths = np.linspace(log_near, log_far, num_bins + 1)
    
    # Step 2: Calculate centers of discretized depth ranges
    # Centers are the midpoints of the log-depth intervals
    log_centers = (log_depths[:-1] + log_depths[1:]) / 2.0
    
    # Convert log-depths back to linear depth for 3D position calculation
    # Assuming exponential mapping: depth = exp(log_depth)
    centers_depth = np.exp(log_centers)
    
    # Step 3: Calculate 3D positions
    # p_i = ray_origin + ray_direction * centers_depth[i]
    # We need to select num_inputs positions from these centers.
    # The source implies providing multiple samples along the ray.
    # If num_inputs > num_bins, we might repeat or interpolate, but typically num_inputs <= num_bins or 
    # the source implies using the centers directly. 
    # Given I=128 and N tested up to 16, there is a discrepancy. 
    # Re-reading: "vary the number of samples per ray N = [2,4,8,16]" and "number of 3D input positions (I = [1,128])".
    # It is likely that the oracle inputs are a subset or specific selection of the ray, 
    # or the "classification steps" refer to the output bins, while inputs are specific query points.
    # However, the text says: "Place those at the centers of the discretized depth ranges".
    # If I > N, we must have more inputs than bins. 
    # Let's assume the standard case where inputs are drawn from the discretized ranges.
    # If I > N, we might use finer discretization for inputs or repeat. 
    # Given the ambiguity, we will assume the inputs are the centers of the bins used for classification.
    # If I != N, we adjust the discretization for the input construction specifically.
    
    # Let's assume the input construction uses its own discretization if I != N, 
    # or simply selects I points from the N bins. 
    # The most robust interpretation: The oracle predicts N bins. The input provides I context points.
    # The text says "Place those at the centers of the discretized depth ranges".
    # This implies the input points ARE the centers of the ranges.
    # Therefore, I should likely equal N, or the "discretized depth ranges" for input are different from output bins.
    # However, the ablation varies I and N independently.
    # Let's assume the input positions are calculated based on I points, logarithmically spaced.
    
    # Recalculate centers for I points
    log_input_depths = np.linspace(log_near, log_far, num_inputs + 1)
    log_input_centers = (log_input_depths[:-1] + log_input_depths[1:]) / 2.0
    input_depths = np.exp(log_input_centers)
    
    # Calculate 3D positions
    positions = ray_origin + ray_direction * input_depths[:, np.newaxis]
    
    # Step 4: Return the positions as the input vector
    # No encoding is applied [S1]
    return positions
```

## Subtopics

## Operating characteristics
*   **Performance Gain:** Providing multiple samples along the ray as input adds approximately 1 dB to PSNR [S1].
*   **Comparison:** Using this classification-based oracle input (instead of predicting a single depth value) adds 0.6 dB–1.5 dB to PSNR [S1].
*   **Cost:** These improvements come at no inference cost for filtering or virtually no inference cost for ray unification and multi-input [S1].
*   **Limitations:** Using an even larger filter size (related to context, not directly input construction) may reduce overall quality as samples are placed in empty space [S1].

## Background
*   **Origin:** This method is part of the DONeRF architecture, introduced in "DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks" [S1].
*   **Development:** It was developed to ease the task of the oracle network by supplying 3D positions along the ray, placed at the centers of discretized depth ranges, and aligning them with the shading network's logarithmic sampling [S1].

## Sources
[S1] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
