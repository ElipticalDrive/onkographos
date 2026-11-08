# NeRF-based Trajectory Optimization

Also called: Differential Flatness-based Trajectory Optimization with NeRF, NeRF-based collision penalty optimization, Proposed trajectory optimizer

## Overview
NeRF-based Trajectory Optimization is a planning method for differentially flat systems (e.g., quadrotors) that optimizes a dense set of flat output waypoints directly using first-order gradient descent, using NeRF density for implicit obstacle representation instead of closed-form polytopes [S1]. Unlike traditional pipelines that use quadratic programs with static, hand-designed waypoints and closed-form obstacle descriptions, this method uses a denser set of optimizable waypoints and implicit NeRF obstacles, requiring nonconvex optimization via automatic differentiation [S1]. It differs from [NeRF with Unknown Camera Parameters](7405-nerf-with-unknown-camera-parameters.md) by focusing on robot state optimization rather than camera pose estimation, and from [Online Receding Horizon Replanning](10509-online-receding-horizon-replanning.md) by providing the core optimization loop that the replanning component relies upon.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained NeRF model $\rho(x)$ representing the environment; current robot state estimate (position, orientation, velocity); goal state; robot dynamics model (defining flat outputs and control effort); a finite collection of points $B$ representing the robot body.
*   **Outputs:** A sequence of optimized flat output waypoints $\{\sigma_1, \dots, \sigma_h\}$ defining the trajectory.

**Data Structures:**
*   **Waypoints:** A sequence of flat output vectors. For a quadrotor, each waypoint $\sigma_i$ contains position $(x, y, z)$ and yaw angle $\psi$. The number of waypoints $h$ is determined by the discretization of the time horizon.
*   **Robot Body Points:** A finite collection $B = \{b_1, \dots, b_k\}$ of 3D points representing the robot's bounding box or complex model, used for collision checking [S1].

**Formulas:**
1.  **Collision Probability:** The collision probability at time $t$ is approximated by summing the NeRF density at the robot body points, weighted by the distance traveled in the timestep [S1]:
    $$ p_{coll}^t \le \sum_{b_t \in B} \rho(b_t) s(b_t) $$
    where $\rho(b_t)$ is the NeRF density at point $b_t$, and $s(b_t)$ is the distance traveled by body-fixed point $b$ in timestep $t$ [S1]. This assumes the probability of terminating a light ray is a strong proxy for terminating a mass particle [S1].
2.  **Objective Function:** The total cost $\mathcal{L}$ to minimize is a combination of collision probability and control effort [S1]:
    $$ \mathcal{L} = \mathcal{L}_{coll} + \lambda \mathcal{L}_{ctrl} $$
    where $\mathcal{L}_{coll}$ is the sum of collision probabilities over the trajectory, $\mathcal{L}_{ctrl}$ is the control effort (derived from dynamics and surrounding waypoints), and $\lambda$ is a weighting factor [S1].

**Steps:**
1.  **Initialization:** Generate preliminary waypoint poses between the current and goal pose using a heuristic, such as a straight line or A* on a coarse grid overlaid on the scene [S1].
2.  **Optimization:** Optimize the initial guesses via gradient descent to minimize the combined objective of collision probability and control effort [S1].
    *   Use the Adam optimizer [S1].
    *   Compute gradients using automatic differentiation through the NeRF density queries and the collision probability formula [S1].
    *   Update the flat output waypoints (position and yaw for quadrotors) [S1].
