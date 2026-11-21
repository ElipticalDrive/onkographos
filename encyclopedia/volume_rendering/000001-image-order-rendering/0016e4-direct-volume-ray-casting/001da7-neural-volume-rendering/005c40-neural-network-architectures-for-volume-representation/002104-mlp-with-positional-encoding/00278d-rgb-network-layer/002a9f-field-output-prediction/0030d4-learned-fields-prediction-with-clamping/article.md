# Learned Fields Prediction with Clamping

Also called: Clamped Field Output Prediction

## Overview
This method predicts field outputs from a neural network model and applies clamping to log density ($\log n_e$) and log temperature ($\log T$) based on configured bounds [S1, unofficial]. It serves as the final output layer in neural volume rendering pipelines, ensuring that physically impossible or numerically unstable values are constrained before further processing [S1, unofficial]. It differs from raw field output prediction by enforcing hard bounds on specific physical quantities, addressing the instability of unconstrained neural network outputs in tomographic reconstruction [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A batch of 3D positions $x_{AABB}$ mapped to an Axis-Aligned Bounding Box (AABB) [S1, unofficial].
*   **Output:** Clamped log density $\log n_e$ and clamped log temperature $\log T$ [S1, unofficial].

**Data Structures**
The method relies on a configuration object containing:
*   `clamp`: Boolean flag enabling/disabling clamping [S1, unofficial].
*   `log_ne_min`, `log_ne_max`: Float bounds for log density [S1, unofficial].
*   `log_temp_min`, `log_temp_max`: Float bounds for log temperature [S1, unofficial].
*   `bounds_dict`: Dictionary containing global bounds for density (`ne`), temperature (`temp`), and radius (`r`) used to validate constraints [S1, unofficial].

**Steps**
1.  **Predict Raw Outputs:** Call the underlying prediction function `self._predict_field_outputs(x_AABB)` to obtain raw outputs `outs` [S1, unofficial]. This function returns a structure containing `log_ne` and `log_temp` [S1, unofficial].
2.  **Extract Log Density:** Extract `log_ne = outs.log_ne` [S1, unofficial].
3.  **Clamp Log Density:** If `self.clamp` is True, clamp `log_ne` to the range `[self.log_ne_min, self.log_ne_max]` [S1, unofficial]. The clamping operation is defined as:
    $$ \log n_e^{clamped} = \text{clamp}(\log n_e, \log n_{e, min}, \log n_{e, max}) $$
    where $\text{clamp}(v, min, max) = \min(\max(v, min), max)$ [S1, unofficial].
4.  **Extract Log Temperature:** Extract `log_temp = outs.log_temp` [S1, unofficial].
5.  **Clamp Log Temperature:** If `log_temp` is not None and `self.clamp` is True, clamp `log_temp` to the range `[self.log_temp_min, self.log_temp_max]` [S1, unofficial]. The clamping operation is defined as:
    $$ \log T^{clamped} = \text{clamp}(\log T, \log T_{min}, \log T_{max}) $$
    [S1, unofficial].
6.  **Return Results:** Return the processed `outs`, `log_ne`, and `log_temp` [S1, unofficial].

**Parameters**
*   `log_ne_min`, `log_ne_max`: Derived from `bounds_dict["ne"]["global_bound_lo"]` and `bounds_dict["ne"]["global_bound_hi"]` [S1, unofficial].
*   `log_temp_min`, `log_temp_max`: Derived from `bounds_dict["temp"]["global_bound_lo"]` and `bounds_dict["temp"]["global_bound_hi"]` [S1, unofficial].
*   Typical values are not explicitly given in the sources, but bounds are printed with 6 decimal places precision [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Invalid Bounds:** If `bounds_dict["temp"]["global_bound_lo"] >= bounds_dict["temp"]["global_bound_hi"]`, a `ValueError` is raised, indicating invalid temperature bounds after combining PSI and atomic constraints [S1, unofficial].
*   **None Temperature:** If `log_temp` is None, clamping is skipped for temperature [S1, unofficial].
*   **Precision:** Bounds are stored and processed as `float64` in grid construction, suggesting high precision requirements for boundary definitions [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predict_with_clamping(self, x_AABB):
    # Step 1: Predict raw outputs
    outs = self._predict_field_outputs(x_AABB)
    
    # Step 2: Extract log density
    log_ne = outs.log_ne
    
    # Step 3: Clamp log density if enabled
    if self.clamp:
        log_ne = max(self.log_ne_min, min(log_ne, self.log_ne_max))
        
    # Step 4: Extract log temperature
    log_temp = outs.log_temp
    
    # Step 5: Clamp log temperature if enabled and present
    if self.clamp and log_temp is not None:
        log_temp = max(self.log_temp_min, min(log_temp, self.log_temp_max))
        
    # Step 6: Return results
    return outs, log_ne, log_temp
```

## Subtopics

### MLP Model (Density Baseline)
A baseline method using a positional encoder and MLP for density reconstruction [S2, unofficial]. It differs from the main clamping method by using a specific MLP architecture for density only, without explicit temperature clamping logic in the description [S2, unofficial]. Choose this for simple density-only reconstruction tasks [S2, unofficial].
*   **Definition:** Input is 3D Cartesian position mapped to AABB [S2, unofficial].
*   **Parameters:** Final-layer bias is initialized to the midpoint of the log ne clamp range [S2, unofficial].
*   **Layout:** Input is concatenation of raw position, axis-wise Fourier positional encoding (L=6 bands, freq $2^0 \dots 2^5$, 36 dims), and spherical-harmonic encoding (degree 2, 9 dims), totaling 48 dims [S2, unofficial]. MLP has 4 hidden layers, width 256, SiLU activations, and a linear final layer outputting a single scalar $\log_{10} n_e$ [S2, unofficial]. Total parameters: 210,177 [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of prediction points, dominated by the underlying `_predict_field_outputs` call [S1, unofficial].
*   **Memory Complexity:** Minimal, only storing bounds and intermediate tensors [S1, unofficial].
*   **Strengths:** Prevents numerical instability and unphysical values in neural volume rendering [S1, unofficial].
*   **Limitations:** Hard clamping may introduce discontinuities if the network output frequently exceeds bounds [S1, unofficial].
*   **When to use:** When physical bounds are known and strict adherence is required [S1, unofficial].
*   **When not to use:** When soft constraints or differentiable relaxation is preferred [S1, unofficial].

## Background
Originated in the CoroNeRF project for neural tomographic reconstruction [S1, unofficial]. Developed to handle the joint reconstruction of electron density and temperature in plasma imaging [S2, unofficial]. The clamping mechanism addresses the challenge of ensuring physically plausible outputs from neural networks trained on sparse or noisy tomographic data [S1, unofficial].

## Sources
[S1] CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Image Fidelity is Not Field Fidelity: Joint Thermodynamic Reconstruction and Error Localization in Neural Tomography, https://doi.org/10.48550/arxiv.2609.28868, fetched 2026-10-26 (unofficial)
