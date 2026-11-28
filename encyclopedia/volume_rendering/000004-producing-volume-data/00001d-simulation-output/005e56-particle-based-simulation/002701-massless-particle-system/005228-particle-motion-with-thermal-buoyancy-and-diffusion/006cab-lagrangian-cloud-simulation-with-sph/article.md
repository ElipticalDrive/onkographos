# Lagrangian Cloud Simulation with SPH

Also called: SPH-based Lagrangian framework, Thermal-based cloud simulation

## Overview
This method is a physics-based procedural model for the real-time animation and visualization of cumulus clouds at landscape scale [S1]. It solves the problem of efficiently simulating large-scale convective cloud dynamics without the computational cost of filling the transparent sky volume with invisible air, which is required by Eulerian CFD methods [S1]. Unlike sibling methods that may require high particle counts for resolution or offline processing [S1], this approach couples a coarse Lagrangian model of air parcels (thermals) with procedural amplification using volumetric noise, ensuring real-time efficiency and scalability to large domains [S1].

## How it works
The method operates on a two-scale model: large particles called "parcels" act as physics units at the macro-level, while volumetric noise generates the micro-level texture [S1]. The simulation tracks the lifecycle of these parcels from ground emission to decay, using Smoothed-Particle Hydrodynamics (SPH) for inter-particle interactions and aerology physics for environmental interactions [S1].

### Inputs and Outputs
**Inputs:**
*   Atmospheric profile: Temperature and humidity as functions of altitude [S1].
*   Ground conditions: Local wetness and temperature probabilities for thermal emission [S1].
*   Physical constants: Latent heat $L$, specific heat $C_p$, adiabatic index $\gamma$, drag coefficient $C_D$, viscosity $\mu$, stiffness $K$, rest density $\rho_0$ [S1].
*   Kernel support radius $sr$ [S1].

**Outputs:**
*   Parcel states: Position, velocity, mass, temperature, and volume for each thermal [S1].
*   Cloud density field: Derived from parcel states and procedural noise for rendering [S1].

### Data Structures
Each parcel $i$ maintains the following state variables:
*   $\mathbf{x}_i$: Position (3D vector) [S1].
*   $\mathbf{v}_i$: Velocity (3D vector) [S1].
*   $m_i$: Mass (scalar, kg) [S1].
*   $V_i$: Volume (scalar, m³) [S1].
*   $T_i$: Temperature (scalar, Kelvin) [S1].
*   $\omega_i$: Mixing ratio (scalar, g/kg) [S1].
*   $\rho_i$: Density (scalar, kg/m³) [S1].
*   $p_i$: Pressure (scalar, Pa) [S1].

### Numbered Steps

1.  **Initialization**: Initialize thermals (large particles) from the ground based on local wetness and temperature probabilities [S1]. Set initial mass, volume, and atmospheric state based on ground conditions [S1].

2.  **SPH Density Computation**: For each particle $i$, compute density using the neighborhood set within support radius $sr$:
    $$ \rho_i = \sum_j m_j W(\mathbf{x}_i - \mathbf{x}_j, sr) $$
    where $W$ is the weighting kernel [S1].

3.  **SPH Pressure Calculation**: Compute pressure based on the deviation from rest density $\rho_0$ and stiffness constant $K$:
    $$ p_i = K(\rho_i - \rho_0) $$
    [S1].

4.  **Force Calculation**: Compute forces acting on particle $i$:
    *   **Pressure Force**:
        $$ \mathbf{f}_{\text{pressure}_i} = - \sum_j m_i m_j \left( \frac{p_i}{\rho_i} + \frac{p_j}{\rho_j} \right) \nabla W(\mathbf{x}_i - \mathbf{x}_j, sr) $$
        [S1].
    *   **Viscosity Force**:
        $$ \mathbf{f}_{\text{viscosity}_i} = \mu \sum_j m_j \frac{\mathbf{v}_j - \mathbf{v}_i}{\rho_j} \nabla^2 W(\mathbf{x}_i - \mathbf{x}_j, sr) $$
        [S1].
    *   **Buoyancy Force**: Calculate virtual potential temperatures for the parcel and atmosphere. The buoyancy force per unit mass is:
        $$ \mathbf{f}_{buoy} \approx \left( \frac{\Theta_v^{parcel} - \Theta_v^{atm}}{\Theta_v^{atm}} \right) g $$
        where $\Theta_v = \Theta(1 + 0.61\omega)$ [S1].
    *   **Drag Force**: Apply air drag:
        $$ f_{airDrag} = -\frac{1}{2} C_D \rho v^2 A $$
        where $\rho$ is air density, $v$ is relative velocity, and $A$ is reference surface [S1].

