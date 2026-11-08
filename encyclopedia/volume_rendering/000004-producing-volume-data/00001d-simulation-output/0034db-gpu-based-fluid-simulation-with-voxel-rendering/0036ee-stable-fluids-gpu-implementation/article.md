# Stable Fluids (GPU Implementation)
Also called: Jacobi Pressure Solver, Jacobi method for pressure-Poisson system, Semi-Lagrangian Advection

## Overview
Stable Fluids (GPU Implementation) is a method for computing incompressible fluid velocity by discretizing the momentum equation on a fixed Eulerian grid and solving for velocity via advection, external forces, and pressure projection [S1, unofficial]. It solves the problem of real-time fluid simulation on graphics hardware by leveraging 3D textures and slice-based rasterization to execute computational kernels [S1, unofficial]. It differs from its parent method, [GPU-based Fluid Simulation with Voxel Rendering](7400-gpu-based-fluid-simulation-with-voxel-rendering.md), by specifically defining the numerical kernels (advection, divergence, Jacobi solve, projection) and data layouts required to perform the simulation steps on the GPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D grid of cubical cells representing the simulation domain; initial velocity field $\mathbf{u}$; pressure field $p$; external forces $\mathbf{f}$ (e.g., gravity); time step $\Delta t$; number of Jacobi iterations.
*   **Outputs:** Updated velocity field $\mathbf{u}^{n+1}$; updated pressure field $p^{n+1}$.

