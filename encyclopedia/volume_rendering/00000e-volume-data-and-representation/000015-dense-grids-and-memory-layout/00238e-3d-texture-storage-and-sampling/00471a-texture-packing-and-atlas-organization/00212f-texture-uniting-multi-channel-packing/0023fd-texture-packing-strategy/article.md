# Texture Packing Strategy

Also called: Separate Scalar Textures for Bandwidth Optimization

## Overview
This method reduces GPU memory consumption and bandwidth usage by mapping a single underlying volume dataset to multiple separate scalar textures rather than packing multiple cell attributes into a single multi-channel texture. It solves the problem of unnecessary memory bandwidth overhead when only a subset of volume attributes is required for a specific computation. It differs from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md) by avoiding the aggregation of data into RGBA formats, thereby preventing the transfer of unused channels.

## How it works
**Inputs and Outputs:**
*   Input: A set of volume attributes (e.g., inside-outside status, velocity, density) derived from a single logical volume dataset.
*   Output: A set of individual 3D scalar textures, one per attribute.

**Data Structures:**
*   Each attribute is stored in a distinct 3D texture object.
*   Format: Typically 16-bit or 32-bit floating point per channel, depending on precision requirements [S1, unofficial].
*   Layout: Standard 3D texture memory layout (x, y, z dimensions).

**Steps:**
1.  Identify the set of volume attributes required for the current rendering or simulation pass.
2.  Determine which attributes are accessed frequently and which are rarely or never accessed in the current context.
3.  For attributes that are frequently accessed (e.g., inside-outside masks for pressure solving), allocate and store them in their own dedicated scalar textures.
4.  For attributes that are not needed for the current pass, do not load them into the texture unit or skip sampling them.
5.  During sampling, access only the relevant scalar textures.

**Formulas:**
*   Memory Transfer Cost (Packed): $C_{packed} = 4 \times N \times B$
*   Memory Transfer Cost (Separate): $C_{separate} = N \times B$
    *   Where $N$ is the number of voxels/pixels and $B$ is the bytes per voxel.
    *   Packing $k$ attributes into a single texture increases transfer by a factor of $k$ if only one attribute is needed.

**Parameters:**
*   Precision: 16-bit storage is often used, but arithmetic operations are performed in 32-bit floating point, leading to periodic truncation upon write [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bandwidth Inefficiency:** Packing inside-outside and velocity into a single RGBA texture increases memory transfer by 4x. If velocity is unused during pressure solving, this is highly inefficient [S1, unofficial].
*   **Precision Loss:** Using 16-bit storage can cause degradation in results due to truncation when writing back to textures, even if computation is done in 32-bit [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume volume_data contains logical attributes: inside_outside, velocity, density

# Step 1: Allocate separate scalar textures for frequently accessed attributes
# This avoids packing unused data into multi-channel textures
texture_inside_outside = create_3d_texture(width, height, depth, format=FLOAT16)
texture_velocity = create_3d_texture(width, height, depth, format=FLOAT16) # Only if needed

# Step 2: Copy data to separate textures
# Only copy the data that is actually required for the current pass
if pass_type == "PRESSURE_SOLVER":
    copy_to_texture(volume_data.inside_outside, texture_inside_outside)
    # Do NOT copy velocity to save bandwidth
elif pass_type == "RENDERING":
    copy_to_texture(volume_data.inside_outside, texture_inside_outside)
    copy_to_texture(volume_data.velocity, texture_velocity)

# Step 3: Sample only the necessary textures
if pass_type == "PRESSURE_SOLVER":
    # Sample only the scalar texture for inside-outside
    inside_value = sample_3d_texture(texture_inside_outside, uvw)
    # Velocity is not sampled, saving 3/4 of the bandwidth for this attribute
else:
    inside_value = sample_3d_texture(texture_inside_outside, uvw)
    velocity_value = sample_3d_texture(texture_velocity, uvw)
```

## Subtopics

### [Texture Packing Strategy — Separate Scalar Textures for Bandwidth Optimization](7033-texture-packing-strategy-separate-scalar-textures-for-bandwi.md)
This method reduces GPU memory consumption by mapping a single underlying volume dataset to multiple texture units, applying scalar multipliers to adjust lookup values for different physical properties (emission, absorption, reflection). It differs from alpha-channel dissociation strategies by avoiding the need to split geometry into non-intersecting subsets; instead, it relies on the mathematical relationship that volume attributes often differ only by scalar factors. This approach minimizes host-to-GPU data transfer by copying volumes only when modified.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for data transfer, where $N$ is the number of voxels. Transfer time is proportional to the number of active textures.
*   **Memory Complexity:** $O(N \times A)$, where $A$ is the number of active attributes. However, by separating textures, unused attributes do not consume bandwidth during sampling.
*   **Strengths:** Minimizes memory bandwidth usage when only a subset of attributes is needed. Reduces cache pressure.
*   **Limitations:** Increases the number of texture units required. May increase driver overhead due to multiple texture bindings.
*   **When to use:** When specific attributes (e.g., inside-outside) are accessed far more frequently than others (e.g., velocity) in the same frame or pass.
*   **When not to use:** When all attributes are accessed equally and frequently, or when texture unit limits are reached.

## Background
Originating from real-time fluid simulation and rendering techniques, this strategy is discussed in the context of optimizing GPU memory bandwidth for 3D fluids [S1, unofficial]. It is particularly relevant in scenarios like pressure-Poisson equation solving where only specific scalar fields are required.

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
