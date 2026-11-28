# Cloud Physics: Temperature and Buoyancy

Also called: Buoyancy Force Calculation, Virtual Temperature and Buoyancy

## Overview
This method calculates the vertical acceleration of an air parcel due to density differences with its surrounding environment, which drives cumulus convection. It solves the problem of determining whether a rising parcel of air will continue to ascend or sink back down by comparing its thermodynamic state to the ambient atmosphere. It sets itself apart from sibling methods by focusing specifically on the thermodynamic variables (temperature, pressure, and humidity) required to compute the buoyant force, rather than the fluid dynamics advection or the saturation phase transitions.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $T$: Temperature of the parcel [K or °C].
    *   $P$: Pressure of the parcel [Pa].
    *   $P_0$: Reference pressure [Pa] (typically surface pressure).
    *   $\omega$: Specific humidity of the parcel [kg/kg].
    *   $T_{air}$, $P_{air}$, $\omega_{air}$: Ambient atmospheric temperature, pressure, and specific humidity at the parcel's current altitude.
    *   $g$: Gravitational acceleration [m/s²].
*   **Outputs:**
    *   $f_{bouy}$: Buoyant force per unit mass [m/s²].

**Data Structures:**
The method operates on scalar values associated with a single air parcel or a grid cell representing a parcel. No complex data structures are required beyond the scalar fields for temperature, pressure, and humidity.

**Definitions:**
1.  **Potential Temperature ($\Theta$):** Accounts for pressure variation.
    $$ \Theta = T \left( \frac{P_0}{P} \right)^{0.286} $$
    *   $T$: Temperature.
    *   $P$: Pressure.
    *   $P_0$: Reference pressure.
    *   $0.286$: Exponent derived from the gas constant for dry air and specific heat capacity ($R_d/C_p$) [S1].

2.  **Virtual Temperature ($T_v$):** Accounts for moisture variation. Moist air is less dense than dry air at the same temperature and pressure.
    $$ T_v = T (1 + 0.61\omega) $$
    *   $\omega$: Specific humidity [S1].
    *   $0.61$: Approximation of $(R_v/R_d) - 1$, where $R_v$ and $R_d$ are gas constants for water vapor and dry air, respectively [S1].

3.  **Virtual Potential Temperature ($\Theta_v$):** Combines both pressure and moisture effects.
    $$ \Theta_v = \Theta (1 + 0.61\omega) $$
    *   This is the key variable for buoyancy comparison [S1].

**Algorithm Steps:**
1.  Calculate the parcel's potential temperature $\Theta_{parcel}$ using its current temperature $T$ and pressure $P$.
2.  Calculate the ambient potential temperature $\Theta_{air}$ using the ambient temperature $T_{air}$ and pressure $P_{air}$ at the parcel's location.
3.  Calculate the parcel's specific humidity $\omega_{parcel}$ (if not already provided as a direct input for the buoyancy step, this may come from the humidity/saturation module).
4.  Calculate the ambient specific humidity $\omega_{air}$ at the parcel's location.
5.  Compute the parcel's virtual potential temperature:
    $$ \Theta_{v,parcel} = \Theta_{parcel} (1 + 0.61\omega_{parcel}) $$
6.  Compute the ambient virtual potential temperature:
    $$ \Theta_{v,air} = \Theta_{air} (1 + 0.61\omega_{air}) $$
