# Data Contiguity Check

Also called: Check contiguity, NumPy array validation

## Overview
This method validates that a NumPy array meets the strict memory layout and type requirements of downstream volume rendering or tomography engines, specifically ensuring C-contiguity and `float32` precision. It differs from general array validation by strictly enforcing a contiguous memory block without copying unless explicitly permitted, preventing silent data corruption in high-performance C/C++ backends. It serves as a prerequisite for [Memory-In Task Creation](20984-memory-in-task-creation.md) to ensure in-memory data structures are compatible with the processing engine.

## How it works
**Inputs and Outputs:**
- Input: A NumPy array `data`.
- Output: The same array object `data` (if valid) or an exception raised. No new array is returned unless a conversion step is implemented by the caller.

**Data Structures:**
- The input `data` is a standard NumPy `ndarray`.
- Validation relies on the `data.flags` attribute, specifically `C_CONTIGUOUS`.
- Type validation relies on `data.dtype`.

**Steps:**
1. **Check Contiguity:** Verify if the array is C-contiguous. C-contiguity means the last index varies the fastest in memory, allowing a single pointer dereference for linear traversal.
   - If `data.flags['C_CONTIGUOUS']` is `False`, the array is not contiguous in C order.
   - Action: Raise an `Exception` or convert using `numpy.ascontiguousarray(data)`. Note that `ascontiguousarray` creates a copy if the array is not already contiguous.
2. **Check Data Type:** Verify if the array dtype is `float32`.
   - If `data.dtype != numpy.float32`, the precision is incorrect for the target engine.
   - Action: Raise an `Exception` or convert using `data.astype('float32')`.
3. **Check Shape:** Verify the array is not empty.
   - If `min(data.shape) == 0`, the array has a zero dimension.
   - Action: Raise an `Exception`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def check_data_contiguity(data: np.ndarray) -> np.ndarray:
    """
    Ensures numpy arrays are C-contiguous and of type float32.
    Raises an exception if not.
    """
    # Step 1: Check contiguity
    if not data.flags['C_CONTIGUOUS']:
        raise Exception("Array is not C-contiguous.")
        # Alternative: data = np.ascontiguousarray(data)

    # Step 2: Check dtype
    if data.dtype != np.float32:
        raise Exception("Array dtype is not float32.")
        # Alternative: data = data.astype('float32')

    # Step 3: Check shape
    if min(data.shape) == 0:
        raise Exception("Array has a zero dimension.")

    return data
```

**Parameters:**
- None. The method is deterministic based on the input array properties.

**Edge Cases and Pitfalls:**
- **Copy Semantics:** If the implementation chooses to convert rather than raise, `np.ascontiguousarray` and `.astype()` create copies. This violates the "no copy" intent of some high-performance pipelines if not handled carefully.
- **Fortran Order:** Arrays created with `order='F'` are not C-contiguous. They must be transposed or copied.
- **Strided Arrays:** Slices of arrays often result in non-contiguous views. The check fails for these unless a copy is made.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ for checks. $O(N)$ if a copy is required (where $N$ is the number of elements).
- **Memory Complexity:** $O(1)$ if no copy is made. $O(N)$ if a copy is made.
- **Strengths:** Prevents subtle bugs in C/C++ extensions that assume contiguous memory. Fast validation.
- **Limitations:** Does not handle non-contiguous data gracefully without copying, which may be expensive for large volumes.
- **When to use:** Before passing data to low-level rendering or reconstruction libraries (e.g., FlexTOMO, VTK) that do not perform internal validation.

## Background
This validation step is part of the data preprocessing pipeline in the FlexTOMO library, a Python-based tomography toolkit. It ensures that NumPy arrays passed to the C++-backed projector and reconstruction engines (like FDK) are in a format the underlying C++ code can directly access without overhead or errors. The method is documented in the FlexTOMO repository [S1, unofficial].

## Sources
[S1] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
