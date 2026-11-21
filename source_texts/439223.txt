# Monte Carlo Radiative Transport with Isotropic Scattering

## Overview

This project implements a simple Monte Carlo simulation of particle transport in a homogeneous scattering medium. Packets are emitted from the origin and undergo isotropic scattering as they propagate through the medium. The simulation tracks the spatial distribution of packets after a large number of scattering events and generates a histogram of their final radial positions.

The code is written in C++ and serves as a basic demonstration of Monte Carlo methods used in radiation transport, astrophysics, neutrino transport, and diffusion problems.

---

## Physical Model

The simulation assumes:

* Homogeneous medium with constant density
* Pure isotropic scattering
* No absorption (`κ = 0`)
* Constant scattering opacity (`σ = 1`)
* Packets initially emitted from the origin
* Random free paths drawn from an exponential optical-depth distribution

The optical depth is sampled as

```math
\tau = -\ln(\xi)
```

where `ξ` is a uniform random number in `(0,1)`.

The free path is

```math
\lambda = \frac{\tau}{(\kappa + \sigma)\rho}
```

where:

* `κ` = absorption opacity
* `σ` = scattering opacity
* `ρ` = density

Scattering directions are sampled isotropically via

```math
\mu = \cos\theta = 2\xi_1 - 1
```

and

```math
\phi = 2\pi\xi_2.
```

The displacement vector is

```math
\Delta x = \lambda \sin\theta \cos\phi
```

```math
\Delta y = \lambda \sin\theta \sin\phi
```

```math
\Delta z = \lambda \cos\theta
```

---

## Features

* Monte Carlo random walk simulation
* Isotropic scattering
* Exponential free-path sampling
* Tracking of packet positions
* Radial distance calculation
* Histogram generation of final packet distribution
* Plain-text output for post-processing and visualization

---

## Code Structure

Each packet is represented by

```cpp
struct quanta{
    double x_coord;
    double y_coord;
    double z_coord;
    double energy;
    double time;
    double radius;
};
```

The simulation tracks:

* Position (`x`, `y`, `z`)
* Energy
* Time
* Radius from the origin

---

## Simulation Parameters

Default values:

```cpp
int n_packt = 10000;
int t_step  = 10000;

sigma = 1.0;
kmu   = 0.0;
rho   = 1.0;
```

| Parameter | Description                   |
| --------- | ----------------------------- |
| `n_packt` | Number of Monte Carlo packets |
| `t_step`  | Number of scattering steps    |
| `sigma`   | Scattering opacity            |
| `kmu`     | Absorption opacity            |
| `rho`     | Medium density                |

---

## Output

The simulation generates

```text
radius_histo.txt
```

containing

```text
radius_bin_center    counts
```

Example:

```text
12.5    31
13.2    44
13.9    57
...
```

---

## Compilation

Compile using GCC:

```bash
g++ -O3 -std=c++17 main.cpp -o transport
```

Run:

```bash
./transport
```

---

## Example Visualization

```python
import numpy as np
import matplotlib.pyplot as plt

r, counts = np.loadtxt("radius_histo.txt", unpack=True)

plt.plot(r, counts)
plt.xlabel("Radius")
plt.ylabel("Counts")
plt.title("Final Packet Radius Distribution")
plt.show()
```

---

## Expected Behaviour

Each packet performs a three-dimensional random walk through the medium.

For a large number of scattering events `N`,

```math
\langle r^2 \rangle \propto N \lambda^2
```

which leads to diffusive behavior. As the number of scatterings increases, the radial distribution broadens and approaches the solution of the diffusion equation.

---


## License

Feel free to use, modify, and extend the code for research and educational purposes.