5.  **Parcel Dynamics Update**: Integrate forces to update velocity and position. Update parcel volume $V$ based on pressure changes and mass flux using Laplace's law:
    $$ V_2 = V_1 \left( \frac{P_1}{P_2} \right)^{\frac{1}{\gamma}} \left( \frac{m_2}{m_1} \right) $$
    where $\gamma \approx 1.4$ for air [S1].

6.  **Thermodynamic Updates**:
    *   **Pressure Variation**: Update atmospheric pressure with altitude $h$:
        $$ P = P_0 e^{-0.143h} $$
        where $P_0 \approx 1013.25$ hPascal [S1].
    *   **Saturation Check**: Compute saturated mixing ratio $\omega_s$:
        $$ \omega_s = 621.97 \frac{e_s}{P - e_s} $$
        where $e_s$ is saturated vapor pressure depending on temperature $T$ [S1].
    *   **Condensation**: If the parcel reaches the condensation level altitude (where saturation starts, $\omega = \omega_s$), condense vapor. Update temperature due to latent heat release:
        $$ \Delta T = \alpha \frac{L}{C_p} \frac{\Delta m}{m} $$
        where $\alpha = 0.3$, $L = 2260$ kJ/kg, $C_p = 1.006$ kJ/kg.K, and $\Delta m$ is condensed mass [S1].

7.  **Mixing (Entrainment/Detrainment)**:
    *   **Entrainment**: Engulf dry environment air into the parcel, modifying water and air content [S1].
    *   **Detrainment**: Leak cloud mass into the environment [S1].
    *   These processes modify the parcel's mass and temperature, leading to decay [S1].

