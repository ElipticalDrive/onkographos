# Voxel-Conditioned Continuous Normalizing Flow (VC-CNF)
Also called: Neural ODE Latent Flow, VC-CNF

## Overview
VC-CNF transforms a base latent distribution into a target distribution conditioned on a voxel grid by integrating a time-dependent neural Ordinary Differential Equation (ODE) from $t=0$ to $t=1$ [S1, unofficial]. It builds on [ODENetwork (Conditioned Neural ODE)](5241-odenetwork-conditioned-neural-ode.md) by explicitly framing the latent code generation as a continuous flow process, allowing for smooth interpolation and inversion of the latent space relative to the voxel conditions.

## How it works
**Inputs:**
- Voxel grid data (implicitly accessed via indices or embeddings).
- Base latent vector $z_0$ at $t=0$, sampled from a prior distribution (e.g., Gaussian).
- Integration time interval $[0, 1]$.

**Outputs:**
- Final conditioned latent vector $z_1$ at $t=1$.

**Data Structures:**
- **VoxelEncoder**: Maps discrete voxel indices to continuous embeddings.
- **ODE Network**: An MLP that computes $\frac{dz}{dt}$.
- **Latent Vector**: A tensor of shape `(latent_dim,)`.

**Parameters:**
- `latent_dim`: Dimension of the latent vector. Default: 512 [S1, unofficial].
- `hidden_dim`: Hidden layer size of the ODE network. Default: 512 [S1, unofficial].
- `num_hidden_layers`: Number of hidden layers in the ODE network. Default: 4 [S1, unofficial].
- `skip_layers`: Indices of layers receiving skip connections with voxel embedding and time. Default: `[1, 3]` [S1, unofficial].
- `voxel_embedding_dim`: Dimension of the voxel condition embedding. Default: 128 [S1, unofficial].

