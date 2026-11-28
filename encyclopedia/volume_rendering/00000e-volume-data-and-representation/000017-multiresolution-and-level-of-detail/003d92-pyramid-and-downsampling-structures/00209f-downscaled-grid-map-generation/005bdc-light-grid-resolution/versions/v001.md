# Light Grid Resolution

Also called: emLightsGrid

## Overview
Light Grid Resolution defines the sampling density of a secondary grid used to approximate fire self-illumination in voxel-based fire simulations. It solves the performance bottleneck of calculating Global Illumination (GI) by placing a discrete light source in each cell of a downscaled grid rather than the full-resolution fire grid. It differs from sibling methods like [Index-Based Upsampling with Recalibration](2352-index-based-upsampling-with-recalibration.md) by operating at the light source generation stage (pre-radiance transfer) rather than during texture coordinate remapping.

## How it works
**Inputs and Outputs:**
- Input: Fire grid resolution ($R_{fire}$), Light Grid Resolution percentage ($P$).
- Output: A light grid with resolution $R_{light}$ containing one Omni light per cell.

**Data Structures:**
The light grid is a 3D array of light sources. Each cell $(i, j, k)$ in the light grid maps to a corresponding region in the fire grid. The light source is an Omni light, which emits radiation equally in all directions.

**Steps:**
1. Calculate the light grid resolution dimensions ($W_{light}, H_{light}, D_{light}$) based on the percentage $P$ of the fire grid dimensions ($W_{fire}, H_{fire}, D_{fire}$):
   $$ W_{light} = \lceil W_{fire} \times \frac{P}{100} \rceil $$
   $$ H_{light} = \lceil H_{fire} \times \frac{P}{100} \rceil $$
   $$ D_{light} = \lceil D_{fire} \times \frac{P}{100} \rceil $$
   [G] Ceiling is used to ensure at least 1 voxel exists if $P > 0$.

2. Initialize an empty light grid of size $W_{light} \times H_{light} \times D_{light}$.

3. Iterate through each cell $(i, j, k)$ of the light grid.

4. For each light grid cell, determine the corresponding volume in the fire grid. The mapping is typically a uniform downsampling where each light grid cell covers a block of fire grid cells.
   [G] The exact spatial mapping (e.g., center-to-center or block average) is not specified in sources, but standard practice implies the light source represents the average emission of the covered fire volume.

5. Populate the light grid cell $(i, j, k)$ with an Omni light. The intensity of this Omni light is derived from the fire data (temperature/density) within the corresponding fire grid region.

6. Use the light grid for illumination calculations during rendering. The fire smoke and surrounding objects are illuminated by these discrete Omni lights.

**Parameters:**
- $P$ (Light Grid Resolution %): Typical values range from 10 to 100.
  - $P=100$: One light per fire cell. Perfect illumination detail but slow rendering.
  - $P < 100$: Fewer lights. Faster rendering, smoother illumination.
  - $P \ll 100$: Very low resolution. May cause poor blending between fire and light on smoke.

**Edge Cases and Pitfalls:**
- **Memory Usage:** High resolution grids (high $P$) require significant RAM, especially with many copied or instanced volumes [S1].
- **Blending Artifacts:** Very low $P$ values may result in visible discontinuities or poor blending between the fire geometry and the illuminated smoke.
- **Shadowing:** Using Grid-based self-shadowing requires additional memory [S1].

## Subtopics

### [Fire Lights (GI Simulation)](17396-fire-lights-gi-simulation.md)
Simulates Global Illumination by placing lights in bright fire parts. Faster than native GI but creates persistent light sources.

## Operating characteristics
- **Time Complexity:** Rendering time is proportional to the number of lights in the light grid ($N_{lights} = W_{light} \times H_{light} \times D_{light}$). Reducing $P$ linearly reduces $N_{lights}$, speeding up rendering.
- **Memory Complexity:** Memory usage scales with $N_{lights}$. High $P$ or large fire grids increase RAM usage [S1].
- **Performance:** Lower resolution yields faster rendering at the expense of illumination detail [S1].
- **Strengths:** Significantly faster than full GI calculation; provides convincing fire self-illumination.
- **Limitations:** Loss of illumination detail at low $P$; potential blending artifacts; high memory cost at high $P$.
- **When to Use:** When real-time or fast rendering of fire self-illumination is needed, and perfect accuracy is not required.
- **When Not to Use:** When memory is constrained and high resolution is needed, or when perfect illumination detail is critical.

## Background
Originated in Phoenix for Maya, a voxel-based fire simulation tool. The method is part of the "Voxel Shader" feature set, which approximates complex lighting effects using grid-based light sources to avoid the computational cost of full ray-traced GI.

## Sources
[S1] Voxel Shader - Phoenix for Maya, https://documentation.chaos.com/space/PHX4MAYA/125768009, fetched 2026-10-29
