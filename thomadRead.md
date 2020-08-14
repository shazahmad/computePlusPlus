Filter 2D Example

1. SW only version

Function to be accelerated in ./src/filder2d_sw.cpp
Simple testbench in ./src/host_sw_only.cpp
- Uses randomly generated images instead of real ones to keep the code simple
- Uses OpenMP to parallelize SW execution (computes Y,U and V in parallel threads)

To run: 
  cd ./run/sw_only
  ./run.sh

Runs 60 1902x1080 images

Results on xsjherver50:
  CPU  Time         :    28.0691 s
  CPU  Throughput   :    12.6814 MB/s
This is about 2.15 frames per second


2. Analysis

The 2D Filter produces 1 output pixel from a VxH window of input pixels. This example uses a 15x15 window. So computing 1 output needs reading 225 inputs and computing 225 MACs.

We are looking at an algorithm with a computational intensity of 1 (225 Ops / 1 Memory Write) from the perspective of outputs.

Assuming a 300Mhz clock, which is the maximum on Alveo cards, we can estimate the HW throughput without any parallelization as follows 300/225 = 1.333MB/sec
To meet SW performance, we would need to parallelize by 12.6814/1.333 = 10x
Sustaining 60 frames per second (1080p) would require a throughput of 1920x1080*60 = 124.416MB/sec and therefore a parallelization factor of 124.416/1.333 = 94x
And to build an accelerator sustaining a throughput 300MB/sec (1 output per cycle), we would need to parallelize by a factor of 225x 

Given the profile of this algorithm, parallelizing by 225x would mean performing 225 MACs in parallel, which would require being able to access 225 input values in parallel. 

This may seem like a tall order. But a quick analysis of the memory access patterns shows that if each output pixel needs 225 input values, 224 of these input values are reused from previous computations.

This is generally known as stencil pattern:
https://en.wikipedia.org/wiki/Stencil_code

Such patterns are well suited to FPGAs and are widely documented. A rapid search for "2D filter FPGA" returns a lot results, including: 
https://basile.be/2019/03/18/a-tutorial-on-non-separable-2d-convolutions-in-vivado-hls/
https://www.mdpi.com/2313-433X/4/12/138/htm

By creating a suitable caching structure to reuse previously read pixels, it is possible to compute 1 output pixel by reading a single 1 new input from global memory while retrieving the other pixels in parallel from local memory. These images illustrate this well:
https://www.researchgate.net/profile/Ondrej_Miksik/publication/322872011/figure/fig4/AS:667766524739599@1536219354833/Memory-requirements-for-sliding-window-operations-in-FPGA-accelerators-Line-buffers_W640.jpg
https://www.researchgate.net/profile/Akif_Oezkan/publication/319877225/figure/fig3/AS:540218954006533@1505809643655/A-combination-of-line-buffers-and-memory-windows-is-typically-used-to-process-local_W640.jpg

The ability to create custom data-movers and custom memory hierarchies is a major differentiating feature of FPGAs and is usually the key to achieving acceleration. In this case, creating a custom cache with line buffers and sliding windows will allow accessing 225 values in parallel while doing a single read from global memory. A custom datapath with 225 MACs can be implemented to process these 225 values in parallel and generate 1 output per cycle, thereby achieving a throughput of 300MB/sec.

By instanciating 3 copies of the accelerator, we can process the Y,U and V components of the image in parallel and achieve a maximum theoretical throughput of 900MB/sec and a speed-up factor of 900/12.68 = 70x

Lastly, if we indeed process 1 sample per cycle, then we should be ideally be able to process a 1920x1080 image in 1920x1080@300Mhz=6.912ms (not counting some latency cycles associated with the depth of the pipeline)

(900MB/s and 6.912ms -> mark these numbers!)


3. Coding the Kernel and Iterating in HLS

source code: ./src/filter2d_hw.cpp
testbench: ./src/hls_testbench.cpp

From the earlier analysis we have derived the following requirements for the kernel:
- Throughput goals of 300MB/sec (1 output value per cycle)
- Latency is not a hard requirement but is expected to be about 7ms for a 1920x1080 image
- 1 Kernel will process 1 color component (Y, U or V)
- Each color component is 1 byte (8 bits)
- 3 Kernels will be instanciated to process Y, U and V in parallel
- The 2D filter will process a window of 15x15 parallel inputs. Its input needs to be 1800 bit wide and the ouput 8 bit wide.
- Interfaces will be 512-bit wide to optimize accesses to global memory and reduce utilization of the interconnect, freeing bandwidth for the other CUs

The code structure follows the load/compute/store pattern. 
Implemented as a set of dataflow processes using streaming connections only

readFromMem streams pixels to Window2D
Window2D streams 15x15 parallel values (1800 bits) to Filter2D
Filter2D streams output pixels to writeToMem

We use a throughput oriented coding style for all functions and loops: 1 input and/or output access per loop iteration. Intuitive way of achieving II=1.
The goal for this design is to achieve II=1 for all loops and functions.
Pragmas are added where necessary to achieve this one the throughput oriented coding style is in place.

readFromMem and writeToMem blocks are coded to maximize bursting and enable auto-widening. Kernel operates on 8-bit values, but interface is automatically widened to 512-bits. Writing these blocks with a single loop instead of two nested loops results in tighter bursting. Making the compiler see that the stride is a multiple of 64 allows it to widen the interface from 8-bits to 512-bits.

Window2D block is the special cache structure which reads 1 input pixel and feeds a 15x15 window to the Filter2D block. 
Uses line buffers and sliding windows to achieve II=1. 
Pragmas are needed to properly partion the arrays and allow parallel accesses.
Needs some ramp-up cycles to preload the buffers before valid windows can be continuously generated.
This code will deliver II=1 regardless of the window size (parameters set in common.h). 
The larger the window, the more memory will be needed.

