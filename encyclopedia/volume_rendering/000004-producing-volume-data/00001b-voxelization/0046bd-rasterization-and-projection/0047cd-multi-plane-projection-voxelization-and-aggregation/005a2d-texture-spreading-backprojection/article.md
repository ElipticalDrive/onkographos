# Texture Spreading Backprojection

Also called: Back-projection Initialization, Texture Spreading, X-ray Density Initialization

## Overview
Texture Spreading Backprojection is a GPU-accelerated volume reconstruction method that maps projection data onto volume slices using texture mapping [S1, unofficial]. It solves the problem of efficiently aggregating multi-angle projection data into a 3D volume without requiring intermediate merge operations [S1, unofficial]. It is distinguished from its sibling methods by grouping spreading operations per slice and using a large texture sheet for accumulation, which eliminates the need for a final merge step but requires all projections to remain in memory throughout the reconstruction [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of 2D projection images (one per projection angle), volume dimensions (width, height, depth/slices).
- **Outputs:** A 3D volume dataset (stack of slices).

**Data Structures:**
- **Projection Textures:** Each projection image is bound as a 1D texture [S1, unofficial].
- **Texture Sheet:** A large 2D texture container (sheet) composed of sub-textures, where each sub-texture holds the contribution from a single projection angle for the current slice being processed [S1, unofficial].
- **Slice Stack:** A buffer or texture stack representing the current volume slice being reconstructed; only one slice stack is maintained in memory at a time [S1, unofficial].

**Steps:**
1. **Initialize Back-projection Parameters:** Assume constant attenuation $\sigma_t = \sigma$ and unit ray length $L=1$, yielding $p = \sigma$ [S2, unofficial].
2. **Initialize Slice Processing:** Select the current volume slice index $z$ to reconstruct [S1, unofficial].
3. **Create Texture Sheet:** Allocate a large texture sheet containing sub-textures for each projection angle [S1, unofficial].
4. **Spread Projections:** For each projection angle $i$, bind the projection as a 1D texture and spread it across the buffer to form its contribution to the slice [S1, unofficial].
5. **Compute Initial Density:** Calculate the initial density of voxel-space points $C_{Vds}$ by averaging the pixel intensities at the projected locations across all views: $C_{Vds} = \frac{1}{n} \sum_{o=1}^n I_o[P_{I_o}]$ [S2, unofficial].
6. **Multi-Pass Accumulation:** Execute a fragment shader pass over the texture sheet to sum the sub-texture tiles, completing the reconstruction of the volume slice [S1, unofficial].
7. **Store and Advance Slice:** Save the accumulated result for slice $z$ to the output volume stack, increment $z$, and repeat steps 2–6 until all slices are reconstructed [S1, unofficial].

**Formulas:**
$p = \sigma$ [S2, unofficial]
$C_{Vds} = \frac{1}{n} \sum_{o=1}^n I_o[P_{I_o}]$ [S2, unofficial]
where $I_o$ denotes the incident pixel intensity at projected location $P_{I_o}$ for view $o$, and $n$ is the total number of views [S2, unofficial].

**Parameters:**
- **Number of Projections ($n$):** Determined by the acquisition geometry [S2, unofficial].
- **Texture Sheet Size:** Must be large enough to hold all $n$ sub-textures [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Constraints:** All projections must remain in GPU memory until the entire volume is reconstructed, which can cause memory traffic congestion and pipeline stalls on GPUs with limited bandwidth [S1, unofficial].
- **Precision:** Using integer representations for input data can reduce memory traffic and alleviate bandwidth issues, but may introduce quantization errors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_spreading_backprojection(projections, volume_dims):
    """
    Reconstructs a volume using texture spreading backprojection.
    """
    width, height, depth = volume_dims
    num_projections = len(projections)
    volume = [[[0.0] * width for _ in range(height)] for _ in range(depth)]
    
    # Initialize back-projection parameters: assume constant attenuation sigma_t = sigma and unit ray length L=1 [S2, unofficial]
    
    for z in range(depth):
        # Create a large texture sheet containing sub-textures for each projection angle [S1, unofficial]
        sheet = [[[0.0] * width for _ in range(height)] for _ in range(num_projections)]
        
        # Spread each projection onto the sheet's sub-texture region
        for i in range(num_projections):
            # Bind projection i as 1D texture and spread across buffer [S1, unofficial]
            # Implementation: map projection pixels to slice coordinates via geometric transform
            for y in range(height):
                for x in range(width):
                    # Geometric projection mapping (sources do not specify exact transform)
                    # [Comment: Sources do not specify exact geometric transform for spreading]
                    pass
        # Compute initial density by averaging pixel intensities at projected locations across all views [S2, unofficial]
        # C_Vds = (1/n) * sum(I_o[P_I_o])
        for y in range(height):
            for x in range(width):
                intensity_sum = sum(sheet[i][y][x] for i in range(num_projections))
                volume[z][y][x] = intensity_sum / num_projections
        
        # Multi-pass accumulation step to sum tiles (sources do not specify exact shader passes)
        # [Comment: Sources do not specify multi-pass accumulation details]
        
    return volume
```

## Subtopics

### Voxel-Centric Transformation Backprojection
An alternative implementation where transformation matrices are computed for each voxel to designate its sampling locations on the projections [S1, unofficial]. Instead of spreading projection data onto a texture sheet, this method iterates through all projection angles in the fragment shader for each voxel, fetching and summing sample values from the projections directly [S1, unofficial]. [Comment: The evidence does not specify performance trade-offs or memory access patterns for this variant.]

## Operating characteristics
- **Memory Usage:** Requires all projection images to remain in GPU memory simultaneously until the entire volume is reconstructed [S1, unofficial].
- **Time Complexity:** Proportional to the number of voxels times the number of projections, $O(W \times H \times D \times N)$, where $W,H,D$ are volume dimensions and $N$ is the number of projections [S1, unofficial].
- **Performance:** Can suffer from memory bandwidth bottlenecks due to repeated access to large projection datasets [S1, unofficial]. Using integer representations for input data can help alleviate bandwidth issues [S1, unofficial].
- **Strengths:** No merge operation is needed at the end, as only one slice stack is maintained [S1, unofficial].
- **Limitations:** High memory consumption; unsuitable for datasets exceeding available GPU memory [S1, unofficial].

## Background
Texture Spreading Backprojection was originally used for parallel-beam projection geometry by [CM03] and later extended to cone-beam geometry by [MX06] [S1, unofficial]. It is a technique developed to leverage GPU texture mapping hardware for efficient volume reconstruction from projections [S1, unofficial].

## Sources
[S1] GPU-Accelerated D2VR, https://doi.org/10.2312/vg/vg06/023-030, fetched 2026-10-26 (unofficial)
[S2] LINGO: Latent Initialization and Gradient Optimization for Sparse-view X-ray Novel View Synthesis and CT Reconstruction with 3D Gaussian Splatting, https://arxiv.org/abs/2609.22849, fetched 2026-10-27 (unofficial)
