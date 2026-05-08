---
project: LFX Mentorship (rv-sparse)
tags: [c, sparse-matrices, algorithms, linear-algebra, lfx-mentorship]
status: Challenge Submitted
---

# Implementing a Zero-Allocation CSR Matrix-Vector Multiplier

**The Result:**[Challenge Submission](https://github.com/alexandr-tk/rv-sparse-challenge)

## 1. Workflow

**Task:** Implement the `sparse_multiply` function for the rv-sparse LFX Mentorship coding challenge.

**Reasoning:**

- I am applying for the LFX Mentorship program to work on the "rv-sparse: Open-source RISC-V Vector accelerated sparse linear algebra library" project.
- I want to understand how low-level systems handle linear algebra and memory constraints.
- Demonstrating my ability to write highly efficient, memory-safe C code is crucial for getting accepted.

This challenge was a great test of my algorithmic logic and pointer manipulation. To make sure my theoretical foundation was solid before touching the code, I started by reading Chapter 3 of the sparse linear algebra book to freshen up my knowledge on the mechanics of sparse matrices.

## 2. Research & Context

### Background

In linear algebra, a sparse matrix is a matrix in which most of the elements are zero. Storing all those zeros in a standard 2D grid wastes a massive amount of memory and CPU cycles during calculations. The **Compressed Sparse Row (CSR)** format solves this by stripping out the zeros and using three 1D arrays to keep track of where the remaining numbers actually belong.

### The Core Constraint

The absolute most critical rule of this challenge: **Zero dynamic memory allocation.** I was strictly forbidden from using `malloc`, `calloc`, or any heap-based memory. All buffers were pre-allocated by the caller, meaning I had to build everything perfectly on the stack using raw pointers and exact array indexing.

### Vocabulary

- **values:** An array storing only the non-zero numbers from the matrix.
- **col_indices:** An array storing the original column index for each of those non-zero numbers.
- **row_ptrs:** An array that acts as a map. It stores the running total of non-zeros, telling the computer exactly at which index in the `values` array a specific row begins and ends.
- **out_nnz:** A pointer used to report the final total Number of Non-Zeros (NNZ) back to the caller.

## 3. The Plan for Solving

I decided to split the `sparse_multiply` function into two distinct phases:

1. **The Compression Phase:** Scan the flat dense matrix $A$ in row-major order, extract the non-zeros, and populate the three CSR arrays (`values`, `col_indices`, `row_ptrs`).
2. **The Computation Phase:** Calculate the dot product $y = Ax$ by iterating over the rows using the strict boundaries defined by `row_ptrs`, bypassing the zeros entirely.

## 4. Solving the Issue

I started by trying to write out the logic using pure pointer arithmetic to really understand the memory addresses.

### Mistake 1: Flattened Matrix Navigation

In my first attempt at the compression phase, I tried to navigate the flat 1D array as a 2D grid using this math: `*(A + i + j)`.

**The Fix:** I quickly realized this is mathematically flawed. Adding the row index and column index directly together means row 1, column 0 (`1 + 0 = 1`) points to the exact same memory address as row 0, column 1 (`0 + 1 = 1`). To correctly find the "step count" from the start of a flattened matrix, I needed to multiply the row number by the total columns: `*(A + i * cols + j)`.

### Mistake 2: Pointer Precedence

When I found a non-zero, I tried to increment my total count using `*out_nnz++`.

**The Fix:** In C, the strict order of operations means the `++` binds to the pointer's address first, not the integer inside it! Instead of adding 1 to my count, I was shifting the pointer to look at random memory next door. I had to wrap it in parentheses: `(*out_nnz)++`.

### Mistake 3: The Empty Row Bug

My logic for updating the `row_ptrs` array was initially trapped inside my `if` statement that checked for non-zeros.

**The Fix:** The CSR format requires that `row_ptrs` records the starting position of _every single row_, even if the row is entirely zeros. Because my logic only triggered when it found a number, completely empty rows were skipped, breaking the compression. I fixed this by moving the assignment `row_ptrs[i] = *out_nnz;` to the very top of the outer loop, right before scanning the columns. This elegantly handles empty rows by giving them identical start and stop boundaries, causing the multiplication phase to safely skip them.

### Mistake 4: The Multiplication Boundary

For the second phase, I set my inner loop boundaries like this:
`int row_end = row_ptrs[i+1] - 1;` and ran the loop while `j < row_end`.

**The Fix:** This was a classic off-by-one error. If a row had exactly one element (start index 0, next row start index 1), `row_end` became `0`. Because `0` is not strictly less than `0`, the loop instantly terminated, skipping the final element of every single row. I fixed this by dropping the `- 1` and using the standard C `<` boundary approach.

## 5. Polishing & Linux Kernel Style

After getting the logic completely watertight using raw pointer arithmetic (like `*(values + *out_nnz)`), I learned that systems programming and Linux Kernel Coding Style universally prefer the array bracket shortcut for readability.

I refactored the code to use clean `array[index]` syntax, padded my binary operators, and dropped the function's opening brace to a new line.

Here is the final, polished core logic for the computation phase:

```c
    for (int i = 0; i < rows; i++) {
        double dot_p = 0;

        for (int j = row_ptrs[i]; j < row_ptrs[i + 1]; j++) {
            dot_p += values[j] * x[col_indices[j]];
        }

        y[i] = dot_p;
    }
```

## 6. Testing

I ran my implementation against the provided test harness. The harness generates random matrices of varying sparsity (densities from 5% to 40%), executes the standard dense matrix multiplication to get a reference array, and compares it against my CSR implementation using a mixed absolute/relative tolerance `1e-7`.

```bash
gcc -lm -o run challenge.c
./run
```

**Result:** `All tests passed! (100/100 iterations passed)`

## 7. Reflection

This was an incredibly cool challenge. The beauty of the CSR format is that the dot product for a row becomes an incredibly tight, cache-friendly loop. Instead of doing 1,000 multiplications for a 1,000-column row, if there are only 3 non-zeros, the CPU only does 3 operations.

Building this from scratch gave me a huge appreciation for what libraries like OpenBLAS do under the hood, and makes me incredibly excited for the potential to apply this to RISC-V hardware acceleration.
