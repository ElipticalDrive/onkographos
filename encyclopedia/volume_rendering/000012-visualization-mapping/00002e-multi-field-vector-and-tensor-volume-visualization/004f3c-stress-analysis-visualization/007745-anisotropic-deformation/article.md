# Anisotropic Deformation
Also called: Layer-based deformation, Normal Vector Deformation, Rubber sheet deformation, Tensor Field Deformation, Tensor Field Deformation via Spring Lattice

## Overview
Anisotropic Deformation is a volume deformation technique that visualizes the directional information of a tensor field by deforming an entire volume using the tensor field as a force field acting on the object's normal vectors [S1]. It is capable of illustrating the compressing and shearing properties of the tensor field [S1]. Unlike [Normal Vector Deformation](52-normal-vector-deformation.md), which emphasizes tensor information in one primary direction and requires a user-specified interrogation vector, Anisotropic Deformation allows the interrogation to vary over space and time, deforming the entire volume to represent tensor information in all directions [S1]. It provides an approximation of the tensor field's effect, specifically visualizing the symmetric portion of the tensor, as rigid rotation does not affect lengths [S1].

## How it works
The method assumes the tensor field is sufficiently smooth so that there are no sudden changes in neighboring tensors, and tensors vary linearly between neighboring tensors [S1]. It also assumes that the tensor data resides in a structured grid such as a rectilinear or curvilinear grid [S1]. The tensor field is considered a special force field with different properties in all directions [S1].

### Inputs and Outputs
*   **Input:** A tensor field $T(x)$ defined on a structured 3D grid.
*   **Output:** A deformed volume where the geometry reflects the local tensor properties (enlarging, compressing, shearing) [S1].

### Data Structures
The volume is represented as a lattice of nodes. In 3D, each node is connected to its immediate neighbors, requiring 26 springs per point [S1].
*   **Node:** Stores position $P(x)$, initial position $P_{init}(x)$, and normal vector $N(x)$ (for Normal Vector Deformation variant).
*   **Spring:** Connects two nodes. Stores rest length $L$ and spring constant $K$.
*   **Tensor Field:** Structured grid of $3 \times 3$ tensors $T(x)$ [S1].

### Parameters
*   **TimeStep:** Integration time step. Must be less than 0.1 for stability. Typical value: 0.01 (range 0.005 to 0.05 for better results) [S1].
*   **$\lambda$:** User-specified parameter controlling the amount of deformation. Range: $[0, 1]$. Typical value: 0.1 to 0.9. If too small, deformation is undetectable. If 1, springs may degenerate to 0 due to singular tensors [S1].
*   **K:** Spring constant. Homogeneous in the Anisotropic Deformation variant [S1].
*   **Interrogation Vector:** In the Normal Vector Deformation variant, this is the normal vector of the current layer. It can be replaced by a user-specified vector [S1].

### Algorithm Steps

#### Variant 1: Normal Vector Deformation (Layer-based)
This variant traverses the tensor volume along one of the 3 axes in computational space, processing the volume one layer or sheet at a time [S1].

1.  **Initialize:** Set the normal vectors $N(x)$ as the normals to the current layer in physical space. Set the rest length of each spring to the distance between the initial positions of the two nodes straddling it [S1].
2.  **Iterate until equilibrium:**
    a.  **Calculate External Force:** For each node $x$, calculate the deforming force $F_e(x)$ by multiplying the tensor $T(x)$ and the normal vector $N(x)$ [S1]:
        $$F_e(x) = T(x) \cdot N(x)$$
    b.  **Calculate Internal Force:** Calculate the internal force $F_i(x)$ based on the deformation function $D(x)$ and the displacement relative to rest length [S1]:
        $$F_i(x) = - \oint \frac{\partial D}{\partial x} \cdot \left( \frac{\left\| \frac{\partial D}{\partial x} \Delta x \right\|}{\left\| \Delta x \right\|} - 1 \right) \cdot \Delta x$$
        where the integral represents the summation over connected springs [S1].
    c.  **Calculate Combined Force:** Sum the external and internal forces [S1]:
        $$F_c(x) = F_e(x) + F_i(x) = T \cdot N + F_i(x)$$
    d.  **Update Position:** Update the position of the deformed layer node $P_{k+1}(x)$ using the combined force and a time step [S1]:
        $$P_{k+1}(x) = P_k(x) + F_c(x) \cdot \text{TimeStep}$$
    e.  **Update Normals:** Update normals according to the new surface orientation as the layer deforms [S1].
