# CUDA keywords

There're three frequently used CUDA keywords:
1. `__global__`
   1. The `__global__` keyword is used to indicate a CUDA kernel function that can be called from the host (CPU) but runs on the device (GPU).
2. `__device__`
   1. The `__device__` keyword is used to indicate that a function or variable is meant to be used on the GPU.
3. `__host__`
   1. The `__host__` keyword is used to indicate that a function or variable is meant to be used on the CPU.

Among them, the CUDA kernel function deserves further explanation. Let's dive into the details.

## CUDA Kernel Function

CUDA kernel function is a special type of function that is executed on the GPU (Graphics Processing Unit) as part of CUDA programming. Kernel functions allow for parallel execution of code across many threads, enabling high-performance computations.

For example:

```cpp
__global__ void myKernel() {
    int threadIdX = threadIdx.x; // Thread index in x dimension
    int threadIdY = threadIdx.y; // Thread index in y dimension
    int blockIdX = blockIdx.x;   // Block index in x dimension
    int blockIdY = blockIdx.y;   // Block index in y dimension

    // Calculate global indices
    int globalIndexX = blockIdX * blockDim.x + threadIdX;
    int globalIndexY = blockIdY * blockDim.y + threadIdY;

    // Do something with globalIndexX and globalIndexY
}

int main() {
    dim3 gridSize(2, 2);     // 2 blocks in x and y dimensions
    dim3 blockSize(4, 4);    // 4 threads per block in x and y dimensions

    myKernel<<<gridSize, blockSize>>>();

    // Wait for GPU to finish
    cudaDeviceSynchronize();

    return 0;
}
```

You may have noticed these variables: `blockSize`/`gridSize`/`blockDim`/`blockIdx`, let's dive into the details.

### `blockSize`

1. Definition
   1. `blockSize` determines the number of threads in a single block. It is specified as a `dim3` object, allowing you to define the size in one, two, or three dimensions.
2. Thread Configuration
   1. When launching a kernel, you specify how many threads should be grouped together in a block. 
   2. For example:
    ```cpp
    dim3 blockSize(16, 16); // Creates a block with 16x16 = 256 threads
    ```
3. Limits
   1. Each block has a maximum number of threads it can contain, which is hardware-dependent (commonly 1024 threads for many GPUs). The size of the block should be chosen based on the problem size and the GPU architecture.
4. Shared Memory
   1. Threads within the same block can share data through shared memory, making it suitable for situations where threads need to cooperate closely.

### `gridSize`

1. Definition
   1. `gridSize` specifies the number of blocks that will be launched in the grid. Like blockSize, it is also specified as a `dim3` object.
2. Grid Configuration
   1. You can define the grid size in one, two, or three dimensions, allowing you to map complex data structures (like matrices) to the GPU.
   2. For example:
    ```cpp
    dim3 gridSize(4, 4); // Creates a grid with 4x4 = 16 blocks
    ```
3. Total Threads
   1. The total number of threads launched when executing a kernel is the product of the number of blocks and the number of threads per block.
   2. For example, if you have a grid of size 4x4 and a block size of 16x16, the total number of threads would be:
    ```cpp
    int totalThreads = gridSize.x×gridSize.y×blockSize.x×blockSize.y = 4×4×16×16 = 1024
    ```

### Relationship between `blockDim` and `blockSize`

1. Definition
   1. `blockDim`: This is a built-in variable that exists within the kernel function. It provides the dimensions (number of threads) of the block currently being executed. It is of type `dim3`, which includes `blockDim.x`, `blockDim.y`, and `blockDim.z`.
   2. `blockSize`: This is the parameter you specify when launching the kernel, indicating how many threads should be in each block.
2. Correspondence
   1. When you launch a kernel, the blockSize you provide directly determines the values of blockDim for that block.
   2. For example, if you launch a kernel with:
    ```cpp
    dim3 blockSize(16, 16);
    myKernel<<<gridSize, blockSize>>>();
    ```
    then within myKernel, `blockDim.x` will be 16 and `blockDim.y` will also be 16.

### Relationship between `blockIdx` and `gridSize`

1. Definition
   1. `blockIdx`: This is a built-in variable that exists within the kernel function. It provides the index of the block currently being executed in the grid. It is also of type `dim3`, which includes `blockIdx.x`, `blockIdx.y`, and `blockIdx.z`.
   2. `gridSize`: This is the parameter you specify when launching a kernel, indicating how many blocks should be created in the grid.
2. Correspondence
   1. While `gridSize` defines the total number of blocks in the grid, `blockIdx` indicates the index of the specific block that is currently executing. For example, if you launch a kernel with a grid size of 4x4 (16 blocks total), `blockIdx.x` will take values from 0 to 3 in the x dimension for each block in that dimension.


