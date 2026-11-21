# Scale Decay

## Overview
Scale Decay is a training-time regularization technique for 3D Gaussian Splatting that gradually reduces Gaussian scales to control final primitive sizes and encourage pruning of redundant or occluded Gaussians [S1]. It trades a slight decrease in reconstruction quality for significantly higher rendering FPS and reduced memory footprint [S1].

## How it works
### Inputs and Outputs
- **Input**: The current set of $N$ active Gaussians, each defined by position, rotation, scale $\mathbf{s}$, opacity, and spherical harmonics coefficients [S1].
- **Output**: The updated set of Gaussians with scaled-down $\mathbf{s}$ values after each training iteration [S1].

### Data Structures
Gaussians store scale as a 3D vector $\mathbf{s} = [s_x, s_y, s_z]$ [S1]. The decay is applied directly to these scale components at every training iteration [S1].

### Parameters
- **$\eta_{decay}$**: Multiplicative decay factor applied per iteration [S1].
  - Typical value: $0.999875$ [S1].
  - Tested values: $0.999875, 0.999900, 0.999925, 0.999950$ [S1].
  - Disabled state: "NoDecay" (equivalent to $\eta_{decay} = 1.0$) [S1].

### Algorithm
1. After the standard gradient update for a training iteration, retrieve the current scale vector $\mathbf{s}$ for each active Gaussian [S1].
2. Multiply each scale component by the decay factor: $\mathbf{s} \leftarrow \mathbf{s} \times \eta_{decay}$ [S1].
3. Proceed to the next iteration; the reduced scales encourage Gaussians to fall below visibility thresholds or become occluded, triggering existing pruning heuristics [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_scale_decay(gaussians, eta_decay):
    """Applies multiplicative scale decay to all active Gaussians."""
    for g in gaussians:
        g.scale = g.scale * eta_decay
```

### Edge Cases and Pitfalls
- **Pruning Interaction**: Scale Decay works in conjunction with pruning heuristics; reducing scale encourages the removal of occluded or out-of-view primitives, which is critical for the observed reduction in Gaussian count [S1].
- **Quality-Speed Trade-off**: Stronger decay (lower $\eta_{decay}$) increases FPS and reduces final Gaussian counts and average scales, but slightly degrades reconstruction quality metrics [S1]. The source text claims that "increasing $\eta_{decay}$ consistently improves FPS", but the provided empirical data confirms that lower values yield higher FPS and smaller scales [S1].
- **Log-Scale Storage**: If an implementation stores scales in log-space ($\mathbf{s}_{\text{log}} = \log(\mathbf{s})$) to ensure positivity, the decay must be applied as an additive shift: $\mathbf{s}_{\text{log}}^{\text{new}} = \mathbf{s}_{\text{log}}^{\text{old}} + \log(\eta_{decay})$ [G].

## Operating characteristics
- **Time Complexity**: $O(N)$ per iteration for the decay operation, where $N$ is the number of active Gaussians [S1].
- **Memory Complexity**: Reduces final model memory footprint by decreasing the total Gaussian count through induced pruning [S1].
- **Performance Metrics** (15K iterations, $\eta_{decay}=0.999875$ vs disabled):
  - **FPS**: Increases from 157 to 248 [S1].
  - **Final Gaussian Count**: Decreases from 1.73M to 1.52M [S1].
  - **Final Average Scale**: Decreases from 0.0211 to 0.016 [S1].
  - **Quality**: PSNR decreases from 26.00 to 26.47, and LPIPS increases from 0.231 to 0.236 [S1]. (Note: Table 16 in the source reports PSNR as 26.60 for NoDecay, indicating minor reporting variance across tables [S1].)
- **Strengths**: Highly effective lever for balancing quality and speed; encourages pruning of redundant Gaussians [S1].
- **Limitations**: Slight degradation in visual quality metrics; requires tuning $\eta_{decay}$ to avoid excessive detail loss [S1].

## Background
Scale Decay was introduced in the GRay paper as a regularization technique to optimize the efficiency of 3D Gaussian Splatting training [S1]. It was found to effectively control final Gaussian sizes and encourage pruning of occluded or out-of-view primitives, leading to noticeable FPS improvements with minimal quality impact [S1].

## Sources
[S1] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