**Steps:**
1. **Encode Voxel Grid**: Compute the voxel embedding $e_{voxel}$ using a `VoxelEncoder` (or `GridEmbedding` as implemented in the source) based on the voxel indices [S1, unofficial].
2. **Initialize Latent State**: Set the initial latent state $z(0) = z_0$, where $z_0$ is a random vector of size `latent_dim` [S1, unofficial].
3. **Define ODE Function**: Define the derivative function $f(z, t, e_{voxel})$ which computes $\frac{dz}{dt}$. This function is implemented by the `ODENetwork` [S1, unofficial].
4. **Integrate**: Solve the ODE $\frac{dz}{dt} = f(z, t, e_{voxel})$ from $t=0$ to $t=1$ using the Runge-Kutta 4th order (RK4) method [S1, unofficial].
5. **Return Result**: The final state $z(1)$ is the conditioned latent code [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
from torchdiffeq import odeint

class VC_CNF:
    def __init__(self, latent_dim=512, hidden_dim=512, num_hidden_layers=4, 
                 skip_layers=[1, 3], voxel_embedding_dim=128):
        self.latent_dim = latent_dim
        self.ode_network = ODENetwork(
            latent_dim=latent_dim,
            hidden_dim=hidden_dim,
            num_hidden_layers=num_hidden_layers,
            skip_layers=skip_layers,
            voxel_embedding_dim=voxel_embedding_dim
        )
        self.integrated_time = torch.tensor([0.0, 1.0])

    def ode_func(self, t, latent):
        """
        Computes the derivative dz/dt.
        t: scalar time tensor
        latent: latent vector tensor
        """
        # The ODENetwork is assumed to have access to voxel_embedding
        # via closure or passed context. In the source, it's called as:
        # self.ode_network(voxel_embedding, t, latent)
        # However, odeint expects f(t, y). We assume voxel_embedding is bound.
        return self.ode_network(voxel_embedding=self.voxel_embedding, t=t, latent=latent)

    def set_voxel_embedding(self, voxel_embedding):
        self.voxel_embedding = voxel_embedding

    def forward(self, z0):
        """
        z0: initial latent vector of shape (latent_dim,)
        """
        # Define the ODE function with bound voxel_embedding
        def ode_func(t, latent):
            return self.ode_network(voxel_embedding=self.voxel_embedding, t=t, latent=latent)
        
        # Integrate from t=0 to t=1
        # odeint returns tensor of shape (len(time), batch_size, latent_dim)
        # We take the last time step
        z1 = odeint(ode_func, z0, self.integrated_time, method="rk4")[-1]
        return z1

class ODENetwork(nn.Module):
    def __init__(self, latent_dim, hidden_dim, num_hidden_layers, skip_layers, voxel_embedding_dim):
        super().__init__()
        self.latent_dim = latent_dim
        self.hidden_dim = hidden_dim
        self.num_hidden_layers = num_hidden_layers
        self.skip_layers = skip_layers
        self.voxel_embedding_dim = voxel_embedding_dim
        
        # Build layers
        layers = []
        input_dim = latent_dim + 1 + voxel_embedding_dim # latent + time + voxel_embed
        for i in range(num_hidden_layers):
            if i in skip_layers:
                # Skip connection adds voxel_embedding and time to the input of this layer
                # The output of previous layer is concatenated with voxel_embed and time
                pass
            # Standard MLP layer logic would go here
            # For simplicity, assuming a standard MLP structure with skip connections
            # implemented as concatenation before the linear layer
            layers.append(nn.Linear(input_dim, hidden_dim))
            layers.append(nn.ReLU())
            input_dim = hidden_dim # Next layer takes hidden_dim unless skip adds more
            
        # Note: The exact architecture of ODENetwork is not fully detailed in [S1, unofficial]
        # but it is described as an MLP conditioned on voxel embedding and time.
        # The skip_layers indicate where voxel_embedding and time are concatenated.
        self.network = nn.Sequential(*layers)

    def forward(self, voxel_embedding, t, latent):
        # Concatenate latent, time, and voxel_embedding
        # t is a scalar, needs to be expanded to match batch/latent dim if necessary
        # Assuming latent is (latent_dim,), t is scalar, voxel_embedding is (voxel_embedding_dim,)
        t_expanded = t.expand(self.latent_dim)
        x = torch.cat([latent, t_expanded, voxel_embedding], dim=-1)
        return self.network(x)
```

**Edge Cases and Pitfalls:**
- **Precision**: RK4 integration requires careful handling of time steps if the ODE is stiff. The source uses fixed time points `[0.0, 1.0]` which implies the solver handles the interval internally [S1, unofficial].
- **Boundaries**: The voxel embedding must be valid for the given indices. Invalid indices should be handled by the `GridEmbedding` layer [S1, unofficial].
- **Degenerate Input**: If `latent_dim` or `hidden_dim` is zero, the network will fail. Default values are provided [S1, unofficial].

## Subtopics

### [Conditional Normalizing Flow (Sylvester Flow)](18452-conditional-normalizing-flow-sylvester-flow.md)
An invertible transformation function used to map samples from a latent prior distribution to radiance and density values, conditioned on 5D location-direction pairs.

## Operating characteristics
- **Time Complexity**: Depends on the number of RK4 steps and the cost of the ODE network forward pass. The source does not specify the number of steps, only the method [S1, unofficial].
- **Memory Complexity**: Proportional to `latent_dim` and `hidden_dim` for storing the latent state and network weights. The source does not specify memory usage details [S1, unofficial].
- **Strengths**: Allows for smooth interpolation in the latent space conditioned on voxel data. The continuous nature of the ODE provides a differentiable path from prior to posterior [S1, unofficial].
- **Limitations**: Requires solving an ODE, which is computationally more expensive than a direct MLP mapping. The source does not provide performance benchmarks [S1, unofficial].
- **When to Use**: When a smooth, invertible mapping from a prior to a conditioned latent space is required, and voxel conditions are available [S1, unofficial].
- **When Not to Use**: When computational efficiency is critical and a direct mapping is sufficient.

## Background
VC-CNF is implemented in the `VC-SDFGen` repository by TurtleJayWang [S1, unofficial]. It is part of a pipeline for generating Signed Distance Functions (SDFs) conditioned on voxel grids. The method leverages the ODENetwork architecture to compute the latent code evolution over time [S1, unofficial].

## Sources
[S1] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
