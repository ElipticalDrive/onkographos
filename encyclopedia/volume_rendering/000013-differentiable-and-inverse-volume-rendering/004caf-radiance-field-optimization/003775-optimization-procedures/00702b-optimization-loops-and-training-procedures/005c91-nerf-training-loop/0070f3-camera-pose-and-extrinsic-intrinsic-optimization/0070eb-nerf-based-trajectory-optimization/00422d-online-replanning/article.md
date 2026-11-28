# Online Replanning


## Overview
Online Replanning is a pipeline component that re-plans collision-free trajectories when state estimates deviate significantly from the planned path [S1]. It solves the problem of catastrophic collisions caused by external disturbances or state estimation errors during execution [S1]. Unlike open-loop execution, which follows a pre-computed plan regardless of deviations, online replanning uses feedback to generate new collision-free paths to the goal, ensuring robustness [S1].

## How it works
Inputs: Current state estimate, planned trajectory, goal state [S1].
Outputs: New collision-free trajectory segment [S1].

Data Structures: Not given in the sources [S1].

Steps:
1. Monitor the current state estimate against the planned trajectory [S1].
2. If deviation exceeds a threshold, trigger replanning [S1].
3. Use the current state estimate as the new initial condition [S1].
4. Optimize a new collision-free trajectory to the goal [S1].
5. Execute the re-planned trajectory [S1].

Formulas: Not given in the sources [S1].

Parameters: Threshold for deviation triggering replanning. Not given in the sources [S1].

Edge Cases: Not given in the sources [S1].

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def online_replanning(current_state, planned_trajectory, goal, threshold):
    deviation = compute_deviation(current_state, planned_trajectory)
    if deviation < threshold:
        return planned_trajectory
    new_trajectory = optimize_trajectory(current_state, goal)
    return new_trajectory

def compute_deviation(state, trajectory):
    min_dist = float('inf')
    for waypoint in trajectory:
        dist = sum((p - q)**2 for p, q in zip(state, waypoint))**0.5
        if dist < min_dist:
            min_dist = dist
    return min_dist

def optimize_trajectory(initial_state, goal):
    # Optimize a new collision-free trajectory to the goal using the NeRF-based environment representation.
    # The specific optimization algorithm, cost function, and solver details are not given in the sources.
    pass
```

## Subtopics
Surrogate Objective-based Trajectory Planning is a method for selecting informative camera poses for training radiance fields without requiring model training or ground-truth meshes during inference [S1]. It solves efficient active view planning by using SOARNet to predict surrogate objective scores for candidate poses [S1]. It sets itself apart by decoupling view selection from radiance field optimization, enabling rapid, training-free planning [S1]. When to choose it: When rapid, training-free view planning is required without ground-truth meshes [S1]. [Surrogate Objective-based Trajectory Planning](13000-surrogate-objective-based-trajectory-planning.md)

## Operating characteristics
Time Complexity: Not given in the sources [S1].
Memory Complexity: Not given in the sources [S1].
Measured Performance: Quadrotor flight path execution with feedback avoids catastrophic collisions caused by external disturbances, unlike open-loop execution [S1].
Scaling: Not given in the sources [S1].
Strengths: Avoids catastrophic collisions from external disturbances and state estimation errors [S1].
Limitations: Not given in the sources [S1].
When to Use: Not given in the sources [S1].
When Not to Use: Not given in the sources [S1].

## Background
Origin: Discussed in the context of robot navigation using Neural Radiance Fields [S1].
History: Not given in the sources [S1].
Usage: Not given in the sources [S1].

## Sources
[S1] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26
