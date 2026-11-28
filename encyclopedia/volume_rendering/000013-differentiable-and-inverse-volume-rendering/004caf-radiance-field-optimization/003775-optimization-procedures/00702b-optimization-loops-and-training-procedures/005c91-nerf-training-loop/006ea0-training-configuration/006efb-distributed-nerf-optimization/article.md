# Distributed NeRF Optimization

Also called: Auxiliary Variable Method in Di-NeRF, Centralized ADMM for Di-NeRF, Consensus ADMM for NeRF, Di-NeRF, Di-NeRF Distributed Optimization

## Overview
Distributed NeRF Optimization (Di-NeRF) is a collaborative learning framework that enables multiple robots to jointly reconstruct a global Neural Radiance Field (NeRF) and refine their relative camera poses without sharing raw sensor data [S1]. It distinguishes itself from centralized training by using Consensus Alternating Direction Method of Multipliers (C-ADMM) to enforce agreement on global network parameters across a network of agents, while simultaneously optimizing local pose and depth distortion parameters [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** For each robot $i \in V$, a set of images $I_i$, initial camera poses $\pi_i$, and an off-the-shelf monocular depth network (e.g., DPT) [S1].
*   **Outputs:** Global NeRF parameters $\theta^*$, relative camera pose transformations $T_{ig}^*$ for each robot, and mono-depth distortion parameters $\psi^{i*}$ [S1].

**Data Structures**
*   $\theta_i$: Local copy of the global NeRF network parameters for robot $i$.
*   $T_{ig}$: Relative camera pose transformation matrix from robot $i$ to the global reference coordinate system $R_g$. Represented as an element of $SE(3)$, decomposed into rotation $\phi = \alpha\omega$ (axis-angle, $\omega \in \mathbb{R}^3$ normalized, $\alpha \in \mathbb{R}$) and translation $t_g \in \mathbb{R}^3$ [S1].
*   $\psi^i$: Set of mono-depth distortion parameters for robot $i$, specifically scale $\alpha_i^m$ and shift $\beta_i^m$ for each image $m$ [S1].
*   $y_i$: Dual variable (Lagrangian multiplier) for robot $i$.
*   $z_{ij}$: Auxiliary variable (complicating variable) representing the consensus value between neighbors $i$ and $j$ [S1].
*   $\Gamma_i$: Combined local variable set $\{\theta_i, T_{ig}, \psi^i\}$ [S1].
*   $N_i$: Set of neighboring robots that robot $i$ communicates with [S1].

**Local Loss Function**
The local loss $L_i$ for robot $i$ combines RGB rendering loss and a depth consistency loss [S1]:
$$ L_i = \sum_{m=0}^{M_i-1} \| I_i^m - \hat{I}_i^m \|^2 + \gamma \sum_{m=0}^{M_i-1} \| D_i^{m*} - \hat{D}_i^m \|^2 $$
where:
*   $I_i^m$ is the input image.
*   $\hat{I}_i^m$ is the NeRF-rendered RGB image.
*   $D_i^m$ is the raw mono-depth map from DPT.
*   $D_i^{m*} = \tilde{\alpha}_i^m D_i^m + \tilde{\beta}_i^m$ is the corrected mono-depth map, where $\tilde{\alpha}_i^m, \tilde{\beta}_i^m$ are the scale and shift parameters optimized during training [S1].
*   $\hat{D}_i^m$ is the NeRF-rendered depth image.
*   $\gamma$ is a weighting factor for the depth loss term [S1].

**Algorithm Steps**
The optimization solves the constrained problem:
$$ \theta^*, T^*, \psi^* = \arg\min_{\theta, T, \psi} \sum_{i \in V} L_i(\theta_i, T_{ig}, \psi^i) \quad \text{s.t.} \quad \theta_i = z_{ij}, \forall j \in N_i $$
using the augmented Lagrangian:
$$ \mathcal{L}_i(\Gamma_i) = L_i(\Gamma_i) + \theta_i^\top y_i + \frac{\rho}{2} \sum_{j \in N_i} \| \theta_i - z_{ij} \|^2 $$
where $\rho$ is the penalty factor [S1].

1.  **Initialization:** Initialize $\theta_i, T_{ig}, \psi^i, y_i, z_{ij}$ for all $i \in V, j \in N_i$. Set iteration counter $k=0$.
2.  **Local Subproblem Update:** Each robot $i$ updates its local variables $\Gamma_i^{(k+1)}$ by minimizing the augmented Lagrangian. This involves gradient descent on $L_i(\Gamma_i)$ plus the ADMM penalty terms [S1]:
    $$ \Gamma_i^{(k+1)} = \arg\min_{\Gamma} \left\{ L_i(\Gamma_i) + \theta_i^\top y_i^{(k)} + \frac{\rho}{2} \sum_{j \in N_i} \| \theta_i - z_{ij}^{(k)} \|^2 \right\} $$
    Specifically, $\theta_i^{(k+1)}$ is updated via gradient descent on the NeRF loss, while $T_{ig}$ and $\psi^i$ are updated jointly with $\theta_i$ [S1].
3.  **Dual Variable Update:** Each robot $i$ updates its dual variable $y_i$ using the central collector's output [S1]:
    $$ y_i^{(k+1)} = y_i^{(k)} + \rho (\theta_i^{(k+1)} - z_{ij}^{(k+1)}) $$
4.  **Auxiliary Variable Update (Central Collector):** A central fusion center computes the auxiliary variable $z_{ij}^{(k+1)}$ by averaging the local parameters and dual variables of the neighbors [S1]:
    $$ z_{ij}^{(k+1)} = \frac{1}{|N_i|} \sum_{i \in N_i} \left( \theta_i^{(k+1)} + \frac{1}{\rho} y_i^{(k)} \right) $$
    This simplifies to the average local parameter plus the scaled average dual variable:
    $$ z_{ij}^{(k+1)} = \bar{\theta}^{(k+1)} + \frac{\bar{y}^{(k)}}{\rho} $$
    where $\bar{\theta}^{(k+1)}$ is the average of $\theta_i^{(k+1)}$ over neighbors and $\bar{y}^{(k)}$ is the average of $y_i^{(k)}$ [S1].
5.  **Iteration:** Repeat steps 2-4 until convergence.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distributed_nerf_optimization(robots, neighbors, images, depth_net, gamma, rho, max_iters):
    """
    robots: List of Robot objects. Each has attributes:
        - id: int
        - images: List of image tensors
        - init_pose: SE(3) matrix
        - theta: NeRF parameters (tensor)
        - T_ig: Relative pose (SE(3))
        - psi: Depth distortion params (scale, shift)
        - y: Dual variable (tensor, same shape as theta)
    neighbors: Dict mapping robot_id -> List of neighbor robot_ids
    images: Dict mapping robot_id -> List of image tensors
    depth_net: Function to compute raw mono-depth from image
    gamma: Weight for depth loss
    rho: Penalty factor
    max_iters: Maximum number of ADMM iterations
    """
    
    # Initialization
    for robot in robots:
        robot.theta = initialize_nerf_params()
        robot.T_ig = robot.init_pose
        robot.psi = initialize_depth_distortion()
        robot.y = zeros_like(robot.theta)
        
    # Initialize auxiliary variables z_ij for all edges
    z = {}
    for robot in robots:
        for neighbor_id in neighbors[robot.id]:
            z[(robot.id, neighbor_id)] = zeros_like(robot.theta)

    for k in range(max_iters):
        # Step 1: Local Subproblem Update (Parallel on each robot)
        local_updates = {}
        for robot in robots:
            # Compute raw mono-depth
            raw_depths = [depth_net(img) for img in images[robot.id]]
            
            # Compute corrected mono-depth using current psi
            corrected_depths = [psi.scale * d + psi.shift for d, psi in zip(raw_depths, robot.psi)]
            
            # Define local augmented loss
            def local_loss(theta, T_ig, psi):
                # Render RGB and Depth from NeRF defined by theta, T_ig
                rendered_rgb, rendered_depth = nerf_render(theta, T_ig, images[robot.id])
                
                # RGB Loss
                rgb_loss = sum((img - r_rgb)**2 for img, r_rgb in zip(images[robot.id], rendered_rgb))
                
                # Depth Loss
                depth_loss = sum((c_d - r_d)**2 for c_d, r_d in zip(corrected_depths, rendered_depth))
                
                base_loss = rgb_loss + gamma * depth_loss
                
                # ADMM Penalty Terms
                penalty = 0
                for neighbor_id in neighbors[robot.id]:
                    z_val = z[(robot.id, neighbor_id)]
                    penalty += rho * 0.5 * sum((theta - z_val)**2)
                    
                # Dual Term
                dual_term = sum(theta * robot.y)
                
                return base_loss + dual_term + penalty

            # Minimize local_loss w.r.t theta, T_ig, psi
            # Note: In practice, this involves gradient descent steps
            # Here we assume a generic optimizer step
            new_theta, new_T_ig, new_psi = minimize(local_loss, robot.theta, robot.T_ig, robot.psi)
            
            local_updates[robot.id] = {
                'theta': new_theta,
                'T_ig': new_T_ig,
                'psi': new_psi
            }

        # Update local variables
        for robot in robots:
            robot.theta = local_updates[robot.id]['theta']
            robot.T_ig = local_updates[robot.id]['T_ig']
            robot.psi = local_updates[robot.id]['psi']

        # Step 2 & 3: Dual and Auxiliary Updates (Centralized)
        # Calculate averages over neighbors for each robot
        # Note: The source describes a central collector computing z_ij based on neighbors.
        # We iterate over each robot to update its duals and the z variables associated with it.
        
        for robot in robots:
            # Update Dual Variable y_i
            # y_i^{(k+1)} = y_i^{(k)} + rho * (theta_i^{(k+1)} - z_ij^{(k+1)})
            # However, z_ij is updated centrally. The source says y update uses z_ij^{(k+1)}.
            # We must compute z_ij first or simultaneously.
            
            # Step 3: Auxiliary Variable Update (Central)
            # z_ij^{(k+1)} = mean(theta_i^{(k+1)} + y_i^{(k)}/rho) for i in N_j?
            # Source Eq 16: z_{ij}^{(k+1)} = (1/N) sum_{i in N_i} (theta_i^{(k+1)} + y_i^{(k)}/rho)
            # This implies z is computed for the edge (i,j) based on the neighbors of i?
            # Actually, Eq 16 sums over i in N_i. This suggests z_{ij} is determined by the 
            # neighborhood of i. But z_{ij} is shared between i and j.
            # Standard C-ADMM: z_{ij} is the average of theta_i and theta_j?
            # Source Eq 16: z_{ij} = mean_{i in N_i} (theta_i + y_i/rho).
            # This formula calculates a single value for the group N_i? 
            # Let's follow the source strictly: z_{ij} is computed by the central collector.
            
            # To implement Eq 16:
            # For each edge (i, j), we need z_{ij}.
            # The source says "z_{ij}^{(k+1)} = 1/N sum_{i in N_i} ...". 
            # This notation is slightly ambiguous. Usually, z_{ij} is associated with the constraint between i and j.
            # If the graph is undirected, N_i is the set of neighbors of i.
            # The central collector computes z_{ij} for all edges.
            # Let's assume the standard consensus: z_{ij} is the average of the variables of the nodes connected by the edge?
            # No, Eq 16 sums over N_i. This implies z_{ij} might be specific to the "center" i?
            # Let's look at Eq 17: z_{ij} = theta_bar + y_bar/rho.
            # theta_bar is average of theta_i for i in N_i.
            # This suggests z_{ij} is a consensus value for the neighborhood of i.
            # But z_{ij} is used in the constraint theta_i = z_{ij}.
            # If z_{ij} depends only on N_i, then for neighbor j, the constraint is theta_j = z_{ji}.
            # Are z_{ij} and z_{ji} the same? In symmetric consensus, yes.
            # If z_{ij} = mean(N_i), then z_{ji} = mean(N_j). These are different if degrees differ.
            # However, Eq 18 shows theta_bar = mean(theta_i).
            # Let's stick to the explicit formula in Eq 16/17 for the central collector.
            
            # We need to compute z for all edges.
            # The source implies a central entity computes z_{ij}.
            # Let's compute z_{ij} for each robot i and its neighbors j.
            
            # Compute local averages for robot i's neighborhood
            theta_vals = [local_updates[nid]['theta'] for nid in neighbors[robot.id]]
            y_vals = [robots_by_id[nid].y for nid in neighbors[robot.id]]
            
            avg_theta = mean(theta_vals)
            avg_y = mean(y_vals)
            
            # Compute z_{ij} for all j in N_i
            # Note: The source uses z_{ij} in the dual update for i.
            # So we compute z_{ij} based on N_i.
            for neighbor_id in neighbors[robot.id]:
                z[(robot.id, neighbor_id)] = avg_theta + avg_y / rho

        # Now update dual variables using the newly computed z
        for robot in robots:
            for neighbor_id in neighbors[robot.id]:
                z_val = z[(robot.id, neighbor_id)]
                robot.y = robot.y + rho * (robot.theta - z_val)

    return robots
```

**Edge Cases and Pitfalls**
*   **Pose Optimization:** Camera rotation is in $SO(3)$ space, requiring axis-angle representation to avoid singularities [S1].
*   **Depth Consistency:** Mono-depth maps from off-the-shelf networks (like DPT) are scale-ambiguous. The optimization jointly learns scale and shift parameters $\psi^i$ to align rendered depth with corrected mono-depth [S1].
*   **Communication:** The method assumes a connected communication graph among robots. Disconnected components will converge to separate local NeRFs [S1].
*   **Convergence:** The penalty factor $\rho$ affects convergence speed. A value not specified in the sources is used [S1].

## Operating characteristics
*   **Time Complexity:** Each iteration requires local gradient descent on the NeRF loss (expensive) and a central averaging step (lightweight). The number of iterations depends on the convergence of ADMM, which is generally slower per iteration than centralized SGD but allows parallel local computation [S1].
*   **Memory Complexity:** Each robot stores its own copy of the NeRF parameters $\theta_i$, poses $T_{ig}$, and depth parameters $\psi^i$. No raw images are stored or transmitted, only gradients or parameters (depending on implementation, here parameters are exchanged via the central collector) [S1].
*   **Strengths:** Enables collaborative mapping in challenging environments where broad communication is not feasible (only local neighbor communication required for the graph structure, though a central collector is used for the auxiliary variable update, implying a star topology or a central server) [S1]. Preserves data privacy as raw data is not shared [S1].
*   **Limitations:** Requires a central fusion center for the auxiliary variable update, which can be a bottleneck. Convergence depends on the choice of $\rho$ and the connectivity of the robot network [S1].

## Background
Di-NeRF was introduced in a 2024 paper published in *IEEE Robotics and Automation Letters* by Asadi et al. [S1]. It addresses the challenge of multi-robot simultaneous localization and mapping (SLAM) and collaborative scene reconstruction, where robots must learn a global representation without sharing raw sensor data, which is often impractical due to bandwidth constraints or privacy concerns [S1]. The method builds on the Alternating Direction Method of Multipliers (ADMM) and specifically uses Consensus ADMM (C-ADMM) to handle the distributed nature of the optimization problem [S1].

## Sources
[S1] Di-NeRF: Distributed NeRF for Collaborative Learning With Relative Pose Refinement, https://doi.org/10.1109/lra.2024.3474551, fetched 2026-10-26
