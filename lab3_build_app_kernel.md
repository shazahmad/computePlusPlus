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

## Building 2-D Convolutional Kernel
As per our performance estimation in previous lab and given performance constraints, the kernel will be built with clock frequency constraint of 300 MHz

Lab 3: ( Hardware software integration xclbin and first version of host)

    Create host code for accelerator
    Run hardware and software emulation
    Building xclbin
    Conclude tutorial with Single Cu per RGB channel
    Discuss timing waveform and perform other analysis deemed fit
    Show the performance with respect to software
    Highlight potential gaps/under utilized resources that can be used to increase acceleration
