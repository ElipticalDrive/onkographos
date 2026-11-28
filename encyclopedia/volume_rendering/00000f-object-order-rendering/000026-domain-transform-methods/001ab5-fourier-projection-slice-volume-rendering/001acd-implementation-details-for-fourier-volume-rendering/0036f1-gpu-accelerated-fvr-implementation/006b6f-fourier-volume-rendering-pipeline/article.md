# Fourier Volume Rendering Pipeline

Also called: LaunchVolumeProcessingCUDA

## Overview
The Fourier Volume Rendering (FVR) Pipeline orchestrates the complete GPU-accelerated processing of volumetric data in the frequency domain. It transforms spatial volume data into a spectral representation using 3D Fast Fourier Transforms (FFT), applies necessary spectral shifts, and packs the result into a format suitable for OpenGL rendering. It sets itself apart from standard ray-casting methods by performing the entire volume integration in the frequency domain, leveraging the convolution theorem to accelerate computation on CUDA-enabled GPUs.

## How it works
**Inputs:**
- `mExtractedVolumeData`: A 1D array of floating-point values representing the spatial volume voxels.
- `sDataSize`: The unified dimension size of the cubic volume ($N \times N \times N$).
- `ptrTo1DArray`: A pointer to a host-side 1D float array, linked to an OpenGL Pixel Buffer Object (PBO) via a CUDA Graphics Resource, where the final reconstructed image will be stored.

**Outputs:**
- `ptrTo1DArray`: Populated with the interleaved real and imaginary components (though typically only the real part is used for rendering) of the spectral volume, ready for OpenGL texture mapping.

**Data Structures:**
1.  **Host Complex Array (`sHostComplexArray`)**:
    -   Type: `cufftComplex*`
    -   Layout: Interleaved `float x` (real) and `float y` (imaginary).
    -   Size: $N^3$ elements. Total bytes: $N^3 \times 2 \times 4$ bytes (assuming 32-bit floats).
    -   Content: Real part contains volume data; Imaginary part is zero-initialized.
2.  **Device Complex Arrays (`sDeviceComplexArray`, `sDeviceComplexArrayTemp`)**:
    -   Type: `cufftComplex*`
    -   Size: $N^3$ elements each.
    -   Role: `sDeviceComplexArray` holds the current state (spatial or spectral). `sDeviceComplexArrayTemp` is used for intermediate FFT outputs and shifts.
3.  **Device 1D Float Array (`sDevice1DFloatArray`)**:
    -   Type: `float*`
    -   Size: $N^3 \times 2$ floats (to store complex data as floats for packing).
    -   Role: Intermediate buffer for packing complex data into a linear float array for OpenGL.

**Steps:**

1.  **Host-Side Preparation**:
    -   Calculate total voxels: $mVolumeSize = sDataSize^3$.
    -   Allocate `sHostComplexArray` of size $mVolumeSize$.
    -   Iterate $i$ from $0$ to $mVolumeSize - 1$:
        -   Set `sHostComplexArray[i].x = mExtractedVolumeData[i]`.
        -   Set `sHostComplexArray[i].y = 0.0f`.
    -   *Pitfall*: CuFFT requires complex input in interleaved format. Failing to zero-initialize the imaginary component will produce incorrect frequency-domain results [S1, unofficial].

2.  **Device Memory Allocation**:
    -   Allocate `sDeviceComplexArray` and `sDeviceComplexArrayTemp` on the GPU using `cudaMalloc`. Size: $sDataSize^3 \times sizeof(cufftComplex)$.
    -   Allocate `sDevice1DFloatArray` on the GPU. Size: $sDataSize^3 \times 2 \times sizeof(float)$.

3.  **Data Upload**:
    -   Copy `sHostComplexArray` to `sDeviceComplexArray` using `cudaMemcpyHostToDevice`.
    -   Copy `sHostComplexArray` to `sDeviceComplexArrayTemp` using `cudaMemcpyHostToDevice`.

