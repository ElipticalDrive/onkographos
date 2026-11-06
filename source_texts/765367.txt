# Parallel Prefix Scan using MPI

This repository contains a C program that performs a **Prefix Scan (also known as a Prefix Sum)** operation using **MPI (Message Passing Interface)** for parallel computation. 

The program supports any input size (power of 2 or not) and distributes work across multiple processes, handling communication between them using `MPI_Send` and `MPI_Recv`.

---

## Features

- Handles arbitrary-sized input arrays.
- Implements a two-phase (up and down) approach for prefix scan:
  - **Up Phase:** Aggregates partial sums up a binary tree structure.
  - **Down Phase:** Propagates prefix sums back down the tree.
- Efficiently parallelised using MPI.
- Includes test cases for validation across different process and input configurations.

---
