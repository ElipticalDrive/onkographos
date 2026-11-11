# GPU-based Fluid Simulation with Voxel Rendering

Also called: Integrating Real-Time Fluid Simulation with a Voxel Engine, Poxels Fluid Integration

## Overview
This method integrates real-time incompressible fluid simulation with sparse voxel-based rasterization on graphics hardware, specifically using Direct3D geometry shaders for rendering and compute shaders for simulation [S1]. It solves the problem of efficiently visualizing fluid dynamics within a voxel engine by offloading the Navier–Stokes solver and voxel activation logic entirely to the GPU, avoiding costly CPU-GPU data transfers [S1]. It sets itself apart from its siblings by combining a fixed-grid Eulerian fluid solver with a geometry-shader-driven voxel generation pipeline, whereas [Gaussian Splashing](1387-gaussian-splashing.md) uses point clouds and [Position Based Fluids](1765-position-based-fluids.md) uses Lagrangian particles [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D simulation domain defined as a "Chunk" with resolution $(N_x, N_y, N_z)$, typically $64 \times 64 \times 64$ [S1]. External forces (e.g., gravity, source injection) [S1].
*   **Output:** A 3D speed grid texture filled with single-precision floats representing the velocity field, and a rendered voxel mesh representing the fluid surface [S1].

**Data Structures:**
The simulation uses GPU compute textures composed of single-precision float 4 (`float4`) types [S1]:
1.  **Velocity Textures:** Three textures storing the current and intermediate velocity fields [S1].
2.  **Pressure Textures:** Two textures storing current and intermediate pressure values for the projection step [S1].
3.  **Active Voxel Texture:** A texture containing only non-zero velocity values, used to determine which voxels are active for rendering [S1].
4.  **Staging Texture:** A 3D texture used only in the CPU approach to transfer data from GPU to CPU memory [S1].

**Algorithm Steps:**
The simulation updates the velocity field at each time step $\Delta t$ using the following stages [S1]:

1.  **Advection:** Advect the velocity field using semi-Lagrangian advection. This involves tracing back from each grid cell to find the source velocity and interpolating [S1].
2.  **Diffusion:** Apply diffusion to the momentum resulting from fluid resistance [S1].
3.  **Force Application:** Apply external forces (static or dynamic) to the velocity field [S1].
4.  **Pressure Computation:** Compute the divergence of the velocity field [S1].
5.  **Pressure Projection:** Solve the pressure-Poisson equation (typically via Jacobi iteration) to obtain the pressure field [S1].
6.  **Gradient Subtraction:** Subtract the pressure gradient from the velocity field to enforce incompressibility [S1].
7.  **Voxel Activation (GPU-Optimized):** In the optimized GPU-only approach, a texture containing only non-zero values is generated. The geometry shader reads this texture to determine which voxels to render [S1].
8.  **Rendering:**
    *   Dispatch the geometry shader for stream amplification with stream-out to a cached vertex buffer [S1].
    *   Apply a color ramp function in the geometry shader, taking a float value (concentration) between 0 and 1 to return an RGB color [S1].
    *   Rasterize the generated triangle geometry using the vertex buffer [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GPU Compute Shader: Fluid Simulation Step
def simulate_fluid_step(velocity_textures, pressure_textures, forces, dt):
    # 1. Advection
    # Trace back from each grid point and interpolate velocity
    new_velocity = advect(velocity_textures.current, dt)
    
    # 2. Diffusion
    # Apply diffusion based on viscosity
    new_velocity = diffuse(new_velocity, dt)
    
    # 3. Force Application
    # Add external forces (gravity, sources)
    new_velocity = add_forces(new_velocity, forces)
    
    # 4. Pressure Projection
    # Calculate divergence
    divergence = calculate_divergence(new_velocity)
    
    # Solve pressure Poisson equation (e.g., Jacobi iteration)
    pressure = solve_pressure(divergence, pressure_textures)
    
    # Subtract pressure gradient
    final_velocity = new_velocity - gradient(pressure)
    
    return final_velocity

# GPU Geometry Shader: Voxel Rendering
def render_voxels(active_voxel_texture, vertex_buffer):
    # Stream amplification: For each active voxel in the texture
    for voxel in active_voxel_texture:
        if voxel.value != 0:
            # Generate cube geometry for the voxel
            vertices = generate_cube_geometry(voxel.position)
            
            # Apply color ramp based on concentration/value
            color = apply_color_ramp(voxel.value)
            
            # Output vertices with color to stream-out
            stream_out(vertices, color)
    
    # Rasterize the generated primitives
    rasterize(vertex_buffer)
```

**Parameters:**
*   **Chunk Size:** The simulation domain size. Common sizes tested: $32^3$, $64^3$, $96^3$, $128^3$ [S1].
*   **Texture Format:** Single precision float 4 (`float4`) for velocity and pressure textures [S1].
*   **Color Ramp:** A function mapping a float in $[0, 1]$ to an RGB color [S1].

**Edge Cases and Pitfalls:**
*   **Obstacles:** The basic simulation does not detect or respond to obstacles [S1].
*   **Memory Alignment:** In the CPU approach, the CPU memory buffer requires appropriate memory address alignment when copying from the staging texture [S1].
*   **Deterministic Ordering:** Stream-out from the geometry shader is used to guarantee deterministic ordering of rendered primitives [S1].

## Subtopics

### [Stable Fluids (GPU Implementation)](7906-stable-fluids-gpu-implementation.md)
A method for computing fluid velocity by discretizing the momentum equation on a fixed Eulerian grid and solving for velocity via advection, external forces, and pressure projection. It sets itself apart from its siblings by using the Jacobi method to solve the pressure-Poisson system because it is easy to implement efficiently on the GPU, and by employing semi-Lagrangian advection which is unconditionally stable. It builds on [GPU-based Fluid Simulation with Voxel Rendering](7400-gpu-based-fluid-simulation-with-voxel-rendering.md) by providing the specific simulation kernels required to update the fluid state on graphics hardware.

## Operating characteristics
**Time and Memory Complexity:**
*   The simulation scales linearly with the number of voxels in the chunk [S1].
*   Memory usage is dominated by the five compute textures (3 velocity, 2 pressure) plus the active voxel texture, all stored in GPU memory [S1].

**Measured Performance:**
*   **CPU Approach (GTX 950M, $64^3$ chunk):** 100 FPS (10.000 ms frame time) [S1]. The "Copy from staging texture to CPU" step takes 23,088,969 ns, while "Activate voxels and render" takes 976,276 ns [S1].
*   **GPU-Only Approach (GTX 950M, $64^3$ chunk):** 1100 FPS [S1].
*   **GPU-Only Approach (GT 540M, $64^3$ chunk):** 180 FPS (5.556 ms frame time) [S1].
*   **Scaling (GPU-Only, GT 540M):**
    *   $32^3$ (32,768 blocks): 652 FPS (1.534 ms) [S1].
    *   $64^3$ (262,144 blocks): 180 FPS (5.556 ms) [S1].
    *   $96^3$ (884,736 blocks): 58 FPS (17.241 ms) [S1].
    *   $128^3$ (2,097,152 blocks): 25 FPS (40.000 ms) [S1] (Note: Table 3 lists 128x128x128 as 25 FPS, but the text mentions 1100 FPS for 64x64x64 on GTX950M, indicating significant variation by hardware) [S1].

**Strengths and Limitations:**
*   **Strengths:** High performance due to elimination of CPU-GPU data transfer; deterministic rendering via stream-out; suitable for real-time applications [S1].
*   **Limitations:** Does not handle obstacles; limited to single-chunk simulation in the described implementation; requires GPU support for geometry shaders and compute shaders [S1].

**When to Use:**
*   Use when real-time fluid visualization in a voxel engine is required and GPU resources are available.
*   Avoid when obstacle collision detection is needed without further modification.

## Background
This method was proposed to address the hurdles of integrating GPU-accelerated fluid simulation with voxel-based rendering [S1]. It builds on recent results in GPU-accelerated fluid simulation (Crane et al. 2007, Harris 2004) and voxel-based rendering (Miller et al. 2014) [S1]. The fluid simulation is based on the Navier–Stokes differential equations (Stam 2003) [S1]. The voxel rendering engine used is Poxels (Miller et al. 2014) [S1]. The method was published in 2016 [S1].

## Sources
[S1] Integrating Real-Time Fluid Simulation with a Voxel Engine, https://doi.org/10.1007/s40869-016-0020-5, fetched 2026-10-26
