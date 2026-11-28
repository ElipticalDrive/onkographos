# TomOpt Optimisation Callbacks

Also called: TomOpt Update Cycle, TomOpt callbacks, TomOpt fitting procedure, TomOpt training loop

## Overview
TomOpt Optimisation Callbacks are a modular system of hooks that augment the basic differentiable optimization loop for muon tomography detector design, stabilizing the training process and ensuring physically meaningful updates [S1]. Unlike generic training loops, this method explicitly handles the stochastic nature of muon propagation and the physical constraints of detector geometry through specialized callbacks such as `MuonResampler` and `PanelCentreing` [S1]. It distinguishes itself by integrating physical unit scaling for learning rates (`OptConfig`) and in-place de-biasing (`LinearCorrection`) directly into the gradient update cycle [S1].

## How it works
The method optimizes detector parameters (e.g., panel positions, sizes) by minimizing a loss function derived from the performance of a reconstructed muon track model against ground truth [S1]. The core optimization relies on back-propagating the derivative of the loss through the inference and hit-recording steps to compute analytic effects of each detector parameter [S1]. Standard PyTorch optimizers (SGD or Adam) are used to step in the direction of steepest descent [S1].

### Inputs and Outputs
*   **Inputs:** A batch of passive volumes (targets), a batch of generated muons, current detector parameters (panel positions, sizes, etc.), and a set of active callbacks.
*   **Outputs:** Updated detector parameters, loss values, and diagnostic metrics.

### Data Structures
The system maintains the state of the detector (panel coordinates, dimensions) and the history of loss values. Muons are represented as batches of trajectories. The `LinearCorrection` callback maintains raw predictions and targets collected during a volume batch to re-fit linear correction parameters [S1].

### Numbered Steps
1.  **Scanning:** Pass a batch of muons through a given passive volume. Infer their tracks and compute the loss for that volume. Muons may be split into smaller batches for computational reasons [S1].
2.  **Epoch Aggregation:** Repeat the scanning process for a batch of volumes. Compute the sum or average of their losses. This aggregate value is used to update the detector parameters via back-propagation [S1].
3.  **Callback Execution:** Callbacks are invoked at specific stages:
    *   **Pre-Scan:** `MuonResampler` regenerates muons until all interact with the passive volume [S1].
    *   **Pre-Backprop:** `SpreadRangeLoss` collects predictions over the volume batch to compute population-based loss metrics [S1]. `LinearCorrection` modifies inference outputs in-place and collects raw data for re-fitting [S1].
    *   **Post-Backprop/Pre-Update:** `NoMoreNaNs` sets any NaN gradients to zero [S1].
    *   **Post-Update:** `PanelCentreing` adjusts panel positions to stabilize optimization [S1]. `OptConfig` may adjust learning rates based on warm-up phase averages [S1].
4.  **Termination:** Run for a specified number of epochs. Validation mode may be run where parameters are not updated [S1].

### Formulas
The learning rate $\gamma$ is computed using the `OptConfig` callback to achieve a desired physical update rate:

$$
\gamma = \frac{\text{desired change}}{\nabla\bar{\text{param}}}
$$

where $\nabla\bar{\text{param}}$ is the average gradient received by the parameter during a warm-up phase with fixed parameters [S1]. For example, if the desired change for panel xy positions is 1 cm per iteration:

$$
\gamma = \frac{1 \text{ cm}}{\nabla\bar{x}_{y-pos}}
$$

The loss function is purely based on performance, with no balancing coefficients between performance and cost [S1].

