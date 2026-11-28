# Checkpoint Loading and Resume Training
Also called: NeRF Checkpointing
## Overview
Checkpoint Loading and Resume Training enables the restoration of a neural radiance field's optimization state from disk to continue training or perform evaluation without restarting from scratch [S1, unofficial].
## How it works
### Inputs and outputs
The method takes a checkpoint file path and restores the model parameters, optimizer state, and training iteration counter to resume training or evaluation [S1, unofficial].
### Data structures
The checkpoint file is a serialized dictionary containing the following fields [S1, unofficial]:
- `model_state_dict`: The model's parameter tensors.
- `optimizer_state_dict`: The optimizer's internal state.
- `iteration`: The training step number at which the checkpoint was saved.
- `scaler_state_dict`: The gradient scaler state, loaded only if mixed precision was used.
### Steps
1. Load the `model_state_dict` into the model [S1, unofficial].
2. Load the `optimizer_state_dict` into the optimizer [S1, unofficial].
3. Load the gradient scaler state if it was used during training [S1, unofficial].
4. Extract the `iteration` value from the checkpoint to set the current training step [S1, unofficial].
### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def load_checkpoint(model, optimizer, checkpoint_path, scaler=None):
    checkpoint = torch.load(checkpoint_path)
    model.load_state_dict(checkpoint['model_state_dict'])
    optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
    if scaler is not None and 'scaler_state_dict' in checkpoint:
        scaler.load_state_dict(checkpoint['scaler_state_dict'])
    iteration = checkpoint['iteration']
    return iteration
```
### Parameters
- `checkpoint_path`: String path to the checkpoint file [S1, unofficial].
- `iteration`: Integer representing the training step, extracted from the checkpoint [S1, unofficial].
### Edge cases and pitfalls
Not given in the sources.
## Subtopics

### Checkpoint Serialization
Checkpoint Serialization saves the current training state to disk at specified intervals [S2, unofficial]. It uses a list of specific iteration numbers (`checkpoint_iterations`) to determine when to save [S2, unofficial]. The process serializes the Gaussian model state via `gaussians.capture()` paired with the current iteration number, and saves it to `scene.model_path + "/chkpnt" + str(iteration) + ".pth"` [S2, unofficial].

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S2] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