**Data Structures:**
Cell attributes (velocity, pressure) are stored in 3D textures, mapping grid cells to voxels [S1, unofficial]. The geometry shader output structure for the simulation kernel is defined as follows [S1, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct GS_OUTPUT_FLUIDSIM {
    // Index of the current grid cell (i,j,k in [0,gridSize] range)
    float3 cellIndex : TEXCOORD0;
    // Texture coordinates (x,y,z in [0,1] range) for the
    // current grid cell and its immediate neighbors
    float3 CENTERCELL : TEXCOORD1;
    float3 LEFTCELL : TEXCOORD2;
    float3 RIGHTCELL : TEXCOORD3;
    float3 BOTTOMCELL : TEXCOORD4;
    float3 TOPCELL : TEXCOORD5;
    float3 DOWNCELL : TEXCOORD6;
    float3 UPCELL : TEXCOORD7;
    float4 pos : SV_Position; // 2D slice vertex in homogeneous clip space
    uint RTIndex : SV_RenderTargetArrayIndex; // Specifies destination slice
};
```

**Numbered Steps:**
The momentum equation solution is split into successive operations: advection, application of external forces, and pressure projection [S1, unofficial].

1.  **Advection (Semi-Lagrangian):**
    *   Trace back along the characteristic: $\mathbf{pos} = \mathbf{current\_pos} - \Delta t \cdot \mathbf{cellVelocity}$ [S1, unofficial].
    *   Convert the resulting index to texture coordinates and sample the velocity field using linear interpolation [S1, unofficial].
    *   Formula: $\mathbf{u}^{advected} = \text{Sample}(\mathbf{u}, \mathbf{pos})$ [S1, unofficial].

2.  **External Forces:**
    *   Add external forces (e.g., gravity) to the advected velocity [S1, unofficial].
    *   Formula: $\mathbf{u}^* = \mathbf{u}^{advected} + \Delta t \cdot \mathbf{f}$ [S1, unofficial].

3.  **Divergence Calculation:**
    *   Compute velocity divergence using central differences on neighboring cell velocities [S1, unofficial].
    *   Formula: $\nabla \cdot \mathbf{u}^* = 0.5 \cdot ((u_R - u_L) + (v_T - v_B) + (w_U - w_D))$ [S1, unofficial].

4.  **Pressure Solve (Jacobi Iteration):**
    *   Solve the pressure-Poisson system $\nabla^2 p = \nabla \cdot \mathbf{u}^*$ using the Jacobi method [S1, unofficial].
    *   Iterate $N_{iter}$ times. For each iteration, compute the new pressure at the center cell from its 6 neighbors and the divergence [S1, unofficial].
    *   Formula: $p_{new} = \frac{p_L + p_R + p_B + p_T + p_D + p_U - \nabla \cdot \mathbf{u}^*}{6.0}$ [S1, unofficial].

5.  **Projection:**
    *   Subtract the pressure gradient from the velocity to make it divergence-free [S2, unofficial].
    *   Compute pressure gradient: $\nabla p = 0.5 \cdot (p_R - p_L, p_T - p_B, p_U - p_D)$ [S1, unofficial].
    *   Update velocity: $\mathbf{u}^{n+1} = \mathbf{u}^* - \nabla p$ [S1, unofficial].

**Pseudocode:**
The following pseudocode implements the kernels described in [S1, unofficial]. It assumes a 3D texture sampling function `Sample(texture, coords)` and a coordinate conversion function `cellIndex2TexCoord(index)`.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel: Semi-Lagrangian Advection
def PS_ADVECT_VEL(cellIndex, velocity_texture, timeStep):
    # Get current cell velocity
    center_coords = cellIndex2TexCoord(cellIndex)
    cell_velocity = velocity_texture.Sample(center_coords)
    
    # Trace back along characteristic
    pos = cellIndex - timeStep * cell_velocity
    pos = cellIndex2TexCoord(pos)
    
    # Sample velocity at traced position
    return velocity_texture.Sample(pos)

# Kernel: Divergence
def PS_DIVERGENCE(cellIndex, velocity_texture):
    # Get velocities from neighbors
    u_L = velocity_texture.Sample(cellIndex2TexCoord(cellIndex + (-1, 0, 0))).x
    u_R = velocity_texture.Sample(cellIndex2TexCoord(cellIndex + (1, 0, 0))).x
    v_B = velocity_texture.Sample(cellIndex2TexCoord(cellIndex + (0, -1, 0))).y
    v_T = velocity_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 1, 0))).y
    w_D = velocity_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 0, -1))).z
    w_U = velocity_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 0, 1))).z
    
    # Compute divergence using central differences
    divergence = 0.5 * ((u_R - u_L) + (v_T - v_B) + (w_U - w_D))
    return divergence

# Kernel: Jacobi Pressure Solve
def PS_JACOBI(cellIndex, pressure_texture, divergence_texture):
    # Get divergence at current cell
    d_C = divergence_texture.Sample(cellIndex2TexCoord(cellIndex))
    
    # Get pressures from neighbors
    p_L = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (-1, 0, 0)))
    p_R = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (1, 0, 0)))
    p_B = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, -1, 0)))
    p_T = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 1, 0)))
    p_D = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 0, -1)))
    p_U = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 0, 1)))
    
    # Compute new pressure
    new_pressure = (p_L + p_R + p_B + p_T + p_D + p_U - d_C) / 6.0
    return new_pressure

# Kernel: Projection
def PS_PROJECT(cellIndex, pressure_texture, velocity_texture):
    # Compute pressure gradient
    p_L = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (-1, 0, 0)))
    p_R = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (1, 0, 0)))
    p_B = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, -1, 0)))
    p_T = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 1, 0)))
    p_D = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 0, -1)))
    p_U = pressure_texture.Sample(cellIndex2TexCoord(cellIndex + (0, 0, 1)))
    
    grad_P = 0.5 * float3(p_R - p_L, p_T - p_B, p_U - p_D)
    
    # Get old velocity
    v_old = velocity_texture.Sample(cellIndex2TexCoord(cellIndex))
    
    # Project velocity
    v_new = v_old - grad_P
    return v_new
```

**Parameters:**
*   `timeStep` ($\Delta t$): The duration of the time step used in semi-Lagrangian advection [S1, unofficial].
*   `Iterations`: The number of Jacobi iterations. For smoke/fire, this can be adjusted based on available resources; for liquids, more iterations are required to maintain volume [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **GPU Rendering Limitations:** GPUs render into 2D buffers, so 3D volume kernels must be executed once per slice by iterating over slice indices [S1, unofficial].
*   **Numerical Smoothing:** Semi-Lagrangian advection is unconditionally stable but introduces numerical smoothing, making water look viscous or smoke lose detail [S1, unofficial].
*   **Volume Loss:** Too few Jacobi iterations for liquids cause volume loss, making water sink through the tank bottom because pressure from the floor cannot counteract gravity [S1, unofficial].

## Subtopics

### Projection Method
A technique to enforce incompressibility (zero divergence) on a velocity field by subtracting the gradient of a pressure field [S2, unofficial]. It involves solving the Poisson equation for pressure $p$ using Jacobi iteration (typically 20-40 iterations) and then subtracting the gradient of the pressure field from the velocity field to make it divergence-free [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The dominant cost is the Jacobi pressure solve, which requires $O(N \cdot I)$ operations, where $N$ is the number of cells and $I$ is the number of iterations [S1, unofficial].
*   **Memory Complexity:** Storage is proportional to the number of cells. Table 30-1 in [S1, unofficial] indicates 9 bytes per cell for simulation (1×RGBA16 velocity, 1×R8 inside-outside, 1×RGBA16 temporary).
*   **Performance:** The method is designed for real-time performance on GPUs [S1, unofficial]. The number of Jacobi iterations can be adjusted to balance quality and speed [S1, unofficial].
*   **Strengths:** Easy to implement on GPU; unconditionally stable advection; real-time capable [S1, unofficial].
*   **Limitations:** Numerical smoothing from semi-Lagrangian advection; volume loss if Jacobi iterations are insufficient for liquids [S1, unofficial].
*   **When to Use:** Real-time fluid simulation where visual quality is prioritized over physical accuracy; smoke and fire effects [S1, unofficial].
*   **When Not to Use:** High-quality animation requiring precise volume conservation; scenes with complex boundary conditions requiring more robust solvers [S1, unofficial].

## Background
The "stable fluids" method for computing velocity was introduced by Jos Stam in 1999 [S1, unofficial]. A GPU implementation of this method for 2D fluids was presented by Mark Harris in 2004 [S1, unofficial]. The 3D GPU implementation described here is detailed in GPU Gems 3 [S1, unofficial]. Alternatives to the Jacobi solver include the Conjugate Gradient method (Bolz et al. 2003), Multigrid (Goodnight et al. 2003), and Cyclic reduction (Kass et al. 2006) [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] tzhazuma/Fluid_simulation, https://github.com/tzhazuma/Fluid_simulation, fetched 2026-10-26 (unofficial)