Filter2D almost identical to the original SW version. Main difference: reads an incoming stream of 15x15 windows instead of reading individual pixels from global memory. It is also possible to directly embedded the Window2D code in the Filter2D code, but here we kept it separated for the sake of clarity.
We also added loop at the start of the function to load and locally store the coefficients streamed by the readFromMem block. Each time the kernel is invoked, new coefficients are loaded this way.
The two inner loops of the Filter are automatically unrolled by the compiler (because of the pipeline pragma on the outer loop).
This results in parallelizing the computation, creating a datapath with 225 MACs consuming 225 inputs and producing 1 output every cycle.

Assertions are used to infer loop trip count and get static performance estimates.

To run HLS: 
  cd ./run/hls
  make hls

Notice the Pipeline and Burst messages in the log file. These are results we want to achieve.
  INFO: [HLS 200-1470] Pipelining result : Target II = 1, Final II = 1, Depth = 3.
  INFO: [HLS 200-1470] Pipelining result : Target II = 1, Final II = 1, Depth = 3.
  INFO: [HLS 200-1470] Pipelining result : Target II = 1, Final II = 1, Depth = 3.
  INFO: [HLS 200-1470] Pipelining result : Target II = 1, Final II = 1, Depth = 2.
  INFO: [HLS 200-1470] Pipelining result : Target II = 1, Final II = 1, Depth = 27.
  INFO: [HLS 200-1470] Pipelining result : Target II = 1, Final II = 1, Depth = 3.
  INFO: [HLS 214-115] Burst read of length 4 and bit width 512 has been inferred on port 'gmem' (../../src/filter2d_hw.cpp:21:17)
  INFO: [HLS 214-115] Burst read of variable length and bit width 512 has been inferred on port 'gmem' (../../src/filter2d_hw.cpp:29:17)
  INFO: [HLS 214-115] Burst write of variable length and bit width 512 has been inferred on port 'gmem' (../../src/filter2d_hw.cpp:51:18)

Notice the static performance estimates in the HLS report: 6.912 ms for the Filter2D block and 7.370ms for the entire kernel. These numbers are very close to our theoretical estimates.

more prj/solution/syn/report/Filter2DKernel_csynth.rpt

+ Latency: 
    * Summary: 
    +---------+---------+----------+----------+-----+---------+----------+
    |  Latency (cycles) |  Latency (absolute) |    Interval   | Pipeline |
    |   min   |   max   |    min   |    max   | min |   max   |   Type   |
    +---------+---------+----------+----------+-----+---------+----------+
    |      375|  2211164| 1.250 us | 7.370 ms |  334|  2211165| dataflow |
    +---------+---------+----------+----------+-----+---------+----------+

    + Detail: 
        * Instance: 
        +-----------------+--------------+---------+---------+-----------+----------+-----+---------+---------+
        |                 |              |  Latency (cycles) |  Latency (absolute)  |    Interval   | Pipeline|
        |     Instance    |    Module    |   min   |   max   |    min    |    max   | min |   max   |   Type  |
        +-----------------+--------------+---------+---------+-----------+----------+-----+---------+---------+
        |Filter2D_U0      |Filter2D      |      232|  2073857|  0.773 us | 6.912 ms |  232|  2073857|   none  |
        |ReadFromMem8_U0  |ReadFromMem8  |      333|  2211164|  1.110 us | 7.370 ms |  333|  2211164|   none  |
        |Window2D_U0      |Window2D      |        7|  2087054| 23.331 ns | 6.956 ms |    7|  2087054|   none  |
        |WriteToMem_U0    |WriteToMem    |        5|  2210834| 16.665 ns | 7.369 ms |    5|  2210834|   none  |
        +-----------------+--------------+---------+---------+-----------+----------+-----+---------+---------+


To run cosim and open waveforms:
  make cosim xsim

Notice the design operates at a nearly continuous rate. Except for a few pauses between bursts, the design processes 1 sample per cycle.

Note: I used the new Dataflow analysis and FIFO profiling capabilities to size the FIFOs in this design.


4. Add OpenCL APIs

Host code #1 ./src/host.cpp <- loads image from disk
Host code #2 ./src/host_randomized.cpp <- same but uses randomly generated images

Run SW and HW emulation 
  cd ./run/sw_emu
  ./run.sh <- uses a 1920x1080 image
  cd ./run/hw_emu
  ./run.sh <- uses a small image to keep runtime managerable

5. Build HW
  
  cd ./build/hw
  make xclbin

Notice the build_options.cfg file and how we setup the build with 3 CUs
All the kernels are connected to DDR[1] which is in the static region of the platform.
This design closes timing at 300Mhz with Vitis 20.1

Open Vitis Analyzer to inspect results.

6. Run on HW

  cd ./run/hw
  ./prof.sh

Runs 60 images on FPGA and SW

Results
  FPGA Time         :     0.4252 s
  FPGA Throughput   :   837.1187 MB/s
  CPU  Time         :    27.8645 s
  CPU  Throughput   :    12.7746 MB/s
  FPGA Speedup      :    65.5300 x

Notice how the throughput if 837MB/s, very close to the theoretical max of 900MB/s. (This is 93% of the theoretical max)

Open Vitis Analyzer, look at the profile summary - CU Average time is 6.99ms, extremely close to what we predicted!

Open the Timeline Trace, see how the CUs are invoked back-to-back, resulting in very high efficiency

./run.sh runs 1000 images without profiling or SW comparison (would be too slow!)

Summary: following the Vitis acceleration methodology and doing proper analysis up-front results in quick and predictible results. (Sharpen the axe before chopping down the tree!). 