3.  **Termination:** Stop when the values in the combined force fields are zero at all points (equilibrium) [S1].

#### Variant 2: Anisotropic Deformation (Spring Lattice)
This variant does not require an interrogation vector or tensor decomposition. It adjusts lengths connecting a node to its neighboring nodes by minimizing resultant displacement force [S1].

1.  **Initialize:** Establish a spring model where each node is connected to its neighbors in 3D. Spring constants are homogeneous [S1]. Set the rest length of each spring to the original distance between neighboring nodes [S1].
2.  **Iterate until equilibrium:**
    a.  **Recalculate Rest Length:** For each pair of neighboring points $x_0$ and $x_1$, recalculate the rest length $L(x_0, x_1)$ using a weighted average of the tensor-transformed distance and the original distance [S1]:
        $$L(x_0, x_1) = \lambda \|T(x_0) \cdot (x_0 - x_1)\| + (1 - \lambda) \|x_0 - x_1\|$$
    b.  **Calculate Internal Force:** Compute internal forces $F(x_0)$ by summing contributions from all connected neighbors $x_1$, using the spring constant $K$ and the deviation of current length from rest length $L$ [S1]:
        $$F(x_0) = \sum_{x_1} K \cdot (x_1 - x_0) \cdot \left( \frac{\|x_0 - x_1\|}{L(x_0, x_1 + x_0)} - 1 \right)$$
    c.  **Update Position:** Use $F(x)$ to compute new positions of all points in each iteration [S1].
3.  **Termination:** Stop when the magnitude of $F(x)$ is below a threshold everywhere (equilibrium) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def anisotropic_deformation(tensor_field, grid_nodes, lambda_param, time_step, max_iterations):
    """
    Implements the Anisotropic Deformation algorithm (Variant 2).
    
    Args:
        tensor_field: 3D array of 3x3 tensors T(x).
        grid_nodes: List of Node objects with position P and neighbors.
        lambda_param: Deformation control parameter [0, 1].
        time_step: Integration time step (< 0.1).
        max_iterations: Maximum number of iterations.
        
    Returns:
        Deformed grid_nodes.
    """
    # Initialize rest lengths based on initial positions
    for node in grid_nodes:
        for neighbor in node.neighbors:
            # Calculate initial distance
            dist = norm(neighbor.P - node.P)
            # Set rest length to initial distance (Variant 2 default)
            # Note: Variant 1 sets rest length to initial distance too, but 
            # Variant 2 recalculates it every iteration based on tensor.
            neighbor.rest_length = dist 
            
    for iteration in range(max_iterations):
        equilibrium_reached = True
        max_force = 0.0
        
        for node in grid_nodes:
            internal_force = Vector3(0, 0, 0)
            
            for neighbor in node.neighbors:
                # Recalculate rest length L(x0, x1)
                # T(x0) is tensor at node
                # v = x0 - x1
                v = node.P - neighbor.P
                tensor_transformed_v = tensor_field[node.index] @ v
                tensor_dist = norm(tensor_transformed_v)
                original_dist = norm(v)
                
                # L(x0, x1) = lambda * ||T(x0) * (x0 - x1)|| + (1 - lambda) * ||x0 - x1||
                new_rest_length = lambda_param * tensor_dist + (1 - lambda_param) * original_dist
                
                # Update rest length for the spring connecting node and neighbor
                neighbor.rest_length = new_rest_length
                
                # Calculate internal force contribution
                # F(x0) = sum K * (x1 - x0) * (||x0 - x1|| / L(x0, x1) - 1)
                # Note: The formula in source uses L(x0, x1 + x0) which implies L depends on direction or is symmetric.
                # Assuming L is symmetric for simplicity in implementation unless specified otherwise.
                # Source formula: F(x0) = sum_{x1} K * (x1 - x0) * (||x0 - x1|| / L(x0, x1 + x0) - 1)
                
                current_dist = norm(neighbor.P - node.P)
                direction = neighbor.P - node.P
                
                # Avoid division by zero
                if new_rest_length < 1e-9:
                    spring_term = 0.0
                else:
                    spring_term = (current_dist / new_rest_length) - 1
                
                # K is homogeneous spring constant
                K = 1.0 # Default spring constant
                force_contribution = K * direction * spring_term
                
                internal_force += force_contribution
            
            # In Variant 2, there is no external force F_e. F_c = F_i.
            combined_force = internal_force
            
            # Update position
            node.P += combined_force * time_step
            
            # Check equilibrium
            force_mag = norm(combined_force)
            if force_mag > max_force:
                max_force = force_mag
            if force_mag > 1e-6: # Threshold for equilibrium
                equilibrium_reached = False
                
        if equilibrium_reached:
            break
            
    return grid_nodes

