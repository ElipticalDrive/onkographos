# Checkpoint Loading and Resume Training
Also called: NeRF Checkpointing
## Overview
Checkpoint Loading and Resume Training enables the restoration of a neural radiance field's optimization state from disk to continue training or perform evaluation without restarting from scratch [S1, unofficial]. It differs from [Training Loop Setup](1522-training-loop-setup.md) by handling state persistence and recovery rather than initial environment configuration, and from [Checkpoint Serialization](#checkpoint-serialization) by focusing on the ingestion and restoration of those states [S1, unofficial][S2, unofficial].
## How it works
### Inputs and outputs
- **Inputs**: Path to the checkpoint directory; model architecture definition; optimizer configuration; (optional) gradient scaler configuration.
- **Outputs**: Restored model parameters; restored optimizer state; current iteration counter; (optional) gradient scaler state.

### Data structures
The checkpoint file is a serialized dictionary (typically PyTorch `.pth` format) containing:
- `model_state_dict`: OrderedDict mapping parameter names to tensors.
- `optimizer_state_dict`: OrderedDict containing optimizer state (e.g., Adam moments).
- `iteration`: Integer representing the training step at which the checkpoint was saved.
- `scaler_state_dict`: Optional dictionary for gradient scaler state if mixed precision is used [S1, unofficial].

### Steps
1. Initialize the model and optimizer objects with their respective architectures and hyperparameters.
2. Load the checkpoint file from disk.
3. Load the `model_state_dict` into the model using `model.load_state_dict()`.
4. Load the `optimizer_state_dict` into the optimizer using `optimizer.load_state_dict()`.
5. If mixed precision training was used, load the gradient scaler state using `scaler.load_state_dict()` [S1, unofficial].
6. Extract the `iteration` value from the checkpoint to set the current training step.
7. Resume the training loop from the extracted iteration.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def load_checkpoint(model, optimizer, checkpoint_path, scaler=None):
    """
    Loads model and optimizer states from a checkpoint file.
    
    Args:
        model: The neural network model.
        optimizer: The optimizer instance.
        checkpoint_path: Path to the .pth checkpoint file.
        scaler: Optional torch.cuda.amp.GradScaler instance.
        
    Returns:
        iteration: The iteration number stored in the checkpoint.
    """
    # Load the checkpoint file
    checkpoint = torch.load(checkpoint_path, map_location='cpu')
    
    # Load model state
    model.load_state_dict(checkpoint['model_state_dict'])
    
    # Load optimizer state
    optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
    
    # Load gradient scaler state if present and scaler is provided
    if scaler is not None and 'scaler_state_dict' in checkpoint:
        scaler.load_state_dict(checkpoint['scaler_state_dict'])
    
    # Extract current iteration
    iteration = checkpoint['iteration']
    
    return iteration
```

### Parameters
- `checkpoint_path`: String path to the checkpoint file.
- `iteration`: Integer, typically extracted from the filename or the checkpoint metadata.

### Edge cases and pitfalls
- **Device mismatch**: Ensure the checkpoint is loaded to the correct device (CPU/GPU) before loading states, or use `map_location` [S1, unofficial].
- **Architecture mismatch**: The model architecture must exactly match the one used to save the checkpoint; otherwise, `load_state_dict` will fail or produce silent errors [G].
- **Optimizer state loss**: If the optimizer hyperparameters (e.g., learning rate, betas) are changed after loading, the optimizer state may need to be re-initialized or adjusted, as the loaded state reflects the previous hyperparameters [G].
- **Partial checkpoints**: If training crashes mid-write, the checkpoint file may be corrupted. Implement atomic writes or verify file integrity [G].

## Subtopics

### Checkpoint Serialization
Checkpoint Serialization is the process of saving the current training state to disk at specified intervals [S2, unofficial]. It differs from loading by focusing on the output and timing of state persistence. It is chosen when fault tolerance or intermediate result inspection is required. It uses a list of specific iteration numbers (`checkpoint_iterations`) to determine when to save [S2, unofficial]. The serialization captures the Gaussian model state (or NeRF parameters) via a method like `gaussians.capture()` and saves it to a file named `chkpnt{iteration}.pth` in the scene's model path [S2, unofficial].

## Operating characteristics
- **Time complexity**: Loading is $O(N)$ where $N$ is the number of parameters, dominated by I/O and deserialization. Serialization is also $O(N)$ plus I/O overhead.
- **Memory complexity**: Requires memory for the model and optimizer states, plus temporary buffer for I/O.
- **Performance**: I/O bound; large models may take seconds to load/save.
- **Strengths**: Enables long-running training, fault tolerance, and evaluation of intermediate models.
- **Limitations**: Disk space consumption grows with checkpoint frequency; loading time adds to startup latency.
- **When to use**: For training runs expected to last hours/days, or when frequent evaluation is needed.
- **When not to use**: For short experiments or when disk I/O is a bottleneck and checkpoints are not needed.

## Background
Checkpointing is a standard practice in deep learning optimization to ensure training continuity. In the context of NeRF and radiance field optimization, it allows researchers to resume training after interruptions or to analyze the convergence of the radiance field at specific iterations [S1, unofficial][S2, unofficial].

## Sources
[S1] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S2] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
