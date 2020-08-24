# Building the 2-D convolutional Kernel and Host Application
The lab will focus on building Vitis kernel using Xilinx provided platform for U200 card. A host side application will be designed that will coordinate all the data movements and execution triggers for invoking the kernel. During this lab real performance measurements will be taken and compared to estimated performance and the CPU only performance.

## Host Application
This section discusses how host application is written to orchestrate the execution of convolutional kernel. As it was estimated in previous lab that to meet the performance constraints of 1080p HD Video multiple compute unit will be needed. The host application is designed to be agnostic to number of compute units. More specifically if the compute units are symmetric ( instance of same kernel and memory connectivity to device DDR is same) the host application can deal with any number of these compute unit. Other tutorials about [host programming and Vitis](https://github.com/Xilinx/Vitis-Tutorials) are also available.

### Host Application Variants
Please go to top level folder for the convolution tutorial and change directory to "src" and do listing of files:
```bash
cd src
ls
```
There are two files namely "**host.cpp**" and "**host_randomized.cpp**". They can be used to build two different version of host application. The way they interact with the kernel compute unit is exactly same except that one uses **pgm** image file as input that is repeated multiple times to emulate an image sequence(video) and the other one uses randomly generated image sequence data. The host with random input image generation has no dependencies whereas the host code in "**host.cpp**" uses OpenCV libraries, specifically it uses **OpenCV 2.4** libraries to load, unload and convert between raw formats.

### Host Application Details
Host application starts by parsing command line arguments following are the command line options provided by host that takes input image and uses OpenCV(in source file "src/host.cpp"):

```cpp
  CmdLineParser parser;
  parser.addSwitch("--nruns",   "-n", "Number of times the image is processed", "1");
  parser.addSwitch("--fpga",    "-x", "FPGA binary (xclbin) file to use");
  parser.addSwitch("--input",   "-i", "Input image file");
  parser.addSwitch("--filter",  "-f", "Filter type (0-6)", "0");
  parser.addSwitch("--maxreqs", "-r", "Maximum number of outstanding requests", "3");
  parser.addSwitch("--compare", "-c", "Compare FPGA and SW performance", "false", true);
```
The other host file(in source file "host_randomized.cpp") that uses randomized input data for as images provides the following command line options:
```cpp
  CmdLineParser parser;
  parser.addSwitch("--nruns",   "-n", "Number of times to image is processed", "1");
  parser.addSwitch("--fpga",    "-x", "FPGA binary (xclbin) file to use");
  parser.addSwitch("--width",   "-w", "Image width",  "1920");
  parser.addSwitch("--height",  "-h", "Image height", "1080");
  parser.addSwitch("--filter",  "-f", "Filter type (0-6)", "0");
  parser.addSwitch("--maxreqs", "-r", "Maximum number of outstanding requests", "3");
  parser.addSwitch("--compare", "-c", "Compare FPGA and SW performance", "false", true);
```
different options can be used to launch the application and performance measurement. In this lab we will set most of this command line inputs to application using a makefile. In the top level directory a file namely **make_options.mk** is provided that allows to set most of these options. This section elaborate more on application structure and host application and compute unit interactions and how it is modeled.
The next thing host application does is to create OpenCL context, read and load xclbin and create a command queue with out of order execution and profiling enabled. After OpenCL setup memory allocation is done and input image is read(or randomly generated). Once the setup is complete "Filter2DDispatcher" object is created and used to dispatch filtering request on number of images. Timer are used to take execution time measurement for both software and hardware execution. Finally host application prints the summary of results. Most of the heavy lifting is done by "**Filter2DDispatcher**" and **Filter2DRequest** classes which manage and coordinate the execution of filtering operations on multiple compute units. Both of the host version are based on these classes. Next sections elaborates how these class abstract these details.

### 2D Filtering Requests
Both the version of host application contains class named "**Filter2DRequest**", which is used by filtering request dispatcher class discussed in next section. An objects of this class essentially allocates and holds handles to OpenCL resources needed for enqueueing 2-D convolution filtering request. An object of this class encapsulates single request that is enough to process single color component for a given image. These resources include OpenCl Buffers, Event Lists and Handles to Kernel and Command queue. The application creates a single command queue that is passed down to enqueue every kernel enqueue command. Once an object of Filter2DRequest class is created, it can be used to make an API call namely "**Filter2D**" that will enqueue all the operations such as moving input data, filter co-efficients, kernel call and reading of output data back to host. The same API call will create a list of dependencies between these transfers and create an output event that signals the completion of output data transfer to host.
### 2D Filter Dispatcher
The "**Filter2DDispatcher**" class is the top level class that provides an end-user API to schedule Kernel calls. Every call schedules a kernel enqueue and related data transfers using Filter2DRequest object as explained in previous section. This is a container class that essentially holds a vector of requests objects. The number of **Filter2DRequest** object that are instantiated is defined as "max" parameter for dispatcher class at construction time. The minimum values this parameter can be as small as number of compute unit to allow at-least one kernel enqueue call per compute unit to happen in parallel. But a larger values is desired since it will allow overlap between input and output data transfers happening between host and device. It wont produce any overlap between compute for different images or image color channels larger than total number of compute unit built in the xclbin.

### Building Host Application
The host application can be built using the makefile that is provided with the tutorial. As mentioned earlier host application has two versions, one takes input images to process other one can generate random data that will be processed as images. Makefile is includes a file called "**make_option.mk**" this file provides most of the options that can be used as knobs to generate different host builds and kernel versions for emulation modes. It also provides a way to launch emulation with specific number of test images. The details of options provided by this file are as follows:

- TARGET: selects build target choices are hw,sw_emu,hw_emu
- PLATFORM: Xilinx platform used for build  
- ENABLE_STALL_TRACE : instrument kernel to generate stall info choice are yes,no
- TRACE_DDR: select memory bank for trace storage choices DDR[0-3] etc.
- FILTER_TYPE: selects between 6 different filter type choice are 0-5
- PARALLEL_ENQ_REQS: application command line argument for parallel enqueued requests
- NUM_IMAGES: Number of images to process
- IMAGE_WIDTH: image width to use
- IMAGE_HEIGHT: image height to use 
- INPUT_TYPE: select between host versions
- INPUT_IMAGE: path and name of image file
- USE_PRE_BUILT_XCLBIN: enable use of pre-built fpga binary file
- ENABLE_PROF: Enable OpenCl profiling for host application 
- PROFILE_ALL_IMAGES: while comparing cpu vs. fpga use all images or not
- NUM_IMAGES_SW_EMU: sets no. of images to use for sw_emu
- NUM_IMAGES_HW_EMU: sets no. of images to use for hw_emu
- OPENCV_INCLUDE: OpenCV include folder path
- OPENCV_LIB: OpenCV lib folder path
- KERNEL_CONFIG_FILE: kernel config file
- VPP_TEMP_DIRS: log dir for vpp
- VPP_LOG_DIRS: log dir for vpp

To build host application with randomized data please follow these steps:

```bash
cd "to the top level tutorial diretory"
vim make_options.mk
```

Once the **make_options.mk** is opened make sure **INPUT_TYPE** is set as "random". This selection will make sure that the host that uses random image is built.
```makefile
############## Host Application Options
INPUT_TYPE :=random
```

If it is required to built host that uses input image please set the following two variables that point to **OpenCV 2.4 ** install path in make_options.mk file:

```makefile
############## OpenCV Installation Paths
OPENCV_INCLUDE :=/**OpenCV.24 User Install Path**/include
OPENCV_LIB :=/**OpenCV.24 User Install Path**/lib
```
Before application can be built it is required to source the user install specific scripts for setting up the Xilinx Run Time Library and Vitis Library paths.
```bash
source /**User XRT Install Path**/setup.sh
source /**User Vitis Install Path**/settings64.sh
```
After setting the appropriate paths host application can be built using the makefile command as follows:
```bash
make compile_host
```
It will build host.exe inside a build folder.
## Running Software Emulation
To build and run the kernel in software emulation please proceed as follows:
Open "make_options.mk" and make sure that target is set to sw_emu:
```bash
TARGET ?=sw_emu
```
after setting the target launch the se_emu as follows:
```bash
make run
```
Once the emulation finishes you should get a console output similar to the one below, the output given below is for random input image case:
```bash
----------------------------------------------------------------------------

Xilinx 2D Filter Example Application (Randomized Input Version)

FPGA binary       : ./fpgabinary.sw_emu.xclbin
Number of runs    : 1
Image width       : 1920
Image height      : 1080
Filter type       : 5
Max requests      : 12
Compare perf.     : 1

Programming FPGA device
Generating a random 1920x1080 input image
Running FPGA accelerator on 1 images
  finished Filter2DRequest
  finished Filter2DRequest
  finished Filter2DRequest
Running Software version
Comparing results

Test PASSED: Output matches reference
----------------------------------------------------------------------------

```
If input image is to be used for processing please set OpenCv paths and INPUT_TYPE as empty string in make_options.mk file and run it again. Following is the expected console output:
```bash
----------------------------------------------------------------------------

Xilinx 2D Filter Example Application

FPGA binary       : ./fpgabinary.sw_emu.xclbin
Input image       : ../test_images/picadilly_1080p.bmp
Number of runs    : 1
Filter type       : 3
Max requests      : 12
Compare perf.     : 1

Programming FPGA device
Running FPGA accelerator on 1 images
  finished Filter2DRequest
  finished Filter2DRequest
  finished Filter2DRequest
Running Software version
Comparing results

Test PASSED: Output matches reference
----------------------------------------------------------------------------
```
The input, out and reference software output are shown below:

![](./images/inputImage50.jpg)
![](./images/outputImage.jpg)
![](./images/refImage.jpg)

## Running Hardware Emulation
The application can be run in hardware emulation mode in a similar way as software emulation the only change needed is in "make_options.mk", please set target as follows:
``bash
TARGET=hw_emu
``

**NOTE**: *_Hardware Emulation may take long time, makefile default setting will make sure it simulates only single image but it is recommended in case of random input image that image size be set smaller by keeping image height in range of 30-100 pixels. Height and width of the image can be specified using "make_options.mk" file in host options_*  

Launch emulation using the following command:
```bash
make run
```
It will build hardware kernel in emulation mode and then launch host application. The output printed in console window will be similar to the sw_emu case. But after hw_emu you can analyze different synthesis reports and using Vitis_analyzer view different waveforms. For more details please refer to other [Vitis Tutorials](https://github.com/Xilinx/Vitis-Tutorials)

## Building Kernel Xclbin
Once the kernel functionality is verified and its resource usage is satisfactory (as single module as synthesized in previous lab) kernel build process can be started. The kernel build process will create an xclbin file which is FPGA executable file which can be read and loaded by host to FPGA card. Since building xclbin take few hours and to avoid such a delay in this lab a builtin xclbin file is provided in folder called "xclbin". During hardware run by default builtin xclbin file will be used. But if it is required to build xclbin please set the following options in make_options.mk file as shown below:

```bash
USE_PRE_BUILT_XCLBIN := 0
```
and launch build and run as follows:
```bash
make build
```
## System Run
In this section we will run the host application using FPGA hardware and analyze the performance of overall system using vitis_analyzer and application console log. If you are using builtin xclbin file please make sure **USE_PRE_BUILT_XCLBIN := 1 ** is set to "1", otherwise build the xclbin as described in last step and keep **USE_PRE_BUILT_XCLBIN := 0** Also make sure that in make_options.mk the TARGET is set to "hw". You can also enable the performance comparison with CPU by setting "ENABLE_PROF?=yes" .

 ```bash
TARGET ?=hw
.
.
.
ENABLE_PROF?=yes
.
.
.
USE_PRE_BUILT_XCLBIN :=0 
```

 To run the application please proceed as follows:
```bash
make run
```
It should produce a console log similar to the one show below:
```bsah
----------------------------------------------------------------------------

Xilinx 2D Filter Example Application (Randomized Input Version)

FPGA binary       : ../xclbin/fpgabinary.hw.xclbin
Number of runs    : 60
Image width       : 1920
Image height      : 1080
Filter type       : 3
Max requests      : 12
Compare perf.     : 1

Programming FPGA device
Generating a random 1920x1080 input image
Running FPGA accelerator on 60 images
Running Software version
Comparing results

Test PASSED: Output matches reference

FPGA Time         :     0.4240 s
FPGA Throughput   :   839.4765 MB/s
CPU  Time         :    28.9083 s
CPU  Throughput   :    12.3133 MB/s
FPGA Speedup      :    68.1764 x
----------------------------------------------------------------------------

```

From the console output it should be clear that acceleration achieved when compared to CPU is 68x. The achieved throughput is 839 MB/s which is close to the estimated throughput of 900 MB/s.

