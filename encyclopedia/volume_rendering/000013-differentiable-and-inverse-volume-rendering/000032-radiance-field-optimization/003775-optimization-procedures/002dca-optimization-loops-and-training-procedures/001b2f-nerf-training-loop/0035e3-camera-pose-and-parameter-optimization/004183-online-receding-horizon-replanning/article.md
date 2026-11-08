# Online Receding Horizon Replanning

Also called: Online Replanning Navigation Pipeline

## Overview
Online Receding Horizon Replanning is a vision-based robot navigation pipeline that combines a trajectory planner and a pose filter in a receding horizon loop to provide dynamic maneuvering through an environment using only an RGB camera for feedback [S1]. It solves the problem of navigating a robot through a pre-trained Neural Radiance Field (NeRF) representation when the robot's initial pose is uncertain and subject to disturbances [S1]. Unlike static waypoint planners, this method allows the robot to change homotopy classes around obstacles in response to disturbances by continuously replanning the trajectory based on updated state beliefs [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial state prior $(\mu_0, \Sigma_0)$, goal state $x_{goal}$, trained NeRF model $N$ of the environment [S1].
*   **Outputs:** A sequence of control actions $u$ executed by the robot [S1].

**Data Structures:**
*   **State Belief:** Represented by mean $\mu$ and covariance $\Sigma$ [S1].
*   **Waypoints:** A sequence $W$ of states [S1].
*   **NeRF:** A neural network $f_\theta$ representing the scene geometry and appearance, used for collision checking and image synthesis [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize waypoints $W$ using A* search between the initial state $\mu_0$ and the goal state $x_{goal}$ [S1].
2.  **Loop:** While the robot is not at $x_{goal}$:
    a.  **Trajectory Optimization:** Optimize the trajectory $W$ using `trajOpt` to minimize a NeRF-based collision metric [S1]. This produces a refined sequence of waypoints [S1].
    b.  **Action Extraction:** Get the states $x$ and actions $u$ from the optimized waypoints $W$ [S1].
    c.  **Execution:** The robot applies the first control action $u[0]$ [S1].
    d.  **Perception:** The robot receives a noisy image $I$ from its onboard RGB camera [S1].
    e.  **Pose Filtering:** The state estimator uses the captured image $I$ to generate a posterior belief over the new state. This updates the state estimate using `poseFilter` with inputs $I$, previous state $(\mu_{t-1}, \Sigma_{t-1})$, and the first action $u[0]$ [S1].
    f.  **Replanning Preparation:** Update $W$ by replacing the first waypoint with the new mean $\mu_t$ and keeping the rest of the previous waypoints ($W[2:end]$) as a hot start [S1].

**Formulas:**
The collision metric evaluated during `trajOpt` relies on the NeRF's ability to synthesize images and estimate depth or occupancy, though the specific mathematical formulation of the collision metric is not detailed in the provided text [S1]. The pose filter uses the NeRF as a nonlinear measurement model to update the belief $(\mu_t, \Sigma_t)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def online_receding_horizon_planning(initial_prior_mu, initial_prior_sigma, goal_state, nerf_model):
    """
    Implements the Online Receding Horizon Replanning pipeline.
    
    Args:
        initial_prior_mu: Initial state mean (mu_0)
        initial_prior_sigma: Initial state covariance (Sigma_0)
        goal_state: Desired final state (x_goal)
        nerf_model: Trained NeRF model of the environment
        
    Returns:
        None (Executes actions until goal is reached)
    """
    # Step 1: Initialize waypoints using A*
    waypoints = a_star_search(initial_prior_mu, goal_state)
    
    current_mu = initial_prior_mu
    current_sigma = initial_prior_sigma
    
    # Step 2: Receding Horizon Loop
    while not is_at_goal(current_mu, goal_state):
        # a) Optimize trajectory
        # trajOpt minimizes a NeRF-based collision metric
        waypoints = traj_optimization(waypoints, nerf_model)
        
        # b) Get states and actions
        states, actions = get_states_and_actions(waypoints)
        
        # c) Execute first action
        first_action = actions[0]
        execute_action(first_action)
        
        # d) Get camera image
        image = get_camera_image()
        
        # e) Update pose belief
        # poseFilter uses NeRF as a nonlinear measurement model
        current_mu, current_sigma = pose_filter(
            image, 
            current_mu, 
            current_sigma, 
            first_action
        )
        
        # f) Update waypoints for next iteration
        # Replace first waypoint with new mean, keep rest
        waypoints = [current_mu] + waypoints[2:]
        
    return
```

**Edge Cases and Pitfalls:**
*   **Model Mismatch:** The method assumes a trained NeRF model. Performance may degrade if there is a mismatch between the real-world scene and the trained NeRF [S1].
*   **Disturbances:** The system is designed to handle disturbances, but severe disturbances may require significant replanning effort [S1].
*   **Homotopy Changes:** While the method allows changing homotopy classes, this is computationally more expensive than static planning [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Demonstrated in simulation with quadrotor and ground robots navigating through pre-trained NeRF models of a playground, a church, and Stonehenge [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Allows the robot to change homotopy classes around obstacles in response to disturbances, which is difficult for existing differential flatness-based planners that use static waypoints [S1].
*   **Limitations:** Requires a pre-trained NeRF model of the environment [S1].
*   **When to use:** When navigating a robot through a known environment represented by a NeRF using only an RGB camera, especially in the presence of disturbances.
*   **When not:** When real-time mapping is required (NeRF is pre-trained) or when other sensors are available.

## Background
*   **Origin:** The method was introduced in "Vision-Only Robot Navigation in a Neural Radiance World" [S1].
*   **History and Development:** The paper builds on related work in neural implicit representations and vision-only navigation systems [S1].
*   **Usage:** Demonstrated in high-fidelity simulation environments [S1].

## Sources
[S1] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26
