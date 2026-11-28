# Robot Pose Reconstruction via Differentiable Rendering

Also called: Joint Angle Estimation, Monocular Robot Pose Optimization

## Overview
This method optimizes a robot's joint angles and camera pose from a single monocular RGB image by minimizing the pixel-wise difference between the input image and a differentially rendered image [S1, unofficial]. It builds on [Differentiable Robot Rendering (Dr. Robot)](15698-differentiable-robot-rendering-dr-robot.md) by leveraging its kinematics-aware deformation and Gaussian Splatting to provide accurate gradients for gradient-based control [S1, unofficial]. Unlike neural network-based refiners such as Robopose, this approach performs direct optimization without a learned prior [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A single monocular RGB image $I$ [S1, unofficial].
*   **Output:** Estimated robot pose vector $\mathbf{p}$ (joint angles) and camera pose parameters $\theta_{\pi}$ [S1, unofficial].

**Data Structures**
*   $\mathbf{p}$: Vector of robot joint angles.
*   $\theta_{\pi}$: Vector of camera pose parameters (extrinsics/intrinsics as defined by the renderer $\pi$).
*   $I$: Input image tensor.
*   $\mathbf{f}(\mathbf{p})$: The robot's geometric representation deformed by pose $\mathbf{p}$, as defined in the parent method [S1, unofficial].
*   $\pi$: The differentiable rasterizer/renderer function [S1, unofficial].

**Algorithm Steps**
1.  **Initialization:**
    *   For the first frame of a video sequence, initialize $\mathbf{p}$ and $\theta_{\pi}$ from scratch (e.g., random or neutral pose) [S1, unofficial].
    *   For subsequent frames, warm start the optimization by setting the initial $\mathbf{p}$ and $\theta_{\pi}$ to the values estimated from the previous frame [S1, unofficial].
2.  **Objective Definition:** Define the reconstruction loss $\mathcal{L}_{\text{Rec}}$ as the L2 norm between the input image $I$ and the rendered image $\pi(\mathbf{f}(\mathbf{p}))$ [S1, unofficial].
3.  **Optimization:** Use gradient descent to minimize the objective function with respect to $\mathbf{p}$ and $\theta_{\pi}$ [S1, unofficial].

**Formula**
The optimization problem is defined as [S1, unofficial]:

$$
\min_{\mathbf{p}, \theta_{\pi}} \mathcal{L}_{\text{Rec}}(\mathbf{p}, I) = \|I - \pi(\mathbf{f}(\mathbf{p}))\|_2
$$

Where:
*   $\mathbf{p}$ is the robot pose vector (joint angles).
*   $\theta_{\pi}$ represents the camera pose parameters.
*   $I$ is the input monocular RGB image.
*   $\pi$ is the differentiable rasterizer/renderer function.
*   $\mathbf{f}(\mathbf{p})$ is the deformed robot geometry.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_robot_pose(I, initial_pose_p, initial_pose_theta_pi, renderer, robot_model, num_iterations=100, lr=0.01):
    """
    Optimizes robot joint angles and camera pose to match input image I.
    
    Args:
        I: Input monocular RGB image tensor.
        initial_pose_p: Initial robot joint angles vector.
        initial_pose_theta_pi: Initial camera pose parameters.
        renderer: Differentiable rasterizer function pi.
        robot_model: Function f that deforms robot geometry based on pose p.
        num_iterations: Number of gradient descent steps.
        lr: Learning rate for gradient descent.
        
    Returns:
        p_opt: Optimized robot joint angles.
        theta_pi_opt: Optimized camera pose parameters.
    """
    # Initialize trainable parameters
    p = torch.tensor(initial_pose_p, requires_grad=True)
    theta_pi = torch.tensor(initial_pose_theta_pi, requires_grad=True)
    
    optimizer = torch.optim.Adam([p, theta_pi], lr=lr)
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        # Get deformed robot geometry
        # f(p) is defined in the parent Dr. Robot method
        deformed_geometry = robot_model(p)
        
        # Render the image using the differentiable renderer
        # pi is the differentiable rasterizer
        rendered_image = renderer(deformed_geometry, theta_pi)
        
        # Calculate L2 loss between input image and rendered image
        loss = torch.norm(I - rendered_image, p=2)
        
        # Backpropagate gradients
        loss.backward()
        
        # Update parameters
        optimizer.step()
        
    return p.detach(), theta_pi.detach()

