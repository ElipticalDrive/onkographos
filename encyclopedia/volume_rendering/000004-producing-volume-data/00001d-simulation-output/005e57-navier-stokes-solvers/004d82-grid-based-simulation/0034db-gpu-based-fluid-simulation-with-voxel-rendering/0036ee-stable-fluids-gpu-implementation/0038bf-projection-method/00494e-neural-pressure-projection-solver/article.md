# Neural Pressure Projection Solver

Also called: Neural Euler Simulator

## Overview
The Neural Pressure Projection Solver is a feed-forward neural network method that predicts velocity updates ($\Delta u$) to enforce incompressibility in fluid simulation, replacing traditional iterative pressure solvers [S1, unofficial]. Unlike the [Projection Method](8336-projection-method.md), which computes pressure via iterative linear system solving, this approach learns the mapping from advected velocity and boundary conditions directly to the divergence-free correction, enabling differentiable simulation [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    1.  Velocity field $u^*$: The velocity field after advection and external force application [S1, unofficial].
    2.  Volume field $V$: A scalar field representing the space distribution of materials (fluid and air) [S1, unofficial].
    3.  Boundary distance field: A scalar field where each grid cell stores the normalized distance between its center and the container boundary [S1, unofficial].
*   **Output**:
    *   Velocity update $\Delta u$: A vector field of the same resolution as $u^*$, used to update the velocity field [S1, unofficial].

**Data Structures**
The inputs are 3D volumetric tensors [G]. Convolutional layers use the notation `C{channels}, K{kernel_size}S{stride}`:
*   `C`: Number of output channels [S1, unofficial].
*   `K`: Kernel size (e.g., `K3` implies a $3 \times 3 \times 3$ kernel) [S1, unofficial].
*   `S`: Stride (e.g., `S1` implies stride 1) [S1, unofficial].

**Algorithm**
The network consists of three parallel mini-networks for feature extraction, followed by a fusion and regression head [S1, unofficial].
1.  **Feature Extraction**: Process each input ($u^*$, $V$, and boundary distance field) through a dedicated mini-network [S1, unofficial]. Each mini-network consists of two identical blocks [S1, unofficial]. Each block contains a 3D convolutional layer (`C32, K3S1`), Batch Normalization, and LeakyReLU activation [S1, unofficial]. Let $E_u$, $E_V$, and $E_B$ be the output embeddings from the velocity, volume, and boundary mini-networks, respectively [G].
2.  **Concatenation**: Concatenate the embeddings along the channel dimension to form a single feature tensor $F$:
    $$ F = \text{Concat}(E_u, E_V, E_B) $$
    The channel dimension of $F$ is $32 + 32 + 32 = 96$ [G].
3.  **Regression Head**: Process $F$ through four convolutional blocks followed by a final convolutional layer [S1, unofficial]. Each of the four blocks applies a 3D convolution (`C32, K3S1`), Batch Normalization, and LeakyReLU activation [S1, unofficial]. The final layer applies only a 3D convolution (`C3, K3S1`) to produce $\Delta u$, omitting Batch Normalization and activation functions [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def conv3d(x, in_ch, out_ch, kernel=3, stride=1, padding=1):
    """Standard 3D convolution operation."""
    return x

def batch_norm(x):
    """Standard batch normalization operation."""
    return x

def leaky_relu(x, slope=0.01):
    """Standard LeakyReLU activation operation."""
    return x

def concat_channels(tensors):
    """Concatenate a list of tensors along the channel dimension."""
    return tensors[0]

def neural_pressure_solver(u_star, volume_field, boundary_field):
    # u_star: (B, 3, D, H, W), volume_field: (B, 1, D, H, W), boundary_field: (B, 1, D, H, W)
    # Mini-network 1: Velocity
    x = conv3d(u_star, 3, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    x = conv3d(x, 32, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    e_u = x

    # Mini-network 2: Volume
    x = conv3d(volume_field, 1, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    x = conv3d(x, 32, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    e_v = x

    # Mini-network 3: Boundary
    x = conv3d(boundary_field, 1, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    x = conv3d(x, 32, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    e_b = x

    # Concatenate along channel dimension
    f = concat_channels([e_u, e_v, e_b])  # 96 channels

    # Regression head: 4 blocks of Conv32
    x = conv3d(f, 96, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    x = conv3d(x, 32, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    x = conv3d(x, 32, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)
    x = conv3d(x, 32, 32, kernel=3, stride=1, padding=1)
    x = batch_norm(x)
    x = leaky_relu(x)

    # Final output layer (Conv only)
    delta_u = conv3d(x, 32, 3, kernel=3, stride=1, padding=1)
    return delta_u
```

**Parameters**
*   **Kernel Size**: $3 \times 3 \times 3$ for all convolutions [S1, unofficial].
*   **Stride**: 1 for all convolutions [S1, unofficial].
*   **Channels**: 32 for all intermediate layers [S1, unofficial].
*   **Activation**: LeakyReLU [S1, unofficial].
*   **Batch Normalization**: Applied after every convolution except the final output layer [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Boundary Conditions**: The boundary distance field is explicitly provided as input to handle container boundaries, avoiding the need for implicit boundary handling within the network weights [S1, unofficial].
*   **Input Resolution**: The network assumes fixed-resolution 3D grids [G].
*   **Differentiability**: The method is fully differentiable, allowing gradients to flow back through the simulation step for inverse problems [S1, unofficial].

### Subtopics
#### Volume-Augmented NeRF
A related method that uses the volume field $V$ to regress density and color for rendering, but does not solve for velocity updates [S1, unofficial]. Builds on [NeRF](52-name.md) [S1, unofficial].

#### Traditional Projection Method
The iterative baseline that solves the Poisson equation for pressure [S1, unofficial]. Builds on [Projection Method](8336-projection-method.md) [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources [S1, unofficial].
*   **Memory Complexity**: Not given in the sources [S1, unofficial].
*   **Performance**: Not given in the sources [S1, unofficial].
*   **Strengths**: Differentiable, avoids iterative convergence issues [S1, unofficial].
*   **Limitations**: Approximate solution [S1, unofficial].
*   **When to use**: Not given in the sources [S1, unofficial].
*   **When not to use**: Not given in the sources [S1, unofficial].

## Background
The Neural Pressure Projection Solver, also referred to as the Neural Euler Simulator, was proposed in the paper "Inferring Fluid Dynamics via Inverse Rendering" [S1, unofficial]. It integrates fluid simulation with inverse rendering, allowing for the inference of fluid dynamics from visual data [S1, unofficial]. The method replaces the traditional pressure projection step with a learned neural network, facilitating end-to-end training for inverse problems [S1, unofficial].

## Sources
[S1] Inferring Fluid Dynamics via Inverse Rendering, https://doi.org/10.48550/arxiv.2304.04446, fetched 2026-10-26 (unofficial)
