# Random Pose Selection

Also called: Random Pose Generation

## Overview
Random Pose Selection generates intermediate camera poses by interpolating between two randomly chosen training poses to ensure rendering occurs within observed regions, avoiding failures on unobserved parts of the scene [S1]. It differs from [Nearest Camera Pose Selection](12726-nearest-camera-pose-selection.md), which explicitly ranks existing training data by geometric proximity to a query pose, by instead synthesizing new poses that lie between existing training views to guarantee visibility [S1].

## How it works
**Inputs:**
- $P = \{ \{R_i, t_i\} \}_{i=1}^{N}$: The set of $N$ training poses, where $R_i \in \mathbb{R}^{3 \times 3}$ is the rotation matrix and $t_i \in \mathbb{R}^3$ is the translation vector.
- $\theta$: Rotation distance threshold.
- $d$: Translation distance threshold.

**Outputs:**
- $p_{out} \in \mathbb{R}^{4 \times 4}$: The output homogeneous transformation matrix.
- Intrinsic matrix and image shape: Randomly selected from the training set corresponding to the selected poses.

**Data Structures:**
- Pose $p_i$: Represented as a pair $(R_i, t_i)$.
- Distance functions $D_R$ and $D_t$ compute the separation between two poses.

**Parameters:**
- $\theta = \frac{\pi}{24}$: Rotation threshold [S1].
- $d = 0.5$: Translation threshold [S1].
- $s \sim U[0, 1]$: Interpolation factor, uniformly sampled [S1].

**Formulas:**
1.  **Rotation Distance:**
    $$D_R(R_1, R_2) = \arccos\left(\frac{\text{Tr}(R_2^{-1} R_1)}{2} - 1\right)$$
    where $\text{Tr}(\cdot)$ is the trace of the matrix [S1].

2.  **Translation Distance:**
    $$D_t(t_1, t_2) = \|t_2 - t_1\|_2^2$$
    where $\|\cdot\|_2$ is the Euclidean norm [S1].

3.  **Axis-Angle Conversion:**
    - $\text{AA2R}(v, \theta)$: Converts axis $v$ and angle $\theta$ to a rotation matrix.
    - $\text{R2AA}(R)$: Converts rotation matrix $R$ to axis $v$ and angle $\theta$ [S1].

4.  **Reduced Angle Rotation:**
    $$\text{ReduceAngle}(R, s) = \text{AA2R}\left(\text{R2AA}(R)_v, s \cdot \text{R2AA}(R)_\theta\right)$$
    This scales the rotation angle of $R$ by factor $s$ while keeping the axis constant [S1].

5.  **Intermediate Rotation:**
    $$\text{IntermediateRot}(R_1, R_2, s) = \text{ReduceAngle}(R_2 R_1^{-1}, s) R_1$$
    This computes the relative rotation from $R_1$ to $R_2$, scales it by $s$, and applies it to $R_1$ [S1].

**Algorithm Steps:**
1.  Initialize output pose $p_{out}$ as a $4 \times 4$ identity matrix.
2.  Randomly select two indices $j$ and $k$ from the range $[1, N]$.
3.  While $D_R(R_j, R_k) \ge \theta$ AND $D_t(t_j, t_k) \ge d$:
    - Re-randomly select $j$ and $k$ from $[1, N]$.