7.  Compute the buoyant force per unit mass:
    $$ f_{bouy} \approx \frac{\Theta_{v,parcel} - \Theta_{v,air}}{\Theta_{v,air}} g $$
    *   If $f_{bouy} > 0$, the parcel is less dense than the environment and accelerates upward.
    *   If $f_{bouy} < 0$, the parcel is denser and accelerates downward [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_buoyancy(T, P, omega, T_air, P_air, omega_air, g, P0=101325.0):
    """
    Calculate buoyant force per unit mass for a cloud parcel.
    
    Args:
        T: Parcel temperature (K)
        P: Parcel pressure (Pa)
        omega: Parcel specific humidity (kg/kg)
        T_air: Ambient temperature (K)
        P_air: Ambient pressure (Pa)
        omega_air: Ambient specific humidity (kg/kg)
        g: Gravitational acceleration (m/s^2)
        P0: Reference pressure (Pa), default 101325.0
        
    Returns:
        f_bouy: Buoyant force per unit mass (m/s^2)
    """
    # Step 1 & 2: Calculate Potential Temperatures
    # Exponent 0.286 is standard for dry air (R_d / C_p)
    exponent = 0.286
    theta_parcel = T * (P0 / P) ** exponent
    theta_air = T_air * (P0 / P_air) ** exponent
    
    # Step 3 & 4: Use provided specific humidities
    
    # Step 5 & 6: Calculate Virtual Potential Temperatures
    # Factor 0.61 approximates (R_v/R_d - 1)
    moisture_factor = 0.61
    theta_v_parcel = theta_parcel * (1 + moisture_factor * omega)
    theta_v_air = theta_air * (1 + moisture_factor * omega_air)
    
    # Step 7: Calculate Buoyant Force
    # Avoid division by zero if ambient virtual potential temp is zero
    if theta_v_air == 0:
        return 0.0
        
    f_bouy = ((theta_v_parcel - theta_v_air) / theta_v_air) * g
    
    return f_bouy
```

**Parameters and Typical Values:**
*   $P_0$: 101325 Pa (standard sea level pressure) [S1].
*   $g$: 9.81 m/s² [S1].
*   Exponent for potential temperature: 0.286 [S1].
*   Moisture factor: 0.61 [S1].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\Theta_{v,air}$ is zero or near-zero, the buoyancy calculation becomes unstable. This should be guarded against [G].
*   **Precision:** The difference $\Theta_{v,parcel} - \Theta_{v,air}$ can be very small, leading to loss of significance if not handled with sufficient floating-point precision [G].
*   **Saturation:** The formula assumes the specific humidity $\omega$ is known. If the parcel is saturated, $\omega$ is determined by the saturation vapor pressure at temperature $T$. This requires coupling with the humidity/saturation module [S1, S2].
*   **Lapse Rates:** The buoyancy force drives the parcel up or down. As the parcel moves, its temperature changes due to adiabatic expansion/compression. The dry adiabatic lapse rate is approximately -10 °C/km, and the moist adiabatic lapse rate is approximately -5 °C/km [S2]. The simulation must update $T$ and $P$ of the parcel as it moves to recalculate buoyancy correctly [S2].

## Subtopics

### Feedback Control Method for Cumuliform Clouds
A framework for animating clouds surrounding the earth as seen from space, controlled by physics including thermodynamics and Coriolis force [S1]. It allows user specification of desired cloud shape via a pressure map or contour [S1]. The method controls cloud motion using physics based on Navier-Stokes equations, cloud thermodynamics, and the Coriolis force [S1]. Input is a pressure map supplied by the user, specifying regions of high and low atmospheric pressure on the planet’s surface [S1]. The physics is based on parcels rising, cooling, and phase transition from vapor to liquid water together with grid-based equations [S1]. The cloud shape is guided by projecting the user-specified contour in the simulation domain and determining its difference from the current cloud formation [S1]. This difference, in addition to an artificial geometric potential force, is fed back into the system to influence the parameters that control cloud growth [S1].

### Aerology Validation
This refers to the validation of single-parcel kinematics against aerology diagrams [S2]. Air parcels that get saturated as they rise cool at a smaller rate than the dry adiabatic lapse rate due to the heat released by the condensation of water vapor [S2]. In the case of no mixing with the atmosphere, the parcel reaches a maximum altitude and comes to an equilibrium height after a series of damping oscillations [S2]. The maximum altitude lies close to the point where parcel temperature intersects the atmospheric temperature profile [S2]. The parcel continues moving further up till its velocity is reversed by the negative buoyancy force [S2]. The dry adiabatic lapse rate is around -10 °C/km, and the moist adiabatic lapse rate is about -5 °C/km [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per parcel per time step, as it involves a constant number of arithmetic operations.
*   **Memory Complexity:** $O(1)$ per parcel, storing only scalar thermodynamic variables.
*   **Strengths:** Physically accurate representation of convective instability; computationally inexpensive; forms the basis for realistic cloud formation in procedural models.
*   **Limitations:** Requires accurate ambient atmospheric profiles (temperature, pressure, humidity); sensitive to numerical precision in small buoyancy differences; does not account for mixing or turbulence without additional models [S2].
*   **When to use:** When simulating cumulus clouds driven by convection; when physical accuracy of vertical motion is required.
*   **When not to use:** For non-convective clouds (e.g., stratus) where large-scale lifting dominates; when computational resources are extremely limited and physical accuracy is not critical.

## Background
The concepts of potential temperature, virtual temperature, and virtual potential temperature were introduced by atmospheric scientists for convenience in analyzing atmospheric stability [S1]. The buoyancy force formula is a standard result in atmospheric thermodynamics [S1]. The application of these principles to computer graphics cloud animation was pioneered by Dobashi and Yamamoto, who developed a framework for animating clouds surrounding the earth as seen from space, incorporating cloud thermodynamics and the Coriolis force [S1]. Later work by Iwasaki et al. and others expanded on these physics-based approaches for real-time landscape-size convective clouds [S2].

## Sources
[S1] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
[S2] Real-time Landscape-size Convective Clouds Simulation and Rendering, https://doi.org/10.2312/vriphys.20171078, fetched 2026-10-26