4.  **Spatial Shifting**:
    -   Configure CUDA execution parameters for 3D shift:
        -   Block: `dim3 sBlock(16, 16, 1)`
        -   Grid: `dim3 sGrid((sDataSize / 16), (sDataSize / 16), 1)`
    -   Launch `fftShift3D_i` kernel:
        -   Input: `sDeviceComplexArray`
        -   Output: `sDeviceComplexArrayTemp`
        -   Size: `sDataSize`
        -   Config: `sBlock`, `sGrid`
    -   This step centers the zero-frequency component in the spatial domain (or prepares for FFT depending on implementation specifics, but here it precedes FFT, suggesting a shift to handle boundary conditions or align with FFT conventions). *Note: Standard FFT pipelines often shift *after* FFT to center the spectrum. The source code performs a shift before FFT, then FFT, then shift again. This specific sequence is preserved as per the source.*

5.  **3D Forward FFT**:
    -   Setup 3D cuFFT plan: `cufftPlan3d(&fftPlan3D, sDataSize, sDataSize, sDataSize, CUFFT_C2C)`.
    -   Execute forward complex-to-complex FFT:
        -   Input: `sDeviceComplexArrayTemp`
        -   Output: `sDeviceComplexArray`
        -   Direction: `CUFFT_FORWARD`
    -   Destroy plan: `cufftDestroy(fftPlan3D)`.

6.  **Spectral Shifting**:
    -   Configure CUDA execution parameters for 3D shift:
        -   Block: `dim3 sBlock(16, 16, 1)`
        -   Grid: `dim3 sGrid((sDataSize / 16), (sDataSize / 16), 1)`
    -   Launch `fftShift3D_i` kernel:
        -   Input: `sDeviceComplexArray`
        -   Output: `sDeviceComplexArrayTemp`
        -   Size: `sDataSize`
        -   Config: `sBlock`, `sGrid`
    -   This step wraps around the spectral volume to center its DC component, which is standard for visualization and further processing in the frequency domain.

7.  **Packing to 1D Float Array**:
    -   Configure packing execution parameters:
        -   Block: `dim3 sBlock1D(512, 1, 1)`
        -   Grid: `dim3 sGrid1D((sDataSize * sDataSize * sDataSize) / 512, 1, 1)`
    -   Launch `Pack1DComplexArray` kernel:
        -   Input: `sDeviceComplexArrayTemp`
        -   Output: `sDevice1DFloatArray`
        -   Size: `sDataSize`
    -   This kernel interleaves the real and imaginary parts of the complex array into a single float array.