3.  **Convergence:** Iterate until the optimizer converges to a trajectory with minimal collision probability and smooth control effort [S1]. In evaluations, convergence to zero intersections with ground truth meshes was observed by iteration 49 in the Stonehenge environment [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_trajectory(nerf_model, current_pose, goal_pose, robot_body_points, num_iterations=2500):
    # 1. Initialize waypoints using heuristic (e.g., straight line or A*)
    waypoints = initialize_waypoints(current_pose, goal_pose)
    
    # 2. Set up optimizer
    optimizer = Adam(learnable_parameters=waypoints)
    
    for i in range(num_iterations):
        # 3. Compute collision cost
        collision_cost = 0.0
        for t in range(len(waypoints) - 1):
            # Interpolate trajectory between waypoints to get body positions
            body_positions = get_body_positions(waypoints[t], waypoints[t+1], robot_body_points)
            
            # Sum NeRF density at body points weighted by distance traveled
            for b_pos in body_positions:
                density = nerf_model.query(b_pos)
                distance_traveled = get_distance_traveled(b_pos, t)
                collision_cost += density * distance_traveled
        
        # 4. Compute control effort cost (e.g., snap or jerk based on dynamics)
        control_cost = compute_control_effort(waypoints)
        
        # 5. Combine costs
        total_cost = collision_cost + lambda_weight * control_cost
        
        # 6. Backpropagate and update
        total_cost.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return waypoints
```

**Parameters:**
*   **Optimizer:** Adam optimizer [S1].
*   **Iterations:** 2500 for initial trajectory optimization [S1].
*   **Flat Outputs:** For quadrotors, position and yaw angle [S1].

**Edge Cases and Pitfalls:**
*   **Nonconvexity:** The trajectory optimization is fundamentally nonconvex, requiring robust first-order methods like Adam [S1].
*   **NeRF Density Proxy:** The assumption that light ray termination probability proxies mass particle termination may not hold for all materials or densities, though it correlates well with ground truth mesh intersections [S1].
*   **Initialization Sensitivity:** Poor initialization (e.g., straight line through obstacles) may lead to local minima, though the dense waypoint optimization helps mitigate this compared to minimum-snap planners [S1].

## Subtopics

### NeRF-based Collision Probability Estimation
Estimates collision probability by treating NeRF volume density as a proxy for the probability of a mass particle (robot body) stopping a ray. It assumes the probability of terminating a light ray is a strong proxy for terminating a mass particle [S1]. The robot body is represented by a finite collection of points $B$ (typically a 3D grid of the bounding box) [S1]. This cost term is minimized during trajectory optimization [S1].

### Online Replanning
A pipeline component that re-plans collision-free trajectories when state estimates deviate significantly from the planned path. It uses the same optimization loop as the initial planning but with fewer iterations (250) and a shorter time step (0.1s) [S1]. Unlike open-loop execution, online replanning with feedback avoids catastrophic collisions caused by external disturbances [S1].

## Operating characteristics
*   **Time Complexity:** Dependent on the number of iterations and waypoints. Initial trajectory optimization takes ~20s over 2500 iterations on an AMD Ryzen 9 5900X CPU and Nvidia RTX 3090 GPU [S1]. Online replanning updates occur in ~2s over 250 iterations [S1].
*   **Memory Complexity:** Depends on the NeRF model size and the number of waypoints. The NeRF model is loaded into memory, and gradients are computed for each waypoint [S1].
*   **Performance:** In the Stonehenge environment, the optimizer converges to a trajectory with zero intersections with ground truth meshes by iteration 49 [S1]. The NeRF collision loss correlates with the intersecting volume of ground-truth meshes [S1].
*   **Strengths:** Handles complex, implicit obstacles without closed-form descriptions; optimizes dense waypoints for smoother trajectories compared to minimum-snap planners; integrates naturally with differential flatness-based feedback controllers [S1].
*   **Limitations:** Computationally expensive compared to traditional QP-based methods; relies on the accuracy of the NeRF representation; nonconvex optimization may get stuck in local minima if initialization is poor [S1].
*   **When to Use:** When the environment is represented by a NeRF and traditional geometric obstacle representations are unavailable or too complex; for differentially flat systems like quadrotors [S1].
*   **When Not to Use:** When real-time performance is critical and computational resources are limited; when the environment is static and can be represented by simple geometric primitives [S1].

## Background
Originated in the paper "Vision-Only Robot Navigation in a Neural Radiance World" [S1]. The method leverages the property of differential flatness, where inputs and states of systems like quadrotors can be represented using a smaller set of flat outputs (position and yaw) [S1]. It was developed to address the limitations of traditional planning pipelines that require closed-form obstacle descriptions and static waypoints [S1]. The approach was evaluated on quadrotors and omnidirectional robots in environments like Stonehenge, demonstrating its ability to navigate tight spaces and avoid collisions using only vision-based NeRF representations [S1].

## Sources
[S1] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26
