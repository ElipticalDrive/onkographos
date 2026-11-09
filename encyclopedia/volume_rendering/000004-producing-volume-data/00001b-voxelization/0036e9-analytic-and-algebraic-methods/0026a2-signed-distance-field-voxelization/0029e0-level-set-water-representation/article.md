# Level Set Water Representation

Also called: Level Set Method for Liquid Surface

## Overview
The Level Set Water Representation tracks a liquid's free surface on a 3D grid by storing the signed distance from each cell center to the water-air interface [S1, unofficial]. It distinguishes itself from sibling methods by requiring only a scalar value per cell, making it particularly well suited to GPU implementation [S1, unofficial]. It builds on [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md) by advecting the distance field with fluid velocity rather than computing it statically from geometry.

## How it works
**Inputs and Outputs**
- **Input**: A 3D grid of resolution $N_x \times N_y \times N_z$, a fluid velocity field $\mathbf{u}$, and an initial level set field $\phi^0$ [S1, unofficial].
- **Output**: A time-evolved level set field $\phi^{n+1}$ representing the water surface [S1, unofficial].

**Data Structures**
- **Level Set Field**: A 3D texture or array storing a scalar value $\phi_{i,j,k}$ for each cell $(i,j,k)$ [S1, unofficial].
- **Velocity Field**: A 3D vector field $\mathbf{u}_{i,j,k}$ storing the fluid velocity at each cell [S1, unofficial].

**Steps**
1. **Initialization**: Set $\phi$ such that $\phi < 0$ inside the water and $\phi \geq 0$ outside (air) [S1, unofficial]. The zero set $\phi = 0$ defines the water surface [S1, unofficial].
2. **Advection**: Advect the level set field using the fluid velocity [S1, unofficial]. This moves the surface according to the flow [S1, unofficial].
3. **Domain Definition**: The simulation domain is defined by $\phi < 0$ [S1, unofficial]. Forces (gravity, buoyancy) and pressure solves are applied only in cells where $\phi < 0$ [S1, unofficial]. Cells with $\phi \geq 0$ are treated as air with zero pressure and no external forces [S1, unofficial].
4. **Reinitialization (Optional)**: Periodically reinitialize $\phi$ to ensure it remains a true signed distance field (shortest distance to the zero set) [S1, unofficial]. This is required for accurate normal computation but is optional for real-time animation where visual quality suffices without strict distance preservation [S1, unofficial].

**Formulas**
$$ \phi_{i,j,k}^{n+1} = \begin{cases} A(\phi^n)_{i,j,k} & \phi_{i,j,k}^{\infty} \geq 0 \\ (1-\beta)A(\phi^n)_{i,j,k} + \beta\phi_{i,j,k}^{\infty} & \phi_{i,j,k}^{\infty} < 0 \end{cases} $$

**Parameters**
- $\beta \in [0, 1]$: Damping parameter controlling the amount of damping applied to the solution from advection [S1, unofficial]. Larger values allow fewer pressure solver iterations but reduce fluid liveliness [S1, unofficial].
- $\phi^{\infty}$: Equilibrium level set. For a tank of water with height $h$, $\phi^{\infty}(x,y,z) = y - h$, where $y$ is the vertical distance from the tank bottom [S1, unofficial].

**Edge Cases and Pitfalls**
- **Volume Gain**: Damping is applied only where $\phi^{\infty} < 0$ [S1, unofficial]. This keeps splashes outside the equilibrium domain lively but can result in temporary volume gain [S1, unofficial].
- **Arbitrary Environments**: This method fails if the equilibrium state is unknown beforehand (e.g., water in a bowl placed at an arbitrary location), as it is a nonphysical hack [S1, unofficial].
- **Reinitialization Cost**: Full reinitialization is computationally expensive [S1, unofficial]. For real-time animation, skipping reinitialization is acceptable if the visual result is decent [S1, unofficial].

## Subtopics

### Equilibrium Level Set Damping
A technique to force liquid toward static equilibrium when the pressure solve is under-converged, preventing sinking through tank floors [S1, unofficial]. It blends the advected level set with a known equilibrium level set [S1, unofficial]. Choose this when simulating contained liquids with known static states and limited solver iterations [S1, unofficial]. It is not suitable for arbitrary environments where the equilibrium state is unknown [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: GPU-friendly due to requiring only a scalar value per cell [S1, unofficial].
- **Limitations**: Requires reinitialization for accurate normals; damping is a nonphysical hack that can cause temporary volume gain; under-converged pressure solves can cause artifacts without damping [S1, unofficial].
- **When to Use**: Real-time fluid simulation where performance is critical and topological changes are frequent [S1, unofficial].
- **When Not to Use**: When physical accuracy of volume conservation is paramount without hacks, or in environments with unknown equilibrium states [S1, unofficial].

## Background
The level set method for liquid surfaces was popularized by Sethian (1999) and adapted for real-time GPU fluid simulation in *GPU Gems 3* (2008) [S1, unofficial]. It is widely used in real-time rendering engines for its efficiency and ability to handle complex fluid dynamics on the GPU [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
