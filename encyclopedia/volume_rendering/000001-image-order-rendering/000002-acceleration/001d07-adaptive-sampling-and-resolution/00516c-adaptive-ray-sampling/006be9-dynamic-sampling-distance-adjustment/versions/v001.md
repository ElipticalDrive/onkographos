# Dynamic Sampling Distance Adjustment

Also called: Adaptive Step Size Control, Time-Budget Sampling, Hardware-Aware Downsampling

## Overview
Dynamic Sampling Distance Adjustment is a family of volume rendering acceleration techniques that modulate the spatial sampling rate (step size) to balance visual fidelity against computational cost. Unlike static methods, these algorithms dynamically adjust the distance between sample points along a ray based on external constraints such as frame rate targets, hardware performance tiers, or geometric alignment requirements. This family is distinct from error-driven adaptive methods (like [Mip-NeRF 360](3393-mip-nerf-360-sampling-and-training.md)) by prioritizing temporal stability or hardware consistency over local pixel error minimization.

## Shared principles
The core mechanism shared by all children is the deviation from a fixed, uniform sampling strategy [G]. Instead of casting a constant number of rays or samples per pixel/ray, the method computes a metric (such as gradient magnitude, rendering error, or temporal difference) and uses it to modulate the sampling rate or resolution [G].

Mathematically, this often involves a control loop:
1. **Metric Evaluation:** Compute a scalar value $M$ representing local complexity or error [G].
2. **Adaptation:** Adjust a parameter $P$ (step size $\Delta t$, resolution $R$, or sampling probability $w$) based on $M$ [G].
3. **Rendering/Training:** Execute the volume integral or loss computation with the adjusted parameter [G].

Common adaptation laws include:
* **Step Size Adaptation:** $\Delta t \propto \frac{1}{M}$ or $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, \frac{k}{M}))$ [G].
* **Resolution Control:** $R_{new} = R_{old} + \alpha \cdot (t_{target} - t_{actual})$ or threshold-based switching [G].
* **Probability Weighting:** $w_i \propto M_i$ for importance sampling [G].

All methods in this family aim to maintain visual fidelity while reducing the total number of volume queries [G].

## Subtopics

### [VTK Frame Rate Targeting & Adaptive Sampling](12375-vtk-frame-rate-targeting-adaptive-sampling.md)
Maintains a target frame rate by adjusting sampling distance per frame based on a global time-budgeting controller. Chosen when strict temporal consistency is required for VR or interactive applications.

### [View Adapted Stepping Distances](13215-view-adapted-stepping-distances.md)
Aligns sample points with voxel cube faces to enable bilinear interpolation and adaptive termination. Chosen to reduce interpolation cost (4 neighbors vs 8) and skip empty space efficiently.

### [GPU-adaptive Volume Downsampling](19919-gpu-adaptive-volume-downsampling.md)
Adjusts sampling multipliers based on a one-time hardware benchmark. Chosen to ensure consistent performance across diverse GPU tiers without manual tuning.

### [Dynamic Subsampling for Preview Rendering](22988-dynamic-subsampling-for-preview-rendering.md)
Combines image-space downsampling with ray-space subsampling for VDI preview. Chosen for interactive remote visualization where camera deviation from the generation viewpoint causes anisotropic performance degradation.

## Background
The concept of dynamic sampling originates from the need to optimize volume rendering performance in real-time systems. Early implementations focused on time-budgeting within visualization toolkits like VTK to handle complex scenes in VR . Subsequent research introduced geometric adaptations, such as view-aligned stepping to simplify interpolation , and hardware-aware strategies to decouple performance from scene complexity . More recent approaches address specific data structures like Volumetric Depth Images (VDIs) to maintain frame rates during navigation .

