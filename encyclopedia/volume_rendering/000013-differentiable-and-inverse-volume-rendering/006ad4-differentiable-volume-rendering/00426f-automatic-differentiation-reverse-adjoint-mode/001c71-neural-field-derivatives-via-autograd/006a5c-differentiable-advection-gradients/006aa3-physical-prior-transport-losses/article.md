# Physical Prior / Transport Losses

Also called: Density transport loss, Differentiable Transport Operator Loss, LA(s), Navier-Stokes based transport, Per-frame transport loss, Self-advection loss

## Overview
Physical Prior / Transport Losses enforce temporal coherence and physical plausibility in fluid reconstruction by minimizing the discrepancy between predicted and observed states using a differentiable transport operator [S1]. Unlike standard temporal smoothing, this method explicitly incorporates the Navier-Stokes equations as a physical prior, ensuring that the reconstructed velocity and density fields adhere to incompressible fluid dynamics [S1]. It builds on [Differentiable Advection Gradients](20606-differentiable-advection-gradients.md) by providing the gradient signal necessary to update velocity fields from sparse or noisy observations.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of state fields $s$ (density $\rho$ or velocity $u$) at discrete time steps $t-1, t, t+1$; a velocity field $u$; and physical parameters (kinematic viscosity $\nu$).
*   **Outputs:** Gradient updates for the velocity field $u$ and density $\rho$ derived from the transport loss $L_A$ and divergence loss $L_{div}$.

**Data Structures:**
*   **State Field $s$:** A 3D grid or implicit representation storing scalar (density) or vector (velocity) values at each voxel/point.
*   **Velocity Field $u$:** A 3D vector field where each element is $(u_x, u_y, u_z)$.
*   **Transport Operator $A(s, u)$:** A differentiable function that advects field $s$ by velocity $u$ over one time step.

**Parameters:**
*   **Transport Scheme:** Second-order transport scheme is used in the discretized operator $A$ to preserve small-scale content [S1].
*   **Kinematic Viscosity ($\nu$):** A scalar parameter in the Navier-Stokes equations [S1].
*   **Time Step:** Implicitly handled by the operator $A$; the loss connects adjacent frames $t-1$ to $t$ and $t$ to $t+1$ [S1].

**Formulas:**
The core transport loss $L_A(s)$ for a state $s$ at time $t$ is defined as the sum of squared errors between the transported previous state and the current state, and the transported current state and the next state [S1]:

$$
L_A(s) = \| A(s_{t-1}, u_{t-1}) - s_t \|^2 + \| A(s_t, u_t) - s_{t+1} \|^2
$$

where:
*   $s$ is the field (either density $\rho$ or velocity $u$) [S1].
*   $A(s, u)$ is the differentiable transport operator advecting $s$ by $u$ [S1].
*   $u$ is the velocity field [S1].

For velocity fields, this self-advection loss $L_A(u)$ induces temporal coherence [S1]. For density fields, $L_A(\rho)$ is used to build the initial velocity sequence [S1].

The physical prior is grounded in the Navier-Stokes equations for incompressible fluid motion [S1]:

$$
\frac{\partial u}{\partial t} + u \cdot \nabla u = -\nabla p + \nu \nabla^2 u, \quad \text{s.t.} \quad \nabla \cdot u = 0
$$

where $u$ is velocity, $p$ is pressure, and $\nu$ is kinematic viscosity [S1].

To enforce the incompressibility constraint ($\nabla \cdot u = 0$), an additional divergence-free constraint loss is minimized [S1]:

$$
L_{div} = \| \nabla \cdot u \|^2
$$

**Numbered Steps:**
1.  **Initialize Velocity:** Acquire an initial velocity sequence $u_0, \dots, u_F$ via a forward-only pre-optimization pass [S1]. This is required to create the initial density transport $A_t(\rho_0)$ [S1].
2.  **Compute Transported States:** For each frame $t$, compute the transported previous state $A(s_{t-1}, u_{t-1})$ and the transported current state $A(s_t, u_t)$ using the second-order transport scheme [S1].
3.  **Calculate Transport Loss:** Compute $L_A(s)$ for both density $\rho$ and velocity $u$ using the formula above [S1].
4.  **Calculate Divergence Loss:** Compute $L_{div} = \| \nabla \cdot u \|^2$ to enforce incompressibility [S1].
5.  **Back-propagate Gradients:**
    *   Compute gradients for density $\rho$ and velocity $u$ using the transport loss $L_A$ [S1].
    *   Back-propagate gradients through the density advection operator $A$ to update $\rho$ [S1].
    *   Back-propagate gradients through the density advection operator $A$ to update $u$ [S1].
    *   **Crucial Constraint:** Do **not** back-propagate gradients further through velocity self-advection (i.e., do not differentiate through $A(u_t, u_t)$ with respect to $u_t$ for the purpose of updating $u$ from the velocity loss term itself, though $u$ is updated via density transport gradients) [S1]. The velocity receives gradients $\nabla u_t^A = \frac{\partial L_\rho(\rho_i)}{\partial u_t} = \frac{\partial A(\rho_t, u_t)}{\partial u_t} \nabla \rho_{t+1}^A$ from the back-propagation through the density advection [S1].
