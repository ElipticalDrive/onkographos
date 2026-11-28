# Iterative Radiative Transfer Equation (RTE) Solver for Light Diffusion

Also called: Diffusion Approximation for Volume Rendering, Diffusion Equation Method, Iterative RTE Solver, Light Diffusion, Light Diffusion Solver

## Overview
This method approximates light transport in participating media by reducing the Radiance Transfer Equation (RTE) to a diffusion equation, enabling interactive, iterative resolution via per-voxel kernels [S1, unofficial]. It visualizes structures and regions of interest in volumes without requiring global segmentation by simulating fluid-like light diffusion based on light transport [S1, unofficial]. Unlike ray-tracing or path-tracing, which are view-dependent and require recomputation on camera move, or photon mapping, which incurs high memory costs for storing the full photon map, this implicit representation allows efficient per-voxel implementation and avoids full-volume storage of light paths [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset defining material properties (extinction, scattering, emission) [S1, unofficial].
*   **Output:** A 3D buffer containing the computed fluence rate $\phi$ and potentially the irradiance vector $\vec{E}$, representing the diffused light field [S1, unofficial].

**Data Structures:**
*   Two 3D buffers are used in a Ping Pong scheme to store the diffusion result during iterative resolution [S1, unofficial].
*   This scheme allows access to data from a previous step during computation [S1, unofficial].
*   Memory usage requires 256 bits per voxel to store the diffusion result, in addition to the original data [S1, unofficial].

**Numbered Steps:**
1. Initialize two 3D buffers for ping-pong iterative resolution [S1, unofficial].
2. Apply a finite difference scheme with forward time and central space pattern to solve the diffusion equation [S1, unofficial].
3. Update the fluence rate $\phi$ iteratively [S1, unofficial].
4. Control parameters $D$, $K_a$, and $dt$ via transfer functions or automatic values based on the scattering albedo $\alpha$ [S1, unofficial].

**Formulas with Symbols Defined:**
The Radiance Transfer Equation (RTE) describes light transport [S1, unofficial]:
$$
\frac{1}{c} \frac{\partial L(p,t,\vec{\omega})}{\partial t} + \vec{\omega} \cdot \nabla L(p,t,\vec{\omega}) = -K_t(p) \cdot L(p,t,\vec{\omega}) + Q_e(p,t,\vec{\omega}) + \frac{K_s(p)}{4\pi} \int_{\Omega_2} P(p,\vec{\omega},\vec{\omega}') L_i(p,t,\vec{\omega}') d\vec{\omega}'
$$
Where $L(p,t,\vec{\omega})$ is radiance, $K_t(p)$ is the extinction coefficient ($m^{-1}$), $Q_e(p,t,\vec{\omega})$ is volumetric emission ($W.m^{-3}$), $P(p,\vec{\omega},\vec{\omega}')$ is the phase function ($sr^{-1}$), and $K_s(p)$ is the scattering coefficient [S1, unofficial].

The Diffusion Approximation assumes radiance $L$ is a combination of an isotropic term $\phi(p)$ and a direction-dependent term [S1, unofficial]:
$$
L(p,\vec{\omega}) = \frac{1}{4\pi}\phi(p) + \frac{3}{4\pi}\langle\vec{\omega}, \vec{E}(p)\rangle
$$
Where $\phi(p)$ is the fluence rate (spherical integral of radiance) and $\vec{E}(p)$ is the irradiance vector (flux) [S1, unofficial].

Taking the moments of order 0 and 1 of the RTE and applying the steady-flux approximation ($\frac{\partial \vec{E}}{\partial t} \approx 0$) leads to the Diffusion Equation for fluence rate $\phi$ [S1, unofficial]:
$$
\frac{1}{c} \frac{\partial \phi(p,t)}{\partial t} = D(p)\Delta_p \phi(p,t) - \langle\nabla_p D(p), \nabla_p \phi(p,t)\rangle - K_a(p)\phi(p,t) + \phi_e(p,t)
$$
Where $D(p) = \frac{1}{3K_t'(p)}$ is the diffusion coefficient, $K_a(p)$ is the absorption coefficient, and $\phi_e(p,t)$ is emission [S1, unofficial].

**Parameters:**
*   $D(p)$: Diffusion coefficient, defined as $\frac{1}{3K_t'(p)}$. Controlled via predefined functions or custom transfer functions [S1, unofficial].
*   $K_a(p)$: Absorption coefficient. Controlled indirectly using an automatic value based on $D$ or by using the scattering albedo $\alpha$ (van de Hulst, 1981) [S1, unofficial].
*   $dt$: Time step. Must be set according to the discretization spatial step to ensure convergence of the method [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** Numerical schemes for steady/unsteady RTE (e.g., finite element with main axis projection, finite differences with first/second order angular moment) are often unstable or lead to grid pattern artifacts [S1, unofficial]. The unsteady RTE with diffusion approximation is chosen for stability [S1, unofficial].
*   **Overflow:** Insufficient constraint on diffusion parameters causes light to overflow the structure it was emitted in [S1, unofficial].
*   **Under-diffusion:** Excessive constraint on diffusion parameters limits the diffusion, preventing full identification of structures (e.g., veins) [S1, unofficial].
*   **Positivity:** To ensure radiance is positive, $\phi \ge ||\vec{E}||$ must hold [S1, unofficial]. The implementation should guard against negative fluence values [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize ping-pong buffers with volume data [S1, unofficial]
buffer_a = [[[0.0] * Nz for _ in range(Ny)] for _ in range(Nx)]
buffer_b = [[[0.0] * Nz for _ in range(Ny)] for _ in range(Nx)]

# Parameters: dt must be set according to spatial step for convergence [S1, unofficial]
dt = 0.01  # Exact stability condition not given in sources [G]
D = [[[0.0] * Nz for _ in range(Ny)] for _ in range(Nx)]  # D(p) = 1 / (3 * Kt_prime(p)) [S1, unofficial]
Ka = [[[0.0] * Nz for _ in range(Ny)] for _ in range(Nx)]  # Controlled indirectly via D or albedo alpha [S1, unofficial]

# Iterative Solver
for iteration in range(max_iterations):
    read_buf = buffer_a if iteration % 2 == 0 else buffer_b
    write_buf = buffer_b if iteration % 2 == 0 else buffer_a

    for x in range(1, Nx - 1):
        for y in range(1, Ny - 1):
            for z in range(1, Nz - 1):
                # Central difference Laplacian for phi [G]
                laplacian = (read_buf[x+1][y][z] + read_buf[x-1][y][z] +
                             read_buf[x][y+1][z] + read_buf[x][y-1][z] +
                             read_buf[x][y][z+1] + read_buf[x][y][z-1] -
                             6 * read_buf[x][y][z])

                # Central difference gradients for D and phi [G]
                grad_D_x = (D[x+1][y][z] - D[x-1][y][z]) / 2
                grad_D_y = (D[x][y+1][z] - D[x][y-1][z]) / 2
                grad_D_z = (D[x][y][z+1] - D[x][y][z-1]) / 2

                grad_phi_x = (read_buf[x+1][y][z] - read_buf[x-1][y][z]) / 2
                grad_phi_y = (read_buf[x][y+1][z] - read_buf[x][y-1][z]) / 2
                grad_phi_z = (read_buf[x][y][z+1] - read_buf[x][y][z-1]) / 2

                gradient_term = grad_D_x * grad_phi_x + grad_D_y * grad_phi_y + grad_D_z * grad_phi_z

                phi_old = read_buf[x][y][z]
                phi_e = volume_data.emission[x][y][z]  # Emission term not specified in sources [G]

                new_phi = phi_old + dt * (D[x][y][z] * laplacian - gradient_term - Ka[x][y][z] * phi_old + phi_e)

                # Guard: Ensure positivity if required by physical model [G]
                if new_phi < 0:
                    new_phi = 0

                write_buf[x][y][z] = new_phi

final_diffusion_field = buffer_a if max_iterations % 2 == 0 else buffer_b
```

## Operating characteristics
*   **Time Complexity:** Computation time scales linearly with the number of voxels [S1, unofficial]. For a $128 \times 128 \times 128$ voxel dataset, it requires an additional 2.5ms per iteration [S1, unofficial]. For larger volumes, this hinders interactivity [S1, unofficial].
*   **Memory Complexity:** Requires 256 bits per voxel to store the result of the diffusion, in addition to the original data [S1, unofficial].
*   **Strengths:** Efficient per-voxel kernel implementation using compute shaders; avoids full-volume storage of light paths; provides fluid-like diffusion for exploration without global segmentation [S1, unofficial].
*   **Limitations:** Tuning constraints is difficult; trade-off between overflow and under-diffusion is hard to find; memory and computation costs limit interactivity for large volumes [S1, unofficial].
*   **When to use:** When interactive exploration of light transport in participating media is needed, and global segmentation is undesirable [S1, unofficial].
*   **When not to use:** For high-fidelity, view-dependent rendering requiring exact path tracing, or for very large volumes where memory and linear scaling time exceed interactive thresholds [S1, unofficial].

## Background
The Diffusion Approximation was introduced in Computer Graphics by Stam [Sta95] and later used by [JMLH01] [S1, unofficial]. The method is inspired by fluoroscopy imaging, where fluorescent liquid or gas is injected and tracked to visualize propagation inside organs [S1, unofficial]. The specific iterative RTE solver for light diffusion described here was proposed by D. Murray, J. Baril, and X. Granier [S1, unofficial].

## Sources
[S1] Volume Exploration with Light Diffusion, https://inria.hal.science/hal-03272284v1, fetched 2026-10-26 (unofficial)
