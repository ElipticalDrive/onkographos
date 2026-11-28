# Turing Reaction-Diffusion Noise Generation

Also called: Reaction Diffusion Scheme

## Overview
This method generates initial noise textures with smooth, large 'spots' using Turing's reaction-diffusion equations, replacing standard white noise for better visual representation of tensor fields [S1, unofficial]. It solves the problem that high-frequency noise textures, such as white noise, are not suitable for achieving a proper visual representation of data in certain visualization approaches [S1, unofficial]. It sets itself apart from its sibling [2D White Noise Texture Generation](2485-2d-white-noise-texture-generation.md) by producing correlated, spatially structured patterns rather than uncorrelated random values [G].

## How it works
**Inputs and Outputs:**
- Input: Grid dimensions, diffusion constants $D_a$ and $D_b$, scalar $s$, and initial random values for grids $a$ and $b$ [S1, unofficial].
- Output: Two discrete grids $a$ and $b$ representing the reaction-diffusion state [S1, unofficial].

**Data Structures:**
- Two 2D arrays (grids) $a$ and $b$ of identical dimensions, indexed by $i$ and $j$ [S1, unofficial].
- Values are floating-point numbers [G].

**Parameters:**
- $D_a$: Diffusion constant for grid $a$. Value: $0.125$ [S1, unofficial].
- $D_b$: Diffusion constant for grid $b$. Value: $0.031$ [S1, unofficial].
- $s$: Scalar controlling the size of the spots. Smaller $s$ leads to larger spots [S1, unofficial].
- Boundary Conditions: Continuous boundary conditions are assumed for grid coordinates $i$ and $j$ to obtain a seamless texture [S1, unofficial].

**Steps:**
1. Initialize grids $a$ and $b$ with random values [G].
2. For each cell $(i, j)$, compute the reaction-diffusion deltas:
   $$ \Delta a_{i,j} = s(16 - a_{i,j} \cdot b_{i,j}) + D_a \cdot (a_{i+1,j} + a_{i-1,j} + a_{i,j+1} + a_{i,j-1} - 4 \cdot a_{i,j}) $$
   $$ \Delta b_{i,j} = s(a_{i,j} \cdot b_{i,j} - b_{i,j} - \beta_{i,j}) + D_b \cdot (b_{i+1,j} + b_{i-1,j} + b_{i,j+1} + b_{i,j-1} - 4 \cdot b_{i,j}) $$
   where $\beta_{i,j}$ is an input term [S1, unofficial].
3. Update the grids using the computed deltas:
   $$ a_{i,j} \leftarrow a_{i,j} + \Delta a_{i,j} $$
   $$ b_{i,j} \leftarrow b_{i,j} + \Delta b_{i,j} $$
4. Repeat steps 2–3 for a fixed number of iterations until pattern stability is reached [G].
5. Apply continuous boundary conditions (e.g., coordinate wrapping) to ensure seamlessness across grid edges [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_reaction_diffusion_noise(width, height, s, num_iterations, beta_grid=None):
    # Initialize grids with random values [G]
    a = [[random_float() for _ in range(width)] for _ in range(height)]
    b = [[random_float() for _ in range(width)] for _ in range(height)]
    
    Da = 0.125
    Db = 0.031
    
    for _ in range(num_iterations):
        new_a = [[0.0 for _ in range(width)] for _ in range(height)]
        new_b = [[0.0 for _ in range(width)] for _ in range(height)]
        
        for i in range(height):
            for j in range(width):
                # Continuous boundary conditions via wrapping
                i_prev = (i - 1) % height
                i_next = (i + 1) % height
                j_prev = (j - 1) % width
                j_next = (j + 1) % width
                
                # Laplacian diffusion terms
                lap_a = a[i_next][j] + a[i_prev][j] + a[i][j_next] + a[i][j_prev] - 4.0 * a[i][j]
                lap_b = b[i_next][j] + b[i_prev][j] + b[i][j_next] + b[i][j_prev] - 4.0 * b[i][j]
                
                # Reaction terms
                # beta_ij is an input term from the source [S1, unofficial]
                beta = beta_grid[i][j] if beta_grid is not None else 0.0
                
                F = s * (16.0 - a[i][j] * b[i][j])
                G = s * (a[i][j] * b[i][j] - b[i][j] - beta)
                
                # Compute deltas and update
                new_a[i][j] = a[i][j] + F + Da * lap_a
                new_b[i][j] = b[i][j] + G + Db * lap_b
        
        a = new_a
        b = new_b
        
    return a, b

def random_float():
    """Returns a uniform random float in [0, 1) [G]."""
    import random
    return random.uniform(0.0, 1.0)
```

**Edge Cases and Pitfalls:**
- Tiling the reaction diffusion texture can cause minor artifacts at boundaries, but these are ignored in practice due to the advection step [S1, unofficial].
- The definition of $\beta_{i,j}$ is sparse in the text; it is treated as an input term [S1, unofficial].

## Subtopics

### [Gray-Scott Reaction-Diffusion](19114-gray-scott-reaction-diffusion.md)
The Gray-Scott model is a 3D reaction-diffusion system that simulates the interaction of two chemical species, $U$ and $V$, to generate organic, self-organizing patterns such as spots, stripes, and coral-like structures.

## Operating characteristics
- **Time Complexity:** The method is computationally expensive, requiring multiple iterations per grid cell [S1, unofficial].
- **Memory Complexity:** Requires storage for two grids of the same size as the output texture [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** The source mentions tiling a 100×100 pixel texture to create larger textures because the method is computationally expensive [S1, unofficial].
- **Strengths:** Produces smooth, large 'spots' that are randomly and almost uniquely distributed, providing a better visual representation for tensor fields than white noise [S1, unofficial].
- **Limitations:** Computationally expensive [S1, unofficial]. Potential boundary artifacts when tiling, though mitigated by subsequent advection steps [S1, unofficial].
- **When to use:** When visual representation of tensor fields requires smooth, correlated noise patterns instead of high-frequency white noise [S1, unofficial].

## Background
The method is based on Turing's reaction-diffusion scheme first introduced by Turing [Tur52] to simulate the mixture of two reacting chemicals [S1, unofficial]. It is used in the context of Image-Space Tensor Field Visualization Using a LIC-like Method [S1, unofficial].

## Sources
[S1] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
