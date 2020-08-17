# Design and Analysis of Hardware Kernel Module for 2-D Video Convolution Filter
The focus of this lab will be to illustrate the design of convolutional filter module, analyze its performance and hardware resource utilization. We are following a bottom up approach here by first designing the hardware block and analyzing its performance before integrating the whole system for accelerating the application. We will use Vitis HLS to build and estimate the performance.
## 2-D Convolution Filter Implementation
This section discuses the design of convolution filter in detail. It goes through top level structure, optimizations and implementation details.
### Top Level Structure of Kernel
The top level of convolution filter is modeled dataflow process consisting of four different functions as given below, for full implementation details you can refer to source file that is in **"src/filter2d_hw.cpp"**.

```cpp
void Filter2DKernel(
        const char           coeffs[256],
        float                factor,
        short                bias,
        unsigned short       width,
        unsigned short       height,
        unsigned short       stride,
        const unsigned char  src[MAX_IMAGE_WIDTH*MAX_IMAGE_HEIGHT],
        unsigned char        dst[MAX_IMAGE_WIDTH*MAX_IMAGE_HEIGHT])
  {
            
#pragma HLS DATAFLOW

	// Stream of pixels from kernel input to filter, and from filter to output
    hls::stream<char,2>    coefs_stream;
    hls::stream<U8,2>      pixel_stream;
    hls::stream<window,3>  window_stream; // Set FIFO depth to 0 to minimize resources
    hls::stream<U8,64>     output_stream;

	// Read image data from global memory over AXI4 MM, and stream pixels out
    ReadFromMem(width, height, stride, coeffs, coefs_stream, src, pixel_stream);

    // Read incoming pixels and form valid HxV windows
    Window2D(width, height, pixel_stream, window_stream);

	// Process incoming stream of pixels, and stream pixels out
	Filter2D(width, height, factor, bias, coefs_stream, window_stream, output_stream);

	// Write incoming stream of pixels and write them to global memory over AXI4 MM
	WriteToMem(width, height, stride, output_stream, dst);

  }

```
Dataflow chain consists of four different functions as follows:

- **ReadFromMem**: reads pixel data or video input from main memory
- **Window2D**:  local cache with wide(15x15 pixels) access on output side
- **Filter2D**:  core kennel filter algorithm
- **WriteToMem**:  writes output data to main memory

Two function at the input and output are very typical modules that read and write data from the memory connecting it with the accelerator. The data read from the main memory is passed to Window2D function which creates a local cache and on every cycle provided a 15x15 pixel sample to filter function/block which can consume it in single cycle to perform @check 225(15x15) MACs per cycle. Please open and have a look at the implementation details of these individual functions. In next section we will elaborate on the implementation details of Window2D and Filter2D functions. 
      ![](images/filterBlkDia.jpg)

###Data Mover
One of the key advantage that comes with the design custom hardware accelerated for which FPGAs are well suited for many reasons is the choice and architecture of custom data movement schemes. Specialized blocks at the interface with main memory can be built at the input and output of the data processing engine or processing elements. Convolutional filter is a very good example of this. Looking from pure software implementation point of view at the source code it seems that to produce single sample at the output side it requires 450 memory accesses and 1 access to write to output.
```bash
Memory Accesses to Read filter Co-efficients = 15x15 = 225
Memory Accesses to Read Neighbouring Pixels  = 15x15 = 225
Memory Acceeses to Write to Output           = 1
Total Memory Acccesses                       = 451 
```  

Even though many of these accesses can become fast because of caching but still large number of memory accesses will be a performance bottleneck. But while designing on FPGA clever data movement and access schemes can be built easily. One of the key and major advantage is availability of huge on-chip memory bandwidth( distributed and block memory) and choice of custom configuration of this bandwidth. In the next section we will elaborate on the design of  "Window2D" block which is essentially an input side data mover facing kernel processing engine. 
#### Window2D : Line and Window Buffers
The Windows2D block is essentially built from two basic blocks, one is a line buffer that can buffer multiple lines of full image and specifically here it is designed to buffer (Filter Height(FILTER_V_SIZE in code) - 1) full image lines. The total number of pixels held by line buffer are **(FILTER_V_SIZE-1) *  MAX_IMAGE_WIDTH**. The other blocks is "Window" which hold **FILTER_V_SIZE * FILTER_H_SIZE** pixels. The 2-D convolution filtering operation consists of centering the filtering mask(filter co-efficients) on the index of output pixel and calculating the sum of product(SOP) as described in the previous lab also. Following figure shows how this centering and SOP is carried. 
    ![](images/convolution.jpg)     
 The figure above shows SOP carried out with reference to full image being processed but if we look carefully when output pixels are produced line by line, it is not required to have all the image pixels in memory but only the line where filtering mask overlaps are required which is essentially (FILTER_V_SIZE) which even can be reduced to FILTER_V_SIZE-1. Essentially this the data that needs to be on-chip or housed by data mover at any given time.
 ![](images/Window2D.jpg) 
 
 
 
        
Lab 2:  ( Using bottom up flow design a kernel, estimate its performance)

    Synthesizing software like kernel vs. Optimized kernel
    Coding the kernel
    Post synthesis analysis and other vitis_hls features that can be useful for bottom up flow
    >>Simulating the kernel if possible 
    Discuss harware optimization applicable to kernel.
    Compare kernel based on line-buffer vs. simple buffer