def normal_vector_deformation(tensor_field, grid_layers, time_step, max_iterations):
    """
    Implements the Normal Vector Deformation algorithm (Variant 1).
    
    Args:
        tensor_field: 3D array of 3x3 tensors T(x).
        grid_layers: List of layers, each containing nodes.
        time_step: Integration time step (< 0.1).
        max_iterations: Maximum number of iterations.
        
    Returns:
        Deformed grid_layers.
    """
    for layer in grid_layers:
        # Initialize normals as normals to the current layer in physical space
        for node in layer.nodes:
            node.N = compute_layer_normal(node, layer)
            
    for layer in grid_layers:
        for iteration in range(max_iterations):
            equilibrium_reached = True
            max_force = 0.0
            
            for node in layer.nodes:
                # Calculate External Force F_e(x) = T(x) * N(x)
                external_force = tensor_field[node.index] @ node.N
                
                # Calculate Internal Force F_i(x)
                # This requires summing over connected springs in the layer
                internal_force = Vector3(0, 0, 0)
                for neighbor in node.neighbors_in_layer:
                    # Approximate derivative term or use spring model
                    # Source formula: F_i(x) = - integral ...
                    # Implementation note: The source describes a spring lattice model.
                    # We approximate the integral as a sum over neighbors.
                    # The term inside the integral involves the deformation gradient.
                    # For a spring lattice, internal force is typically derived from spring extensions.
                    # However, the source explicitly gives a formula for F_i involving D(x).
                    # Since D(x) is the position P(x), we approximate the gradient.
                    
                    # Simplified spring force for internal force calculation
                    # Note: The source formula for F_i is complex. 
                    # A common implementation for rubber sheet deformation uses spring forces.
                    # Let's use the standard spring force for internal consistency with the lattice.
                    # F_spring = -K * (current_length - rest_length) * direction
                    # But the source formula is specific.
                    
                    # Re-reading source: "F_i(x) is then defined as follows: [formula]"
                    # The formula involves partial derivatives of D.
                    # In a discrete lattice, we can approximate this.
                    # However, to keep it implementable from text, we use the spring model logic
                    # described for the lattice.
                    
                    # Let's stick to the spring model for internal forces as described in the context
                    # of "physics-based model to constrain the movement".
                    # The formula provided might be a continuous approximation.
                    # For implementation, we use the discrete spring force.
                    
                    # Actually, the source says: "To restore this balance, the system will have to displace...
                    # until the internal and external forces reach a new equilibrium state."
                    # And "F_c(x) = F_e(x) + F_i(x)".
                    # And "F_i(x) = - integral ...".
                    
                    # Given the complexity of implementing the exact integral formula from text,
                    # and the fact that it's a spring lattice, we approximate F_i as the sum of spring forces.
                    # This is a gap fill [G].
                    
                    dist = norm(neighbor.P - node.P)
                    direction = neighbor.P - node.P
                    rest_len = neighbor.rest_length # Set to initial distance
                    
                    if rest_len < 1e-9:
                        spring_force = Vector3(0,0,0)
                    else:
                        # Standard spring force
                        spring_force = -1.0 * (dist - rest_len) / rest_len * direction 
                        # Note: Source formula has a more complex term. 
                        # The term (||dD/dx dx|| / ||dx|| - 1) is essentially (current_length / rest_length - 1).
                        # So F_i is proportional to (current_length / rest_length - 1) * direction.
                        # The negative sign indicates restoring force.
                        
                    internal_force += spring_force
                
                # Combined Force
                combined_force = external_force + internal_force
                
                # Update Position
                node.P += combined_force * time_step
                
                # Update Normals
                node.N = compute_layer_normal(node, layer)
                
                # Check equilibrium
                force_mag = norm(combined_force)
                if force_mag > max_force:
                    max_force = force_mag
                if force_mag > 1e-6:
                    equilibrium_reached = False
                    
            if equilibrium_reached:
                break
                
    return grid_layers