8.  **Result Retrieval**:
    -   Copy `sDevice1DFloatArray` back to the host array `mTextureArray` (linked to `ptrTo1DArray`) using `cudaMemcpyDeviceToHost`.
    -   Size: $sDataSize^3 \times 2 \times sizeof(float)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def LaunchVolumeProcessingCUDA(mExtractedVolumeData, sDataSize, ptrTo1DArray):
    """
    Orchestrates the GPU volume processing pipeline.
    
    Args:
        mExtractedVolumeData (list[float]): Spatial volume data.
        sDataSize (int): Unified dimension size of the cubic volume.
        ptrTo1DArray (list[float]): Output pointer to the interleaved 1D float array.
    """
    import math

    # 1. Host-Side Preparation
    mVolumeSize = sDataSize ** 3
    sHostComplexArray = [complex(0, 0) for _ in range(mVolumeSize)]
    
    for i in range(mVolumeSize):
        sHostComplexArray[i] = complex(mExtractedVolumeData[i], 0.0)

    # 2. Device Memory Allocation
    # Note: In a real implementation, these would be cudaMalloc calls.
    # Here we simulate the allocation sizes.
    sSizeComplex = mVolumeSize * 8  # sizeof(cufftComplex) is 8 bytes (2 floats)
    sSizeFloat = mVolumeSize * 8    # sizeof(float) * 2 * mVolumeSize
    
    sDeviceComplexArray = [complex(0, 0)] * mVolumeSize
    sDeviceComplexArrayTemp = [complex(0, 0)] * mVolumeSize
    sDevice1DFloatArray = [0.0] * (mVolumeSize * 2)

    # 3. Data Upload
    # Simulate cudaMemcpyHostToDevice
    for i in range(mVolumeSize):
        sDeviceComplexArray[i] = sHostComplexArray[i]
        sDeviceComplexArrayTemp[i] = sHostComplexArray[i]

    # 4. Spatial Shifting
    sBlock = (16, 16, 1)
    sGrid = (sDataSize // 16, sDataSize // 16, 1)
    # Launch fftShift3D_i kernel
    fftShift3D_i(sDeviceComplexArray, sDeviceComplexArrayTemp, sDataSize, sBlock, sGrid)

    # 5. 3D Forward FFT
    # Setup 3D cuFFT plan: cufftPlan3d(&fftPlan3D, sDataSize, sDataSize, sDataSize, CUFFT_C2C)
    # Execute forward complex-to-complex FFT
    cufftExecC2C(sDeviceComplexArrayTemp, sDeviceComplexArray, CUFFT_FORWARD)
    # Destroy plan

    # 6. Spectral Shifting
    # Launch fftShift3D_i kernel
    fftShift3D_i(sDeviceComplexArray, sDeviceComplexArrayTemp, sDataSize, sBlock, sGrid)

    # 7. Packing to 1D Float Array
    sBlock1D = (512, 1, 1)
    sGrid1D = (mVolumeSize // 512, 1, 1)
    # Launch Pack1DComplexArray kernel
    Pack1DComplexArray(sDeviceComplexArrayTemp, sDevice1DFloatArray, sDataSize, sBlock1D, sGrid1D)

    # 8. Result Retrieval
    # Simulate cudaMemcpyDeviceToHost
    for i in range(mVolumeSize * 2):
        ptrTo1DArray[i] = sDevice1DFloatArray[i]

def fftShift3D_i(devArrayInput, devArrayOutput, sDataSize, block, grid):
    """
    Performs a 3D FFT shift on the device.
    This kernel handles the wrapping of the array to center the DC component.
    """
    # Implementation details of the kernel are not fully provided in the source,
    # but it involves calculating indices and swapping quadrants.
    # The source code shows a conditional swap based on index ranges.
    # For the purpose of this pipeline description, we assume this kernel
    # correctly shifts the array as per standard FFT shift logic.
    pass

def cufftExecC2C(input_array, output_array, direction):
    """
    Simulates the cuFFT execution.
    """
    # In a real implementation, this calls the cuFFT library.
    # Here, we assume the output_array is filled with the FFT result of input_array.
    pass

def Pack1DComplexArray(devArrayInput, devArrayOutput, sDataSize, block, grid):
    """
    Packs the complex array into a 1D float array.
    """
    # Implementation details of the kernel are not fully provided in the source.
    # It interleaves real and imaginary parts.
    pass
```

## Operating characteristics
-   **Time Complexity**: The FFT computation complexity depends on the CuFFT implementation, typically $O(N^3 \log N)$ for a volume of size $N^3$. The spatial and spectral shifts are $O(N^3)$.
-   **Memory Complexity**: Memory allocation and host-to-device transfer scale as $O(N^3)$. The pipeline requires storage for two complex arrays and one float array on the device, totaling approximately $3 \times N^3 \times 8$ bytes (for 32-bit floats) plus host overhead.
-   **Performance**: The implementation achieves a speed-up of 117x compared to a single-threaded hybrid CPU/GPU implementation [S1, unofficial].
-   **Strengths**: High performance due to GPU acceleration and the efficiency of FFTs for volume rendering. Suitable for large volumetric datasets where ray-casting would be too slow.
-   **Limitations**: Requires cubic volumes (or padding to cubic). The frequency domain representation may not be as intuitive for direct manipulation as spatial domain data. Precision issues may arise if the imaginary part is not correctly zero-initialized.

## Background
The Fourier Volume Rendering Pipeline was implemented in the Master's thesis project "High Performance Fourier Volume Rendering on Graphics Processing Units (GPUs)" by Marwan Abdellah in April 2011 [S1, unofficial]. It is part of the `cuFVR` project, which aims to leverage CUDA and CuFFT for efficient volume rendering. The method builds on the principles of Fourier Projection-Slice Volume Rendering, where the volume rendering integral is computed in the frequency domain.

## Sources
[S1] marwan-abdellah/cuFVR, https://github.com/marwan-abdellah/cuFVR, fetched 2026-10-26 (unofficial)