### Parameters
*   **Optimizer:** Standard PyTorch optimizers (SGD, Adam) [S1].
*   **Learning Rate Schedule:** `OneCycle` schedule adapts the learning rate (and optionally momentum) to allow larger updates at the beginning and gradual decrease for convergence [S1].
*   **Epochs:** Specified number of epochs for the fit [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tomopt_training_loop(detector, volumes, muons, callbacks, num_epochs):
    # Warm-up phase for OptConfig
    if 'OptConfig' in callbacks:
        warmup_gradients = collect_gradients(detector, volumes, muons, steps=10)
        detector.set_learning_rates_from_gradients(warmup_gradients)

    for epoch in range(num_epochs):
        epoch_loss = 0.0
        volume_batch_losses = []
        volume_batch_predictions = []
        
        for volume in volumes:
            # 1. Scanning
            # MuonResampler: Regenerate muons until all interact with volume
            active_muons = muons
            if 'MuonResampler' in callbacks:
                active_muons = callbacks['MuonResampler'].resample(muons, volume)
            
            # Pass muons through volume, infer tracks
            predictions = detector.infer_tracks(active_muons, volume)
            
            # LinearCorrection: Modify outputs in-place
            if 'LinearCorrection' in callbacks:
                predictions = callbacks['LinearCorrection'].apply(predictions, volume)
                callbacks['LinearCorrection'].collect_raw(predictions, volume)
            
            # Compute loss
            loss = compute_loss(predictions, volume)
            
            # SpreadRangeLoss: Collect predictions for population-based loss
            if 'SpreadRangeLoss' in callbacks:
                volume_batch_predictions.append(predictions)
            
            epoch_loss += loss
            volume_batch_losses.append(loss)
        
        # Aggregate loss
        avg_loss = sum(volume_batch_losses) / len(volume_batch_losses)
        
        # Back-propagation
        avg_loss.backward()
        
        # NoMoreNaNs: Set NaN gradients to zero
        if 'NoMoreNaNs' in callbacks:
            callbacks['NoMoreNaNs'].sanitize_gradients(detector)
        
        # Update parameters
        detector.step()
        
        # Post-update callbacks
        if 'PanelCentreing' in callbacks:
            callbacks['PanelCentreing'].centre_panels(detector)
        
        if 'LinearCorrection' in callbacks:
            callbacks['LinearCorrection'].refit_correction()
        
        # SpreadRangeLoss: Set loss value prior to back-propagation (if not done per volume)
        if 'SpreadRangeLoss' in callbacks:
            avg_loss = callbacks['SpreadRangeLoss'].compute(volume_batch_predictions)
            # Note: In this loop structure, loss is already computed. 
            # The callback likely adjusts the loss value used for the epoch metric.
        
        # Logging and Saving
        if 'PanelMetricLogger' in callbacks:
            callbacks['PanelMetricLogger'].log(detector, avg_loss)
        
        if 'EpochSave' in callbacks:
            callbacks['EpochSave'].save(detector, epoch)
        
        # OneCycle: Update learning rate schedule
        if 'OneCycle' in callbacks:
            callbacks['OneCycle'].step()

    return detector
```

### Edge Cases and Pitfalls
*   **NaN Gradients:** Invalid gradients may occur due to the complexity of reconstruction and inference. The `NoMoreNaNs` callback is essential to prevent optimization failure [S1].
*   **Instability:** Unconstrained panel movement can lead to divergences. `PanelCentreing` limits configuration originality to avoid instabilities [S1].
*   **Muon Background:** Muons that do not interact with the passive volume form background noise. `MuonResampler` improves generation efficiency by regenerating these muons [S1].
*   **Learning Rate Selection:** Standard learning rate finders are designed for DNNs without physical meaning. `OptConfig` uses physical units (e.g., meters) to set appropriate learning rates [S1].

## Subtopics

### MuonResampler
Regenerates muons to ensure interaction with the passive volume, removing background and improving generation efficiency [S1].

### LinearCorrection
Implements a de-biasing process by modifying inference outputs in-place and re-fitting linear correction parameters after each detector update [S1].

### SpreadRangeLoss
Computes loss metrics that require a population of predictions, collecting predictions over the volume batch before back-propagation [S1].

### NoMoreNaNs
Sanitizes gradients by setting NaN values to zero after back-propagation to prevent optimization failure [S1].

### OptConfig
Computes learning rates based on desired physical update rates (e.g., cm/iteration) derived from average gradients during a warm-up phase [S1].

### PanelCentreing
Stabilizes optimization by setting the mean x-y position for panels above and below the passive volume separately [S1].

### PanelMetricLogger
Provides real-time feedback on detector layout, loss history, and diagnostics [S1].

### EpochSave
Saves the detector state after every update for recovery or selection [S1].

### OneCycle
Adapts the learning rate using a 1-cycle schedule to allow large initial updates and gradual convergence [S1].

## Operating characteristics
*   **Time Complexity:** The optimization runs for a specified number of epochs. Each epoch involves scanning multiple volumes with batches of muons. The computational cost is dominated by muon propagation and track inference [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The callback system stabilizes the optimization process, allowing for the optimization of detector parameters that would otherwise be unstable [S1].
*   **Strengths:** Handles physical constraints explicitly; robust against NaN gradients and background muons; allows for physically meaningful learning rate selection [S1].
*   **Limitations:** `PanelCentreing` limits the originality of detector configurations that can be explored [S1]. The implementation of multiple Coulomb scattering prioritizes efficiency over precision [S1].
*   **When to use:** When optimizing detector geometry for muon tomography where physical stability and meaningful parameter updates are critical [S1].

## Background
TomOpt is a differential optimization framework for task- and constraint-aware design of particle detectors in the context of muon tomography [S1]. The callback system was introduced to allow users to augment the basic optimization loop with different functionality to stabilize the process, addressing challenges such as muon background, gradient instability, and physical parameter scaling [S1]. The method leverages standard PyTorch optimizers and differentiable rendering/inference pipelines [S1].

## Sources
[S1] TomOpt: Differential optimisation for task- and constraint-aware design of particle detectors in the context of muon tomography, https://doi.org/10.1088/2632-2153/ad52e7, fetched 2026-10-26
