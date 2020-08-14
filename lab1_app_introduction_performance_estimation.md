# Video Convolution Filter : Introduction and Performance Estimation
In this lab we will have a look at 2D video convolution filter measure its performance on host machine. These measurement will establish a performance baseline. Based on the required performance constraints and software performance we can calculate what acceleration a hardware implementation should provide. We will also estimate the performance of FPGA accelerator that will be implemented in latter labs. In nutshell during this lab you will:
- Learn about video convolution filter
- Measure the performance of host based software implementation
- Calculate required acceleration vs. software implementation
- Estimate the performance of hardware accelerator

## Video Filtering Applications and 2-D Convolution Filters
Video applications use different type of filters extensively for multiple reasons such as to filter noise,  manipulate motion blur, color and contrast enhancements, edge detection, creative effects etc. The essential purpose of video filter is to carry out some form of data average around a pixel that effects the amount and type of correlation any pixel has to its surrounding  area. Such filtering is carried out for all the pixels in a video frame. Convolution filters are defined by a matrix of co-efficients. Convolution operation is essentially a sum of products carried on a pixel set( a frame/image sub-matrix centered around a give pixel) and co-efficient matrix. The figure below shows how convolution is calculated for a particular pixel highlighted in yellow. Here the filter has a co-efficient matrix that is 3x3 in size. The figure also displays how output images is generated during filtering process. The index of output pixel being generated is the index of input pixel highlighted in yellow that is being currently filtered. In algorithmic terms the process of filtering consists of:
- Selecting an input pixel as highlighted in yellow in the figure below
- Extracting a sub-matrix whose size is same as filter coefficients
- Calculating sum of product of extracted sub-matrix and co-efficients matrix
- Placing the sum of product as output pixel in output image/frame
      ![](images/convolution.jpg)

## Performance Requirements for 1080p HD Video
We can easily calculate the application performance requirements given the standard performance specs for 1080p High Definition (HD) video. These top level requirements can be then translated into constraints for hardware implementation or software throughput requirements. For 1080p HD video at 60 frames per seconds(FPS) the specs are listed below as well as required throughput in terms of pixels per seconds is calculated:
```bash
Video Resolution        = 1920 x 1080
Frame Width (pixels)    = 1920 
Frame Height (pixels)   = 1080 
Frame Rate(FPS)         = 60 
Pixel Depth(Bits)       = 8 
Channels(YUV)           = 3 
Throughput(Pixel/s)   = Frame Width * Frame Height * Channels * FPS
Throughput(Pixel/s)   = 8*1920*1080*3*60
Throughput (MB/s)     = 356
``` 

The required throughput to meet 60 FPS performance turns out to be 373 MB/sec ( since each pixel is 8-bits).

## Software Implementation and Performance Estimation
This section will discuss the baseline software implementation and performance measurements which will be used to gauge the acceleration requirements given the performance constraints.
### Software Implementation
The convolutional filter is implemented in software using typical multi-level nesting loop structure. Outer two loops define the pixel to be processed and inner two loops perform the sum of product(SOP) between co-efficient matrix and the selected sub-matrix from the image centered around the pixel being processed. Boundary conditions where it is not possible to center sub-matrix around given pixel requires special processing in our case we have assumed all pixel beyond boundary of the image have zero values.

```cpp
void Filter2D(
		const char           coeffs[FILTER_V_SIZE][FILTER_H_SIZE],
		float		         factor,
		short                bias,
		unsigned short       width,
		unsigned short       height,
		unsigned short       stride,
		const unsigned char *src,
		unsigned char       *dst)
{
    for(int y=0; y<height; ++y)
    {
        for(int x=0; x<width; ++x)
        {
        	// Apply 2D filter to the pixel window
			int sum = 0;
			for(int row=0; row<FILTER_V_SIZE; row++)
			{
				for(int col=0; col<FILTER_H_SIZE; col++)
				{
					unsigned char pixel;
					int xoffset = (x+col-(FILTER_H_SIZE/2));
					int yoffset = (y+row-(FILTER_V_SIZE/2));
					// Deal with boundary conditions : clamp pixels to 0 when outside of image 
					if ( (xoffset<0) || (xoffset>=width) || (yoffset<0) || (yoffset>=height) ) {
						pixel = 0;
					} else {
						pixel = src[yoffset*stride+xoffset];
					}
					sum += pixel*coeffs[row][col];
				}
			}
			
        	// Normalize and saturate result
			unsigned char outpix = MIN(MAX((int(factor * sum)+bias), 0), 255);

			// Write output
           	dst[y*stride+x] = outpix;
        }
    }
}
```
The following snapshot shows how top-level calls core convolution filter function for an image with three different components or channels. Here OpenMP pragma is used to parallelize software execution using multiple threads. You can open **_src/host_randomized.cpp_** and **_src/filter2d_sw.cpp_** to have look all implementation details.

```cpp
   #pragma omp parallel for num_threads(3)
  for(int n=0; n<numRunsSW; n++) 
  {
    // Compute reference results
    Filter2D(filterCoeffs[filterType], factor, bias, width, height, stride, y_src, y_ref);
    Filter2D(filterCoeffs[filterType], factor, bias, width, height, stride, u_src, u_ref);
    Filter2D(filterCoeffs[filterType], factor, bias, width, height, stride, v_src, v_ref);
  }
```
### Running the Software Application
To run the software application go to directory called "sw_run" and launch the application as follows:

