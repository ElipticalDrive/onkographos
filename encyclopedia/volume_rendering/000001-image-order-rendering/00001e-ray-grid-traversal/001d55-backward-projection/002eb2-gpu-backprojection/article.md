# GPU Backprojection

Also called: Scale field rendering, Scale-Space Backprojection, Weighted blob blending

## Overview
GPU Backprojection is a rendering pass that visualizes detected scale-space blobs by generating small slices for each blob, computing weighted contributions, and blending them to form the final image, followed by normalization [S2, unofficial]. It decouples the expensive scale-space computation from the frequent backprojection, allowing interactive adjustment of smoothness and visibility parameters [S2, unofficial].

## How it works
**Inputs:**
- Scale abstraction: A set of detected scale-space blobs, each defined by its center position, radius, and weight.
- Diffusion parameters: Used to determine blob properties (recomputed only when these change) [S2, unofficial].

**Outputs:**
- Final normalized image.

**Data Structures:**
- Scale Abstraction: Stored on the CPU, containing blob metadata (position, radius, weight) [S2, unofficial].
- Accumulation Buffers: Two render targets (textures) on the GPU:
    1. Color/Value Buffer: Stores the weighted sum of contributions.
    2. Weight Buffer: Stores the sum of weights for normalization.

**Steps:**
1. **Slice Generation:** For each detected scale-space blob, generate a series of small slices. The sides of these slices are proportional to the blob's radius [S1][S2, unofficial].
2. **Weighted Contribution:** For each pixel covered by the slices of a blob, compute the weighted contribution of that blob. The weight is derived from the blob's properties [S2, unofficial].
3. **Blending:** Use alpha blending to accumulate the sum of weighted contributions from all blobs into the Color/Value Buffer. Simultaneously, accumulate the sum of weights into the Weight Buffer [S2, unofficial].
4. **Normalization:** In a separate pass, normalize the accumulated result in the Color/Value Buffer by dividing it by the sum of weights stored in the Weight Buffer [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume blob_list contains objects with attributes: center, radius, weight
# Assume render_target_color and render_target_weight are GPU textures initialized to 0

def gpu_backprojection(blob_list, resolution):
    # Pass 1: Accumulate weighted contributions and weights
    for blob in blob_list:
        # Generate slices proportional to blob radius
        # Each slice covers a region in the image plane
        slices = generate_slices(blob.center, blob.radius, resolution)
        
        for slice in slices:
            # Compute weighted contribution for pixels in this slice
            # The exact weighting function is not specified in sources [G]
            # Typically involves distance from center and blob weight
            contribution = compute_weighted_contribution(slice, blob)
            weight = blob.weight
            
            # Blend into color buffer
            render_target_color.blend_add(slice.region, contribution)
            
            # Add to weight buffer
            render_target_weight.add(slice.region, weight)

    # Pass 2: Normalize
    # Divide color buffer by weight buffer
    normalized_image = render_target_color.divide(render_target_weight)
    
    return normalized_image

def generate_slices(center, radius, resolution):
    # Generate small slices with sides proportional to radius
    # This is a placeholder for the specific geometric generation
    # The sources do not specify the exact slice geometry or count [G]
    pass

def compute_weighted_contribution(slice, blob):
    # Compute the weighted contribution of the blob for the slice
    # The exact formula is not provided in the sources [G]
    pass
```

**Edge Cases and Pitfalls:**
- **Zero Weight:** If the sum of weights is zero for a pixel, division by zero occurs during normalization. A guard is needed to handle this case [G].
- **Blob Overlap:** Multiple blobs may contribute to the same pixel. The blending process handles this, but the order of processing may affect precision if not using proper accumulation [G].
- **Resolution:** The size of the slices is proportional to the blob radius. If the radius is very small, the slices may not cover enough pixels, leading to aliasing or loss of detail [G].

## Operating characteristics
**Time Complexity:**
- Scale-space computation: $O(N)$ where $N$ is the number of voxels, done once or when diffusion parameters change [S2, unofficial].
- Backprojection: $O(B \cdot S)$ where $B$ is the number of blobs and $S$ is the average number of slices per blob [G].

**Memory Complexity:**
- Requires storage for the scale abstraction on the CPU and two accumulation buffers on the GPU [S2, unofficial].

**Measured Performance:**
- On an Intel Core 2 Duo 2.4 GHz with 2GB RAM and an nVidia GeForce 8800 GTX with 768 MB texture memory, the GPU implementation showed an improvement of about two orders of magnitude over the CPU version for data set sizes of $64^3$, $128^3$, and $256^3$ [S2, unofficial].
- The CPU implementation was deemed impractical for interactive visualization [S2, unofficial].

**Strengths and Limitations:**
- **Strengths:** Decoupling allows for interactive adjustment of parameters. GPU acceleration provides significant speedup [S2, unofficial].
- **Limitations:** Reliance on a temporary volume to compute the scale field. The approach may not be real-time for very large datasets or complex blob structures without further optimization [S2, unofficial].

## Background
GPU Backprojection is part of the "Size-based Transfer Functions: A New Volume Exploration Technique" presented by Correa et al. in 2008 [S1][S2, unofficial]. It was developed to address the computational cost of scale-space computation in volume rendering, enabling interactive exploration by separating the heavy computation from the rendering pass.

## Sources
[S1] Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
[S2] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