6.  **Update Fields:** Update the velocity and density fields using the computed gradients [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_transport_loss_and_gradients(s_prev, s_curr, s_next, u_prev, u_curr, u_next, transport_op, divergence_op):
    """
    Computes the physical prior transport loss and gradients for a state s (rho or u).
    Note: This function assumes the caller handles the specific back-propagation constraints
    described in the source, particularly regarding velocity self-advection.
    """
    # Step 1: Advect previous state using previous velocity
    s_prev_transported = transport_op(s_prev, u_prev)
    
    # Step 2: Advect current state using current velocity
    s_curr_transported = transport_op(s_curr, u_curr)
    
    # Step 3: Compute transport loss
    # L_A(s) = ||A(s_{t-1}, u_{t-1}) - s_t||^2 + ||A(s_t, u_t) - s_{t+1}||^2
    loss_term_1 = norm_squared(s_prev_transported - s_curr)
    loss_term_2 = norm_squared(s_curr_transported - s_next)
    transport_loss = loss_term_1 + loss_term_2
    
    # Step 4: Compute gradients for transport loss
    # Note: In a real autograd engine, this is done automatically.
    # Here we conceptually define the components.
    grad_s_prev = transport_op.gradient_wrt_input(s_prev, u_prev) * (s_prev_transported - s_curr)
    grad_u_prev = transport_op.gradient_wrt_velocity(s_prev, u_prev) * (s_prev_transported - s_curr)
    grad_s_curr = transport_op.gradient_wrt_input(s_curr, u_curr) * (s_curr_transported - s_next)
    grad_u_curr = transport_op.gradient_wrt_velocity(s_curr, u_curr) * (s_curr_transported - s_next)
    
    # Combine gradients for the current state s_curr
    # The source specifies that for velocity u, gradients come from density transport back-propagation.
    # For density rho, gradients come from density transport.
    # This function returns the loss and the raw gradient components.
    return transport_loss, grad_s_curr, grad_u_curr

def compute_divergence_loss(u_curr, divergence_op):
    """
    Computes the divergence-free constraint loss.
    """
    div_u = divergence_op(u_curr)
    loss_div = norm_squared(div_u)
    grad_u = divergence_op.gradient_wrt_input(u_curr) * div_u
    return loss_div, grad_u

def optimize_velocity_sequence(initial_u, rho_sequence, transport_op, divergence_op, num_frames):
    """
    Main optimization loop for velocity sequence.
    """
    # Initialize velocity sequence from pre-optimization pass [S1]
    u_sequence = initial_u 
    
    for t in range(1, num_frames - 1):
        # Get states at t-1, t, t+1
        rho_prev = rho_sequence[t-1]
        rho_curr = rho_sequence[t]
        rho_next = rho_sequence[t+1]
        
        u_prev = u_sequence[t-1]
        u_curr = u_sequence[t]
        u_next = u_sequence[t+1]
        
        # Compute transport loss for density rho
        # L_A(rho) is used to build the initial velocity sequence and provides gradients for u
        rho_loss, grad_rho_curr, grad_u_curr_from_rho = compute_transport_loss_and_gradients(
            rho_prev, rho_curr, rho_next, u_prev, u_curr, u_next, transport_op, divergence_op
        )
        
        # Compute divergence loss for velocity u
        # L_div = ||div(u)||^2
        div_loss, grad_u_curr_from_div = compute_divergence_loss(u_curr, divergence_op)
        
        # Total gradient for u_curr
        # Source [S1]: "Those [gradients for u] are, however, not back-propagated further through velocity self-advection."
        # This implies we use the gradients derived from density transport and divergence,
        # but do not chain gradients through u's own advection loss term for u's update.
        total_grad_u_curr = grad_u_curr_from_rho + grad_u_curr_from_div
        
        # Update u_curr using total_grad_u_curr [S1]
        u_sequence[t] = update_step(u_curr, total_grad_u_curr)
        
        # Update rho_curr using grad_rho_curr [S1]
        rho_sequence[t] = update_step(rho_curr, grad_rho_curr)
        
    return u_sequence
```

**Edge Cases and Pitfalls:**
*   **Velocity Self-Advection:** Gradients must not be back-propagated through velocity self-advection. This is a critical implementation detail to avoid incorrect gradient flow [S1].
*   **Initial Velocity:** A forward-only pre-optimization pass is required to acquire the initial velocity sequence before the global transport optimization can begin [S1].
*   **Small-Scale Content:** Using a first-order transport scheme may lose small-scale details; a second-order scheme is recommended [S1].

## Subtopics

### Density Transport Loss
A specific instance of the transport loss where $s = \rho$. It is used to build the initial velocity sequence and provides gradients for velocity updates via back-propagation through the density advection operator [S1].

### Self-Advection Loss
A specific instance of the transport loss where $s = u$. It induces temporal coherence in the velocity fields [S1].

### Divergence-Free Constraint Loss
A loss term $L_{div} = \| \nabla \cdot u \|^2$ used to enforce the incompressibility constraint of the Navier-Stokes equations [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method is applied to sparse-view input sequences for fluid reconstruction [S1]. It helps reconstruct inflow regions and motion within the visual hull [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Enforces physical plausibility (Navier-Stokes) and temporal coherence; helps reconstruct motion in regions with sparse visual data (inflow regions) [S1].
*   **Limitations:** Requires an initial velocity sequence from a pre-optimization pass; relies on a differentiable transport operator [S1].
*   **When to use:** When reconstructing fluid dynamics from sparse or noisy observations and physical consistency is required [S1].
*   **When not to use:** When computational resources are extremely limited and pre-optimization is not feasible, or when the fluid is compressible (the method assumes incompressibility) [S1].

## Background
The method originates from the paper "Global Transport for Fluid Reconstruction with Learned Self-Supervision" by Aleksandra Franz, Barbara Solenthaler, and Nils Thuerey, presented at CVPR 2021 [S1]. It leverages recent progress in gradient-based differentiable rendering to connect physical priors to target observations [S1]. The approach uses a discriminator with RaLSGAN loss for training stability [S1].

## Sources
[S1] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