```bash
cd sw_run
./run.sh
```
Once the application is launched it should produce an output similar to the one show below. The software application will process a randomly generated set of images and report performance. Here we have used randomly generated images to avoid any extra library dependencies such as OpenCV. But in next labs, while working with the hardware implementation the option to use OpenCV library for loading images or use random generated images will be provided given the user has OpenCV 2.4 installed on the machine. If other version of OpenCV is needed user can modify the host application to use different APIs for loading and storing images from the disk.
```bash
----------------------------------------------------------------------------
Number of runs    : 60
Image width       : 1920
Image height      : 1080
Filter type       : 6

Generating a random 1920x1080 input image
Running Software version on 60 images

CPU  Time         :    24.4447 s
CPU  Throughput   :    14.5617 MB/s
----------------------------------------------------------------------------
```
The application run measures performance using high precision timers and reports it as throughput. The machine which was used for experiments produced a throughput of 14.51 MB/s. The details of machine used are given below:

```bash
    CPU Model : Intel(R) Xeon(R) CPU E5-1650 v2 @ 3.50GHz
    RAM       : 64 GB
```
The measured performance is **_"2.34 FPS"_** only whereas the required throughput is **60 FPS**. The acceleration needed to meet the required performance of 60 FPS:
```bash
   Acceleration Factor = Throughput ( Required) / Throughput(Sw only)
   Acceleration Factor = 356/14.56 = 24.44X
```
So to meet required performance it is required that software implementation be accelerated by almost 25x. 

## Hardware Implementation
To understand what kind of hardware implementation is needed given the performance constraints, lets analyze the convolutional kernel. The core compute is done in a 4-level nested loop but we can break the compute per output pixel produced. In terms of output pixels produced it is clear from the filter source code that single output pixel is produced when inner two loops finish execution once. The inner two loops are essentially doing sum of product on co-efficient matrix and image sub-matrix. Here the matrix sizes are defined by co-efficient matrix and in this case it is chosen to be 15x15. So inner two loop are essentially performing a dot product of size 225.
### Baseline Hardware Implementation Performance
The most simplest and straight forward implementation in hardware can be achieved by passing this kernel source code as it is through the Vitis HLS tool, it will pipeline the inner most loop with II=1 hence computing only one multiply accumulate(MAC). The performance can be estimated based on the MACs as follows:
```bash
 MACs per Cycle = 1
 Hardware Fmax(MHz) = 300
 Throughput  = 300/225 = 1.17 (MPixels/s) =  1.33 MB/s
```
Here the hardware clock frequency is assumed to be 300MHz because in general for Xilinx Alveo Data Center Cards this is the maximum supported clock frequency. The performance turns out to be 1.33 MB/s with baseline hardware implementation. From the convolutional filter source code it can also be estimated how much memory bandwidth is needed at the input and output for achieved throughput. From the convolutional filter source code also shown above it is clear that inner two loops while calculating a single output pixel performs 225(15*15) reads at the input so:
```bash
Output Memory Bandwidth = Throughput = 1.33 MB/s
Input Memory Bandwidth Required = Throughput * 255 = 299.25 MB/s
```   
For the baseline implementation the memory bandwidth requirement are very trivial assuming that PCIe and device DDR memory bandwidths on Xilinx Acceleration Cards/Boards are of the order of 10s of GB/s.
As we have seen in previous sections that throughput required for 60FPS 1080p HD video is 355 MB/s. So it clear that to meet the performance requirement:
```bash
Acceleration Factor to Meet 60FPS Performance = 355/1.33 = 266x
Acceleration Factor to Meet SW Performance    = 14.5/1.33 = 10.9x
```
### Performance Estimation for Optimized Hardware Implementation
From the above calculations it is clear that we need to improve the performance of baseline hardware implementation by 266x to meet the performance. One of the path we can take is to start unrolling the inner loops and pipeline. For example by unrolling the inner most loop which iterated 15 times we can improve the performance by 15x. With that the performance will be better than software only implementation but still we cannot meet required video performance. Another approach that we can follows is to unroll the inner two loops and hence gain in performance by 15*15=225 which means  a throughput  of 1-output pixel per cycle. The performance and memory bandwidth requirements will be as follows:
```bash
    Throughput  = 300/1 = 300 MB/s
    Output Memory Bandwidth =  300 MB/s
    Input Memory Bandwidth =  300 * 225 = 67.5 GB/s
```

The required output memory bandwidth scales linearly with throughput but input memory bandwidth has gone up enormously and might not be sustainable. But a closer look at the convolution filter will reveal that it is not required to read all 225(15x15) pixels from the input memory for processing. A clever caching scheme can be built to avoid such extensive use of input memory bandwidth. The convolutional filter belongs to a class kernels known as stencil kernels which can be optimized to increase input data use extensively. Which can result in substantially reduced memory bandwidth   requirements. Actually with a caching scheme we can bring the input bandwidth requirement to be same as output which is around 300 MB/s. When both inner loops are unrolled it will require that only 1-input pixel is read for producing one output pixel on average.
Given that we can bring down the input bandwidth the achieved performance will be 300 MB/s, which is lower than required 355 MB/s. To deal with this we can look for other ways to increase throughput of hardware or follow a simple approach of duplicating hardware. Said in heterogeneous computing terminology we can increase the number of compute units which can process data in parallel,in case of video we can process color channels separately on separate compute units. Assuming that we will use 3 compute unit one for each color channel, the expected performance summary will be as follows:
```bash
 Throughput(estimated)  =  300 x 3 = 900 MB/s
 Acceleration Against Software Implementation = 900/14.5 = 62x
 Kernel Latecny ( per image on any color channel ) = (1920*1080) / 300 = 6.9 ms
 Video Processing Rate = 144  FPS   
```

Given these performance numbers and architecture selection or implementation next lab will show how we can embark on the design of kernel hardware and finally end up with an accelerated application that provides a performance that is very close to the estimates.

In this lab you learnt about:
- Basics of convolution filter
- Profiled the performance of software only implementation
- Estimated the performance and requirements for hardware implementation