4.  Compute the intermediate rotation $R_{jk} = \text{IntermediateRot}(R_j, R_k, s)$.
5.  Set the upper-left $3 \times 3$ block of $p_{out}$ to $R_{jk}$.
6.  Set the translation column of $p_{out}$ to $s t_k + (1 - s) t_j$.
7.  Set the bottom-right element $p_{out}[3, 3]$ to $1$.
8.  Randomly select an intrinsic matrix and image shape from the training set for the output pose [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def random_pose_selection(P, theta, d):
    """
    Generate a random intermediate pose from a set of training poses.
    
    Args:
        P: List of tuples (R, t), where R is 3x3 rotation matrix, t is 3x1 translation vector.
        theta: Rotation distance threshold (default pi/24).
        d: Translation distance threshold (default 0.5).
        
    Returns:
        pout: 4x4 homogeneous transformation matrix.
    """
    N = len(P)
    
    # Helper: Rotation distance
    def rotation_dist(R1, R2):
        # Tr(R2^-1 R1) / 2 - 1
        # Note: R2^-1 is R2.T for rotation matrices
        val = np.trace(R2.T @ R1) / 2.0 - 1.0
        # Clamp for numerical stability in arccos
        val = np.clip(val, -1.0, 1.0)
        return np.arccos(val)
    
    # Helper: Translation distance (squared Euclidean)
    def translation_dist(t1, t2):
        return np.linalg.norm(t2 - t1)**2
    
    # Helper: Axis-Angle to Rotation Matrix
    def aa2r(v, angle):
        # Rodrigues' rotation formula
        # v is assumed to be normalized
        vx, vy, vz = v
        c = np.cos(angle)
        s = np.sin(angle)
        C = 1 - c
        
        rot = np.array([
            [c + vx*vx*C,     vx*vy*C - vz*s,  vx*vz*C + vy*s],
            [vy*vx*C + vz*s,  c + vy*vy*C,     vy*vz*C - vx*s],
            [vz*vx*C - vy*s,  vz*vy*C + vx*s,  c + vz*vz*C]
        ])
        return rot

    # Helper: Rotation Matrix to Axis-Angle
    def r2aa(R):
        # Extract angle from trace
        angle = np.arccos((np.trace(R) - 1) / 2.0)
        angle = np.clip(angle, 0, np.pi)
        
        if angle < 1e-6:
            return np.array([1, 0, 0]), 0.0
            
        # Extract axis
        # R - R^T = 2 * sin(angle) * [v]_x
        axis = np.array([
            R[2, 1] - R[1, 2],
            R[0, 2] - R[2, 0],
            R[1, 0] - R[0, 1]
        ])
        axis = axis / (2 * np.sin(angle))
        return axis, angle

    # Helper: Reduce Angle
    def reduce_angle(R, s):
        v, theta = r2aa(R)
        new_theta = s * theta
        return aa2r(v, new_theta)

    # Helper: Intermediate Rotation
    def intermediate_rot(R1, R2, s):
        # Relative rotation R_rel = R2 * R1^-1
        R_rel = R2 @ R1.T
        # Reduce the relative rotation
        R_red = reduce_angle(R_rel, s)
        # Apply to R1
        return R_red @ R1

    # Step 1: Select random indices
    j = np.random.randint(0, N)
    k = np.random.randint(0, N)
    
    # Step 2: Check distances and re-sample if necessary
    # Note: The source uses AND logic for the while loop condition.
    # If EITHER distance is too large, we continue? 
    # Source: "While DR(Rj, Rk) >= theta AND Dt(tj, tk) >= d"
    # This implies we only stop if BOTH are within thresholds? 
    # Or does it mean "While (DR >= theta) AND (Dt >= d)"?
    # Let's look at the text: "choose a pair of poses that are close enough."
    # Usually, you want BOTH to be close. If the condition is DR>=theta AND Dt>=d,
    # the loop continues ONLY if BOTH are far. If one is close, the loop terminates.
    # However, standard logic for "close enough" usually requires BOTH to be close.
    # Let's re-read carefully: "We compute the distance ... to choose a pair of poses that are close enough."
    # If the loop condition is `while DR >= theta AND Dt >= d`, then if DR < theta (close) but Dt > d (far),
    # the condition is False, and the loop stops. This would accept a pose that is rotationally close but translationally far.
    # This seems like a potential pitfall or specific design choice in [S1].
    # However, looking at Algorithm 1 in [S1]:
    # "while DR (Rj , Rk ) ≥ θ and Dt (tj , tk ) ≥ d do"
    # This is strictly AND. I will implement exactly as written.
    
    while rotation_dist(P[j][0], P[k][0]) >= theta and translation_dist(P[j][1], P[k][1]) >= d:
        j = np.random.randint(0, N)
        k = np.random.randint(0, N)
        
    # Step 3: Compute intermediate rotation
    Rjk = intermediate_rot(P[j][0], P[k][0], np.random.uniform(0, 1))
    
    # Step 4: Construct output pose
    pout = np.eye(4)
    pout[:3, :3] = Rjk
    pout[:3, 3] = np.random.uniform(0, 1) * P[k][1] + (1 - np.random.uniform(0, 1)) * P[j][1]
    # Note: The formula is s*tk + (1-s)tj. The s for rotation and translation might be different instances of random s?
    # Source says: "s: A random value from [0, 1]" (singular).
    # And "pout [:3, 3] = stk + (1 − s)tj".
    # It implies a single s is used for both.
    # Let's fix s for the whole pose generation.
    
    # Re-evaluating with single s:
    s = np.random.uniform(0, 1)
    Rjk = intermediate_rot(P[j][0], P[k][0], s)
    pout = np.eye(4)
    pout[:3, :3] = Rjk
    pout[:3, 3] = s * P[k][1] + (1 - s) * P[j][1]
    pout[3, 3] = 1
    
    return pout
```

**Edge Cases and Pitfalls:**
- **Unobserved Regions:** If the selected poses are too far apart, the interpolated pose may lie in an unobserved region, causing rendering failures [S1]. The distance thresholds mitigate this.
- **Threshold Logic:** The loop condition uses logical AND. If one distance metric is within threshold but the other is not, the loop terminates, potentially accepting a pose that is only partially "close" [S1].
- **Intrinsic Parameters:** The method requires randomly selecting intrinsic matrices and image shapes from the training set to accompany the generated pose [S1].

## Subtopics

### [Nearest Camera Pose Selection](12726-nearest-camera-pose-selection.md)
Nearest Camera Pose Selection identifies the most relevant training images for a novel view by finding camera poses with minimal positional and angular distance. It selects a subset of training poses that maximizes image content overlap with the novel view, serving as a prerequisite for feature reprojection in enhancement pipelines. Unlike Random Pose Selection, which interpolates between random poses to ensure visibility, this method explicitly ranks existing training data by geometric proximity to the query pose.

### Random Pose Perturbation
Random Pose Perturbation generates random pose perturbations from the SE(3) group biased around a reference pose to ensure feature visibility and uniform coverage of the local pose space [S2]. It samples rotation axes uniformly on a unit-2 sphere in the neighborhood of a reference axis $\tilde{e}$ as $\tilde{e} + \tilde{\rho}\tilde{e}$ [S2]. It samples rotation angles from a non-uniform distribution in $[0, \theta]$ for small $\theta$ to yield uniform random rotation matrices [S2]. These rotation matrices are converted to Gibbs vectors (q) for orientation representation [S2]. Translation components are sampled from a bounded uniform random distribution along each axis, covering transverse and forward motions [S2]. If tracking of at least four features is unsuccessful, the method re-initializes with conservative pose samples [S2].

## Operating characteristics
**Time Complexity:**
- The expected number of iterations to find a valid pair depends on the density of training poses and the thresholds $\theta$ and $d$. In the worst case, it is unbounded, but typically converges quickly if the training set is dense enough [S1].

**Memory Complexity:**
- $O(1)$ auxiliary space beyond the storage of the training poses $P$ [S1].

**Strengths:**
- Ensures rendered frames are within observed regions, preventing failures on unobserved parts [S1].
- Simple to implement and computationally inexpensive per sample [S1].

**Limitations:**
- Relies on the density and distribution of the training set; sparse training sets may require large thresholds, reducing the quality of interpolation [S1].
- The AND condition in the loop may accept poses that are close in one metric but far in another [S1].

**When to Use:**
- When generating novel views for training or rendering where visibility is guaranteed by interpolation between known views [S1].

**When Not to Use:**
- When precise alignment with specific training views is required (use [Nearest Camera Pose Selection](12726-nearest-camera-pose-selection.md) instead) [S1].
- When local Jacobian approximation around a specific reference pose is needed (use [Random Pose Perturbation](#random-pose-perturbation) instead) [S2].

## Background
Random Pose Selection was proposed in the context of PeRFception, a method for perception using radiance fields [S1]. It addresses the issue of rendering failures when attempting to render extremely unobserved parts of a scene that are significantly out of train-view coverage [S1]. The method is part of the data generation pipeline for training neural radiance field models [S1].

## Sources
[S1] PeRFception: Perception using Radiance Fields, https://doi.org/10.48550/arxiv.2208.11537, fetched 2026-10-26
[S2] Differentiable Rendering for Pose Estimation in Proximity Operations, https://core.ac.uk/download/552738213.pdf, fetched 2026-10-30