8.  **Lifecycle Termination**: Parcels rise until they reach equilibrium altitude where buoyancy vanishes. They decay and die due to entrainment and detrainment effects [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
L = 2260.0       # Latent heat kJ/kg
Cp = 1.006       # Specific heat kJ/kg.K
alpha = 0.3      # Empirical constant
P0 = 1013.25     # Reference pressure hPascal
gamma = 1.4      # Adiabatic index
g = 9.81         # Gravity m/s^2
K = 1000.0       # Stiffness constant (example)
rho0 = 1.2       # Rest density kg/m^3
mu = 0.01        # Viscosity coefficient
Cd = 0.47        # Drag coefficient (example)

def get_atmospheric_pressure(h):
    """Calculate atmospheric pressure at altitude h (meters)."""
    # P = P0 * e^(-0.143 * h)
    # Note: Source uses hPascal. Conversion to Pa might be needed for consistency with other units.
    # Assuming h is in meters and formula yields hPascal directly as per source text.
    return P0 * math.exp(-0.143 * h)

def get_saturated_vapor_pressure(T):
    """Calculate saturated vapor pressure es from temperature T (Kelvin)."""
    # Source does not provide explicit formula for es(T), only that it depends on T.
    # Implementer must use standard formula (e.g., Magnus formula) [G].
    return 611.2 * math.exp((17.67 * (T - 273.15)) / (T - 29.65))

def get_saturated_mixing_ratio(P, T):
    """Calculate saturated mixing ratio omega_s."""
    es = get_saturated_vapor_pressure(T)
    # omega_s = 621.97 * es / (P - es)
    # P must be in hPascal to match source formula context
    return 621.97 * es / (P - es)

def get_virtual_potential_temperature(Theta, omega):
    """Calculate virtual potential temperature."""
    return Theta * (1 + 0.61 * omega)

def get_potential_temperature(T, P, P0):
    """Calculate potential temperature Theta."""
    # Theta = T * (P0 / P)^0.286
    return T * math.pow(P0 / P, 0.286)

def compute_kernel_gradient(x_i, x_j, sr):
    """Compute gradient of kernel W(x_i - x_j, sr)."""
    # Source refers to MON05 for kernels. Standard SPH kernel gradient implementation required [G].
    pass

def compute_kernel_laplacian(x_i, x_j, sr):
    """Compute Laplacian of kernel W(x_i - x_j, sr)."""
    # Standard SPH kernel Laplacian implementation required [G].
    pass

class Parcel:
    def __init__(self, x, v, m, T, omega, V):
        self.x = x
        self.v = v
        self.m = m
        self.T = T
        self.omega = omega
        self.V = V
        self.rho = m / V
        self.p = 0.0
        self.f_buoy = [0.0, 0.0, 0.0]
        self.f_pressure = [0.0, 0.0, 0.0]
        self.f_viscosity = [0.0, 0.0, 0.0]
        self.f_drag = [0.0, 0.0, 0.0]

def simulate_step(parcels, dt):
    # 1. SPH Density and Pressure
    for i, p_i in enumerate(parcels):
        rho_sum = 0.0
        for j, p_j in enumerate(parcels):
            if i == j: continue
            # Simplified kernel support check needed in real implementation
            dist = math.sqrt(sum((p_i.x[k] - p_j.x[k])**2 for k in range(3)))
            if dist < sr:
                # W is kernel value, assumed available or computed
                w = compute_kernel_value(p_i.x, p_j.x, sr) 
                rho_sum += p_j.m * w
        p_i.rho = rho_sum
        p_i.p = K * (p_i.rho - rho0)

    # 2. Forces
    for i, p_i in enumerate(parcels):
        # Pressure Force
        f_p = [0.0, 0.0, 0.0]
        for j, p_j in enumerate(parcels):
            if i == j: continue
            dist = math.sqrt(sum((p_i.x[k] - p_j.x[k])**2 for k in range(3)))
            if dist < sr:
                grad_w = compute_kernel_gradient(p_i.x, p_j.x, sr)
                term = p_i.m * p_j.m * (p_i.p / p_i.rho + p_j.p / p_j.rho)
                for k in range(3):
                    f_p[k] -= term * grad_w[k]
        p_i.f_pressure = f_p

        # Viscosity Force
        f_v = [0.0, 0.0, 0.0]
        for j, p_j in enumerate(parcels):
            if i == j: continue
            dist = math.sqrt(sum((p_i.x[k] - p_j.x[k])**2 for k in range(3)))
            if dist < sr:
                lap_w = compute_kernel_laplacian(p_i.x, p_j.x, sr)
                for k in range(3):
                    f_v[k] += mu * p_j.m * (p_j.v[k] - p_i.v[k]) / p_j.rho * lap_w
        p_i.f_viscosity = f_v

        # Buoyancy Force
        h = p_i.x[2] # Assuming z is altitude
        P_atm = get_atmospheric_pressure(h)
        # Assume atmospheric Theta and omega are known from profile
        Theta_atm = get_potential_temperature(T_atm, P_atm, P0)
        omega_atm = get_mixing_ratio_from_profile(h) # Placeholder for atmospheric profile lookup
        Theta_v_atm = get_virtual_potential_temperature(Theta_atm, omega_atm)
        
        Theta_parcel = get_potential_temperature(p_i.T, P_atm, P0)
        Theta_v_parcel = get_virtual_potential_temperature(Theta_parcel, p_i.omega)
        
        buoyancy_mag = ((Theta_v_parcel - Theta_v_atm) / Theta_v_atm) * g
        p_i.f_buoy = [0.0, 0.0, buoyancy_mag]

        # Drag Force
        v_sq = sum(v**2 for v in p_i.v)
        v = math.sqrt(v_sq)
        drag_mag = -0.5 * Cd * p_i.rho * v_sq * 1.0 # A=1 assumed
        if v > 0:
            for k in range(3):
                p_i.f_drag[k] = drag_mag * (p_i.v[k] / v)

    # 3. Update Velocity and Position
    for p_i in parcels:
        total_f = [
            p_i.f_pressure[0] + p_i.f_viscosity[0] + p_i.f_buoy[0] + p_i.f_drag[0],
            p_i.f_pressure[1] + p_i.f_viscosity[1] + p_i.f_buoy[1] + p_i.f_drag[1],
            p_i.f_pressure[2] + p_i.f_viscosity[2] + p_i.f_buoy[2] + p_i.f_drag[2]
        ]
        
        # Acceleration
        a = [total_f[k] / p_i.m for k in range(3)]
        
        # Euler integration
        for k in range(3):
            p_i.v[k] += a[k] * dt
            p_i.x[k] += p_i.v[k] * dt

    # 4. Thermodynamic Updates
    for p_i in parcels:
        h = p_i.x[2]
        P_atm = get_atmospheric_pressure(h)
        
        # Check condensation
        omega_s = get_saturated_mixing_ratio(P_atm, p_i.T)
        if p_i.omega > omega_s:
            delta_m = (p_i.omega - omega_s) * p_i.m / 1000.0 # Convert g to kg
            delta_T = alpha * (L / Cp) * (delta_m / p_i.m)
            p_i.T += delta_T
            p_i.omega = omega_s
            # Update volume based on new mass and pressure
            # V2 = V1 * (P1/P2)^(1/gamma) * (m2/m1)
            # Here P1/P2 is approx 1 for small step, but mass changes
            p_i.V = p_i.V * (p_i.m / (p_i.m - delta_m)) # Simplified volume update for mass loss
            
        # Entrainment/Detrainment logic would modify m and omega here
        # delta_m / delta_z = epsilon - delta
        # This requires altitude step tracking [G]

```

### Parameters
*   Latent heat $L = 2260$ kJ/kg [S1].
*   Specific heat of air $C_p = 1.006$ kJ/kg.K [S1].
*   Empirical constant $\alpha = 0.3$ [S1].
*   Reference pressure $P_0 \approx 1013.25$ hPascal [S1].
*   Adiabatic index $\gamma \approx 1.4$ for air [S1].
*   Drag coefficient $C_D$: Not given in the sources.
*   Viscosity coefficient $\mu$: Not given in the sources.
*   Stiffness constant $K$: Not given in the sources.
*   Rest density $\rho_0$: Not given in the sources.
*   Kernel support radius $sr$: Not given in the sources.

### Edge Cases and Pitfalls
*   **Precision**: The pressure variation formula uses an exponential decay with altitude. Ensure altitude $h$ is in the correct units consistent with the coefficient $0.143$ [S1].
*   **Boundaries**: The method avoids sampling the transparent sky, but parcels must be initialized from the ground. Parcels that reach equilibrium altitude or decay due to entrainment/detrainment must be removed to maintain efficiency [S1].
*   **Degenerate Input**: If $P = e_s$ in the saturated mixing ratio formula, division by zero occurs. This corresponds to 100% relative humidity at saturation, which is the boundary condition for condensation [S1].

## Subtopics

### Adaptive Fluid Sampling (Splitting and Merging)
A sibling method that adapts particle density by splitting and merging particles to maintain resolution. Choose this when variable resolution is needed within the cloud volume itself, rather than the macro-level thermal tracking of Lagrangian clouds. [Adaptive Fluid Sampling (Splitting and Merging)](21185-adaptive-fluid-sampling-splitting-and-merging.md)

## Operating characteristics
*   **Time Complexity**: Real-time efficiency is achieved by using large particles (thermals) for macro-level physics, avoiding the high particle counts required by Eulerian CFD or high-resolution Lagrangian methods [S1].
*   **Memory Complexity**: Low memory usage as it does not store a volumetric grid for the entire sky volume, only the active parcels [S1].
*   **Scaling**: Scales to large domains (landscape size) due to particle-implicit environment interactions, which avoid sampling the transparent environment [S1].
*   **Strengths**: Physically validated, controllable (user can control cloud base and top altitude), real-time performance, captures small-scale details via procedural noise [S1].
*   **Limitations**: Relies on procedural noise for micro-details, which may lack the physical consistency of full fluid simulation for small-scale turbulence [S1].
*   **When to use**: Real-time animation and visualization of cumulus clouds at landscape scale where performance and physical plausibility are required [S1].
*   **When not to use**: When high-resolution, physically accurate small-scale cloud turbulence is required without procedural approximation, or when offline visualization is acceptable and higher particle counts are feasible [S1].

## Background
The method was introduced by Prashant Goswami and Fabrice Neyret in 2017 [S1]. It is based on their earlier poster [GN15] and published in the Eurographics Proceedings [S1]. The work draws on aerology (atmospheric physics of hydrostatic atmosphere with thermodynamics transforms) and Smoothed-Particle Hydrodynamics (SPH) [S1]. It is used for real-time animation and visualization of natural phenomena, specifically cumulus clouds [S1].

## Sources
[S1] Real-time Landscape-size Convective Clouds Simulation and Rendering, https://doi.org/10.2312/vriphys.20171078, fetched 2026-10-26
