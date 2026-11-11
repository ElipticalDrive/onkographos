# Voxel-based 3D Fire Simulation

Also called: Eulerian Fire Simulation, Grid-based Fire

## Overview
This method simulates fire dynamics by storing velocity, smoke density, temperature, and fuel level attributes in a discrete 3D voxel grid [S1, unofficial]. It solves the Navier-Stokes equations to propagate fluid motion, augmented by specific buoyancy and vorticity forces to model the rising, curling behavior of fire [S1, unofficial]. It is distinguished from particle-based methods by its structured grid storage, which allows for efficient divergence and advection calculations using 3D textures [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: Initial 3D voxel grid containing velocity ($u$), pressure ($p$), density ($\rho$), temperature ($T$), and fuel level [S1, unofficial]. External parameters include grid resolution, time step $\Delta t$, and physical constants [S1, unofficial].
*   **Outputs**: Updated 3D textures for velocity, density, temperature, and fuel level for the next frame, suitable for volume rendering [S1, unofficial].

**Data Structures**
*   **Voxel Grid**: A 3D array of cells. In the reference implementation, the grid size is $128 \times 128 \times 128$ [S1, unofficial].
*   **Attributes**: Each cell stores scalar and vector fields:
    *   Velocity: Vector field $u$.
    *   Pressure: Scalar field $p$.
    *   Density: Scalar field $\rho$ (molecule mass density).
    *   Temperature: Scalar field $T$.
    *   Fuel Level: Scalar field.
*   **GPU Kernels**: Computed in blocks of $16 \times 16 \times 16$ threads, totaling 512 threads per block [S1, unofficial].

**Numbered Steps**
1.  **Initialize**: Set initial values for velocity, pressure, temperature, and fuel in the 3D voxel grid [S1, unofficial]. Ensure global variables (e.g., time step) are assigned in the driver script, not within the shader, to avoid zero-initialization bugs [S1, unofficial].
2.  **Compute Forces**: Calculate buoyancy force for each voxel based on local temperature [S1, unofficial]. Calculate vorticity from velocity, compute its magnitude gradient, normalize it, and compute the vorticity force [S1, unofficial]. Sum forces: $f = f_{buoyancy} + f_{vc}$ [S1, unofficial].
3.  **Advection and Divergence**: Compute advection term based on interactions between neighboring voxels [S1, unofficial]. Compute divergence of velocity to update pressure or enforce incompressibility [S1, unofficial].
4.  **Update Velocity**: Apply the Navier-Stokes update to propagate changes in velocity correspondingly [S1, unofficial].
5.  **Propagate Attributes**: Update temperature, pressure, and velocity fields correspondingly to the changes in dynamics [S1, unofficial]. Advect smoke density and fuel level using the velocity field [S1, unofficial].
6.  **Update Textures**: Write the updated values back to the 3D textures [S1, unofficial].
7.  **Render**: Inject a shader into the post-processing stage to ray march the 3D textures and render the fire [S1, unofficial].

**Formulas**
The simulation is based on the Navier-Stokes equation for incompressible flow [S1, unofficial]:
$$
\frac{\partial u}{\partial t} = -(u \cdot \nabla) u - \frac{1}{\rho} \nabla p + f
$$
Where:
*   $u$: Velocity vector.
*   $p$: Pressure scalar.
*   $\rho$: Density of molecule mass.
*   $f$: External force vector.
*   $-(u \cdot \nabla) u$: Advection term (transport of velocity by itself) [S1, unofficial].
*   $-\frac{1}{\rho} \nabla p$: Pressure gradient term (force generation from pressure gathering) [S1, unofficial].

The external force $f$ consists of buoyancy and vorticity forces [S1, unofficial]:
1.  **Buoyancy Force**:
    $$
    f_{buoyancy} = \frac{Pmg}{R} \left(\frac{1}{T_0} - \frac{1}{T}\right)z
    $$
    Where:
    *   $P$: Pressure constant.
    *   $m$: Mass constant.
    *   $g$: Gravity constant.
    *   $R$: Gas constant.
    *   $T_0$: Reference temperature.
    *   $T$: Current temperature.
    *   $z$: Vertical direction vector.
    Higher temperatures result in larger upward velocity [S1, unofficial].
2.  **Vorticity Force**:
    $$
    f_{vc} = \epsilon(\Psi \times \omega)\delta x
    $$
    Where:
    *   $\epsilon$: Scaling factor.
    *   $\omega$: Vorticity vector.
    *   $\delta x$: Spatial step or grid spacing.
    *   $\Psi$: Normalized gradient of vorticity magnitude.
3.  **Vorticity Gradient**:
    $$
    \Psi = \frac{\eta}{|\eta|}
    $$
    $$
    \eta = \nabla |\omega|
    $$
    Where $\eta$ is the gradient of the magnitude of vorticity [S1, unofficial]. This force restores curling behavior lost due to discretization [S1, unofficial].

**Parameters**
*   Voxel grid size: $128 \times 128 \times 128$ [S1, unofficial].
*   GPU block size: $16 \times 16 \times 16$ [S1, unofficial].
*   Threads per block: 512 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **HLSL Global Variables**: Global variables in HLSL compute shaders cannot be assigned within the shader file using an assignment operator during declaration; this results in zero values without a syntax error. The variable must be manually assigned in the C# driver script before each invocation of the compute kernels [S1, unofficial].
*   **Debugging**: Debugging attribute propagation across timestamps is difficult due to the lack of print debugging in shaders. Use debug shaders that fill 3D textures when values pass a threshold, or external tools like RenderDoc [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simulate_fire_step(grid, dt, constants):
    """
    Advances the fire simulation by one time step.
    grid: Object containing 3D arrays for u, p, rho, T, fuel, smoke_density.
    dt: Time step size.
    constants: Dict containing P, m, g, R, T0, epsilon, dx, z.
    """
    # 1. Calculate Vorticity and Vorticity Gradient
    # Standard finite difference curl: omega = grad x u [G]
    omega = compute_vorticity(grid.u)
    mag_omega = abs(omega)
    # Standard central difference gradient [G]
    eta = compute_gradient(mag_omega)
    norm_eta = eta / max(abs(eta), 1e-8)
    
    # 2. Calculate External Forces
    buoyancy = (constants['P'] * constants['m'] * constants['g'] / constants['R']) * \
               ((1.0 / constants['T0']) - (1.0 / grid.T)) * constants['z']
    
    vorticity_force = constants['epsilon'] * cross(norm_eta, omega) * constants['dx']
    f = buoyancy + vorticity_force
    
    # 3. Compute Navier-Stokes Terms
    # Standard (u . grad) field computation [G]
    advection = -compute_advection(grid.u, grid.u)
    pressure_grad = -(1.0 / grid.rho) * compute_gradient(grid.p)
    
    # 4. Update Velocity
    grid.u = grid.u + dt * (advection + pressure_grad + f)
    
    # 5. Propagate Attributes
    # Standard semi-Lagrangian or finite difference advection [G]
    grid.smoke_density = advect_scalar(grid.smoke_density, grid.u, dt)
    grid.fuel = advect_scalar(grid.fuel, grid.u, dt)
    # Source mentions propagating changes in temperature correspondingly [S1, unofficial]
    grid.T = update_temperature(grid.T, grid.fuel, dt)
    
    # 6. Update Textures
    write_to_texture(grid.u, "velocity")
    write_to_texture(grid.p, "pressure")
    write_to_texture(grid.T, "temperature")
    write_to_texture(grid.smoke_density, "density")
    write_to_texture(grid.fuel, "fuel")

def compute_vorticity(u):
    # Standard finite difference curl: omega = grad x u [G]
    pass # Source does not specify discretization method

def compute_gradient(field):
    # Standard central difference gradient [G]
    pass # Source does not specify discretization method

def compute_advection(u, field):
    # Standard (u . grad) field computation [G]
    pass # Source does not specify discretization method

def advect_scalar(scalar_field, velocity, dt):
    # Standard semi-Lagrangian or finite difference advection [G]
    pass # Source does not specify discretization method

def update_temperature(T, fuel, dt):
    # Source mentions propagating changes in temperature correspondingly [S1, unofficial]
    pass # Source does not specify temperature update formula

def write_to_texture(grid_data, name):
    # Standard GPU 3D texture upload [G]
    pass # Source does not specify API details
```

## Operating characteristics
*   **Performance**: Achieves real-time performance (60+ fps) on an NVIDIA RTX 2060 graphics card [S1, unofficial].
*   **Complexity**: Not given in the sources.
*   **Strengths**: Real-time capability; realistic fire behavior due to buoyancy and vorticity forces; structured grid allows efficient neighbor interactions [S1, unofficial].
*   **Limitations**: Resolution is limited by memory and compute power (128^3 in the reference); discrete nature may lose some fine curling details without vorticity force [S1, unofficial].
*   **When to use**: Not given in the sources [S1, unofficial].
*   **When not to use**: Not given in the sources [S1, unofficial].

## Background
This method is based on a project implementing a 3D fire simulation and rendering pipeline using the Navier-Stokes equations [S1, unofficial]. The approach combines fluid dynamics simulation with volume rendering techniques to create realistic fire effects [S1, unofficial].

## Sources
[S1] Fire Simulation and Rendering - GitHub Pages, https://cs184-firesim.github.io/final-report/, fetched 2026-10-27 (unofficial)
