# MAP Grid Sampling and Filtering
Also called: MAP, MAP program

## Overview
MAP is a program that constructs a rectangular grid from irregular input data and samples it using either Z-Buffer or A-Buffer techniques, applying optional filtering [S1, unofficial]. It differs from its siblings by explicitly supporting variable plane spacing and specific filtering modes (Gaussian or density weighting) within the A-Buffer framework, whereas Z-Buffer sampling does not specify filtering in this context [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Two binary files: a grid index file and a grid data file. The index file's first line contains the number of data points, followed by x, y, and z indices. The data file contains data points in the index order [S1, unofficial].
- **Output:** A rectangular grid file conforming to VIPER's input requirements [S1, unofficial].

**Data Structures:**
- **Configuration File:** A text file containing parameters in a specific order [S1, unofficial].
- **Grid Index File:** Binary format. First line: integer $N$ (number of points). Subsequent lines: x, y, z indices for each point [S1, unofficial].
- **Grid Data File:** Binary format. Sequence of data values corresponding to the index order [S1, unofficial].

**Parameters:**
1. Input grid index file path.
2. Input grid data file path.
3. Output rectangular grid file path.
4. Carving flag: 'T' to carve, 'O' to not carve. If 'T', next six lines are i, j, k lower and upper bounds [S1, unofficial].
5. Viewing parameter: '1' for pressure numbers, '0' for mach numbers [S1, unofficial].
6. Number of planes: '1' for default (100 per axis), '0' for custom. If '0', next line contains x, y, z values [S1, unofficial].
7. Plane spacing method: '1' for constant spacing, '0' for variable spacing [S1, unofficial].
8. Sampling method: '1' for Z-Buffer, '0' for A-Buffer [S1, unofficial].
9. Filter radius multiplier: floating point value [S1, unofficial].
10. Filter type (if A-Buffer selected): '1' for Gaussian filter, '0' for density weighting filter [S1, unofficial].
11. Gaussian standard deviations: floating point number spanning the filter radius (used only with Gaussian filter) [S1, unofficial].

**Steps:**
1. Read the configuration file to determine parameters [S1, unofficial].
2. Read the binary grid index file to obtain the number of data points and their x, y, z indices [S1, unofficial].
3. Read the binary grid data file to obtain the data values in index order [S1, unofficial].
4. Construct a rectangular grid. The grid has variable or constant plane spacing based on the plane spacing method parameter [S1, unofficial].
   - If variable spacing is selected, the `-f` command line option can be used to force the program to maintain input data coordinate ranges [S1, unofficial].
5. If carving is enabled, carve the grid using the specified i, j, k lower and upper bounds [S1, unofficial].
6. Sample the grid using the selected method:
   - **Z-Buffer:** Samples the grid without specified filtering [S1, unofficial].
   - **A-Buffer:** Samples the grid and applies filtering if selected:
     - If density weighting is selected, apply density weighting [S1, unofficial].
     - If Gaussian filter is selected, apply a Gaussian filter using the specified standard deviations and filter radius multiplier [S1, unofficial].
7. Write the output rectangular grid file [S1, unofficial].

**Pitfalls:**
- Variable spacing method may alter input data coordinate ranges unless the `-f` option is used [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
MAP is part of the Visualization Toolkit and was used in conjunction with VIPER for viewing radar cross section data in three dimensions [S1, unofficial]. It was developed by the Computational Fluid Dynamics Group at WPAFB [S1, unofficial].

## Sources
[S1] DTIC ADA258905: Methods for Viewing Radar Cross Section Data in Three Dimensions, https://archive.org/details/DTIC_ADA258905, fetched 2026-10-26 (unofficial)
