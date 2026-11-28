# Memory-In Task Creation

Also called: Memory-Out Task Creation

## Overview
This method creates a "memory-in" task to feed a pre-existing NumPy array into a processing graph, or a "memory-out" task to receive processed data into a pre-allocated NumPy array. It differs from file-based input tasks by operating directly on in-memory data structures, eliminating disk I/O overhead for intermediate steps. It builds on [Data Contiguity Check](18253-data-contiguity-check.md) to ensure the underlying array is compatible with the processing engine.

## How it works
**Inputs and Outputs:**
- **Input:** A NumPy array (for memory-in) or a pre-allocated NumPy array (for memory-out).
- **Output:** A configured task object (`in_task` or `out_task`) attached to the processing graph.

**Data Structures:**
The method interacts with the UFO (Unified Framework for Optimization) task system. The task object exposes a `props` attribute containing configuration fields.
- `in_task.props`:
  - `complex_layout`: Boolean indicating if the data is complex.
  - `pointer`: Integer memory address pointing to the start of the array data.
  - `width`: Integer, number of columns (doubled for complex data).
  - `height`: Integer, number of rows.
  - `number_of_frames`: Integer, fixed at 1.
  - `bit_depth`: Integer, fixed at 32.
- `out_task.props`:
  - `pointer`: Integer memory address pointing to the start of the array data.
  - `max_size`: Integer, total number of bytes in the array.

**Steps:**
1. **Validate Dimensions:** Check if `array.ndim == 2`. If not, raise `ValueError`.
2. **Validate Data Type:** Check if `array.dtype` is `float32` or `complex64`. If not, raise `ValueError`.
3. **Determine Complexity:** Set `is_complex = (array.dtype == np.complex64)`.
4. **Retrieve Task:** Call `get_task('memory-in')` for input or `get_task('memory-out')` for output.
5. **Configure Memory-In Task:**
   - Set `in_task.props.complex_layout = is_complex`.
   - Set `in_task.props.pointer = array.__array_interface__['data'][0]`.
   - Set `in_task.props.width = 2 * array.shape[1]` if `is_complex` else `array.shape[1]`.
   - Set `in_task.props.height = array.shape[0]`.
   - Set `in_task.props.number_of_frames = 1`.
   - Set `in_task.props.bit_depth = 32`.
   - Store reference: `in_task.np_array = array` to prevent garbage collection.
6. **Configure Memory-Out Task:**
   - Set `out_task.props.pointer = array.__array_interface__['data'][0]`.
   - Set `out_task.props.max_size = array.nbytes`.
   - Store reference: `out_task.np_array = array` to prevent garbage collection.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def create_memory_in_task(array):
    # Validate dimensions
    if array.ndim != 2:
        raise ValueError("Input array must be 2D.")
    
    # Validate dtype
    if array.dtype not in [np.float32, np.complex64]:
        raise ValueError("Input array dtype must be float32 or complex64.")
    
    # Determine if complex
    is_complex = (array.dtype == np.complex64)
    
    # Get task
    in_task = get_task('memory-in')
    
    # Set properties
    in_task.props.complex_layout = is_complex
    in_task.props.pointer = array.__array_interface__['data'][0]
    
    # Width calculation
    if is_complex:
        in_task.props.width = 2 * array.shape[1]
    else:
        in_task.props.width = array.shape[1]
        
    in_task.props.height = array.shape[0]
    in_task.props.number_of_frames = 1
    in_task.props.bit_depth = 32
    
    # Prevent garbage collection
    in_task.np_array = array
    
    return in_task

def create_memory_out_task(array):
    # Validate dimensions
    if array.ndim != 2:
        raise ValueError("Output array must be 2D.")
    
    # Validate dtype
    if array.dtype not in [np.float32, np.complex64]:
        raise ValueError("Output array dtype must be float32 or complex64.")
    
    # Get task
    out_task = get_task('memory-out')
    
    # Set properties
    out_task.props.pointer = array.__array_interface__['data'][0]
    out_task.props.max_size = array.nbytes
    
    # Prevent garbage collection
    out_task.np_array = array
    
    return out_task
```

**Parameters:**
- `array`: NumPy array, 2D, `float32` or `complex64`.
- `get_task(task_name)`: Function to retrieve a task instance from the graph context.

**Edge Cases and Pitfalls:**
- **Garbage Collection:** The NumPy array must remain alive until the graph is executed. The method attaches the array to the task object (`in_task.np_array = array` or `out_task.np_array = array`) to prevent Python's garbage collector from freeing the memory before processing [S1, unofficial].
- **Memory Address:** The pointer is obtained via `array.__array_interface__['data'][0]`. This assumes the array is contiguous and its data buffer is valid.

## Operating characteristics
- **Time Complexity:** O(1) for task creation, as it involves only property assignments and validation.
- **Memory Complexity:** O(1) additional memory, as it reuses existing array memory.
- **Strengths:** Eliminates disk I/O for intermediate data; allows direct manipulation of NumPy arrays within the processing graph.
- **Limitations:** Requires the array to be in memory; strict type and dimension constraints; potential for memory leaks if references are not managed correctly (mitigated by the internal reference storage).
- **When to use:** When integrating NumPy-based data processing into a UFO graph without writing to disk.
- **When not to use:** For large datasets that do not fit in memory; when data is already in file format (use file-based input tasks).

## Background
This method is part of the UFO (Unified Framework for Optimization) toolkit, specifically within the `tofu` package. It enables seamless integration of NumPy arrays into the UFO processing pipeline, supporting both complex and real-valued 2D data. The approach is documented in the UFO toolkit repository [S1, unofficial].

## Sources
[S1] ufo-kit/tofu, https://github.com/ufo-kit/tofu, fetched 2026-10-26 (unofficial)