def process_video_sequence(video_frames):
    """
    Processes a sequence of video frames using warm-start optimization.
    """
    prev_p = None
    prev_theta_pi = None
    
    for i, frame in enumerate(video_frames):
        if i == 0:
            # Initialize from scratch for the first frame
            # [G] Typical initialization might be neutral joint angles and standard camera pose
            init_p = get_neutral_pose() 
            init_theta_pi = get_standard_camera_pose()
        else:
            # Warm start with previous frame's estimates
            init_p = prev_p
            init_theta_pi = prev_theta_pi
            
        p_opt, theta_pi_opt = optimize_robot_pose(
            I=frame,
            initial_pose_p=init_p,
            initial_pose_theta_pi=init_theta_pi,
            renderer=dr_robot_renderer,
            robot_model=dr_robot_model
        )
        
        prev_p = p_opt
        prev_theta_pi = theta_pi_opt
        
    return prev_p, prev_theta_pi
```

**Edge Cases and Pitfalls**
*   **Local Minima:** Optimization from scratch on the first frame may converge to local minima if the initial pose is far from the ground truth. Warm starting subsequent frames mitigates this for video sequences [S1, unofficial].
*   **Camera Pose Ambiguity:** If the camera pose is unknown, it is optimized jointly with the robot pose. This can lead to ambiguity if the robot's appearance is symmetric or lacks distinctive features [S1, unofficial].
*   **Occlusions:** The method assumes the robot is visible. Severe occlusions may degrade reconstruction accuracy [S1, unofficial].

## Subtopics

### Text to Robot Pose with CLIP
Uses CLIP similarity between rendered images and text prompts to optimize robot pose for open-world tasks. Sets itself apart by leveraging vision-language models for action planning rather than just reconstruction. Choose when text-guided robot manipulation is required. See [Text to Robot Pose with CLIP](15699-text-to-robot-pose-with-clip.md).

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization iterations and the cost of differentiable rendering. Typically slower than neural network-based methods like Robopose per frame, but does not require training time [S1, unofficial].
*   **Memory Complexity:** Low, as it does not store large neural network weights for the refiner. Memory usage is dominated by the differentiable renderer and robot model [S1, unofficial].
*   **Measured Performance:** On the Panda-3cam dataset:
    *   Azure camera: 9.33 L1 error (std 2.65) vs Robopose 16.8 (std 3.38) [S1, unofficial].
    *   Kinect360 camera: 12.52 L1 error (std 3.58) vs Robopose 14.37 (std 3.69) [S1, unofficial].
    *   Realsense camera: 17.83 L1 error (std 5.5) vs Robopose 24.57 (std 3.71) [S1, unofficial].
*   **Scaling:** Performance is consistent across different camera platforms with varying fields of view and mounting positions [S1, unofficial].
*   **Strengths:** High accuracy, outperforms SOTA neural refiner (Robopose) by 32.9% in joint angle estimation accuracy (L1 distance) [S1, unofficial]. Can reconstruct challenging parts like gripper poses when visible [S1, unofficial].
*   **Limitations:** Requires gradient descent optimization per frame, which is computationally more expensive than a single forward pass of a neural network. Sensitive to initialization for the first frame [S1, unofficial].
*   **When to use:** When high accuracy is required and computational budget allows for per-frame optimization. When a differentiable renderer is already available.
*   **When not to use:** When real-time performance is critical and a neural network refiner is preferred. When the robot is heavily occluded.

## Background
The method was introduced in the paper "Differentiable Robot Rendering" by [S1, unofficial]. It evaluates the quality of gradients produced by Dr. Robot on the practical task of robot joint angles estimation from a single in-the-wild image. The evaluation uses the Panda-3CAM dataset, which consists of around 10K images with corresponding robot poses [S1, unofficial]. The method is compared against Robopose, a state-of-the-art method that uses a refiner neural network [S1, unofficial].

## Sources
[S1] Differentiable Robot Rendering, https://core.ac.uk/works/391740986, fetched 2026-11-05 (unofficial)