def compute_layer_normal(node, layer):
    """
    Computes the normal vector for a node in a layer.
    [G] Approximation using cross product of two edge vectors from neighbors.
    """
    if len(layer.nodes) < 3:
        return Vector3(0, 0, 1) # Default normal
    
    # Get two neighbors to define the plane
    n1 = layer.nodes[0].P - node.P
    n2 = layer.nodes[1].P - node.P
    
    normal = cross(n1, n2)
    if norm(normal) < 1e-9:
        return Vector3(0, 0, 1)
    
    return normalize(normal)
```

### Edge Cases and Pitfalls
*   **Instability:** Heterogeneous tensors in small areas can cause wild fluctuations in normal vectors. This is solved by using a physics-based spring model to constrain movement and reach equilibrium [S1].
*   **Visual Clutter:** Displaying multiple deformed slices can cause clutter. This is solved by adding weak spring links between layers to make deformation regular [S1].
*   **Smooth Deformation Hiding Non-Smooth Fields:** A smooth-looking deformation may hide a non-smooth tensor field, leading to high error rates. Users should specify smaller volumes to inspect details if error is high [S1].
*   **Approximation:** The method provides an approximation, not an exact solution. It only visualizes the symmetric portion of the tensor because rigid rotation (from polar decomposition) does not affect lengths [S1].
*   **Singular Tensors:** If $\lambda$ is set to 1, springs may degenerate to 0 due to singular tensors [S1].

## Operating characteristics
*   **Time Complexity:** The method is iterative and stops when equilibrium is reached. The number of iterations depends on the smoothness of the tensor field and the time step.
*   **Memory Complexity:** Requires storing the tensor field, node positions, and spring connections. In 3D, each node requires 26 springs [S1].
*   **Measured Performance:** On the single point load dataset, the maximal error detection factor was 0.0311 (3.11%) [S1]. On the delta wing dataset, the maximal error rate was 7.61% [S1]. On the cylinder with hemispherical cap dataset, the maximal error rate was 6.35% [S1]. The error factor is below 10% for most simulations [S1].
*   **Strengths:** Good at exhibiting enlarging, compressing, and shearing properties in all directions. It provides an approximation that preserves anisotropy information in high-error regions [S1].
*   **Limitations:** It is an approximation, not an exact solution. It is less accurate than normal vector deformation in some metrics [S1]. The resulting deformation differs when different layer orientations are used to sweep the volume (for Normal Vector Deformation) [S1].

## Background
Anisotropic Deformation was introduced in the paper "Volume Deformation For Tensor Visualization" [S1]. It is an improvement over previous methods like Boring and Pang's Normal Vector Deformation, which used a homogeneous interrogation vector [S1]. The method is applied to visualize stress due to deformation in linear elasticity isogeometric analysis, where external forces lead to a displacement field [Parent]. It has been applied to the single point load tensor dataset, the delta wing velocity gradient tensor dataset, and the cylinder with hemispherical cap rate of strain tensor dataset [S1].

## Sources
[S1] Volume Deformation For Tensor Visualization, https://avis.soe.ucsc.edu/images.tensordef/vis02.tensordef.pdf, fetched 2026-10-28
