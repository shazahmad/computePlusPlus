﻿<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2020.1 Vitis™ Application Acceleration Development Flow Tutorials</h1>
   <a href="https://github.com/Xilinx/Vitis-Tutorials/branches/all">See 2019.2 Vitis Application Acceleration Development Flow Tutorials</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h1>Accelerating Video Convolution Filtering Application
 </td>
 </tr>
</table>

# Experience the Acceleration
The first lab is designed to be very simple and straight forward. The objective of the labs is let the user experience what acceleration performance can be achieved by porting the video filter to an FPGA card. The card being used is from Xilinx's Alveo series which are designed for accelerating data center applications. But in general this tutorial can be adapted to other FPGA cards with some simple changes.
 The steps to be carried out for this lab include:
- Setting up Vitis acceleration environment
- Running hardware optimized accelerator and comparing its performance with baseline

The lab simply demonstrates the huge performance gain that can be achieved as compared to CPU performance. Whereas next labs in this tutorial will guide through how such performance can be achieved using different optimizations and design techniques for 2D convolutional kernel and the host side application.
## Cloning Repo and Vitis setup
Clone the repository using following command:
```commandline
git clone https://github.com/Xilinx/Vitis-Tutorials.git
```
Setup application build and runtime environment using the following commands as per your local installation:

```commandline
source XILINX_VITIS_INSTALL_PATH/settings64.sh
source XRT_INSTALL_PATH/setup.sh
```


## Baseline Application Performance
The software application is design to process HD video resolution images(1920x1080). It performs convolution on a set of images and prints the summary of performance results. It is used for measuring baseline software performance. Run the application to measure performance as follows:

```commandline
cd REPO_PATH/docs/video-filter/sw_run
./run.sh 
```
Results similar to the following will be printed out, note down the CPU throughput.

```commandline
----------------------------------------------------------------------------
Number of runs    : 60
Image width       : 1920
Image height      : 1080
Filter type       : 6

Generating a random 1920x1080 input image
Running Software version on 60 images

CPU  Time         :    28.0035 s
CPU  Throughput   :    12.7112 MB/s
----------------------------------------------------------------------------
```

## Running FPGA Accelerated Application
### Building the Host Application
Run the following script to build host application. The binary file for FPGA(xclbin) takes considerable time for building so it is provided prebuilt, in next lab we will also provide commands to build.

### Launching the Host Application
Now launch the application to run using FPGA accelerated video convolution filter. 

```commandline
cd REPO_PATH/docs/video-filter/
make run
```
The result summary similar to the one given below will be printed.
```commandline
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

## Results
 From the console output coming out of the host application it is clear that FPGA accelerated kernel can out perform CPU only implementation by 68x. It is big gain in terms of performance over CPU and in next labs we will see it allows us to process more than 3 HD video channles with 1080p resolution. Given this big gain in perfoamcne next labs will be dedicated in describing how it was made possible by building a kernel modeled in software(c++) as well as how to write a host application, which can effectively unleash the compute power of this custom built hardware kernel.


---------------------------------------

<p align="center"><b>
Next Lab Module: <a href="./lab1_app_introduction_performance_estimation.md">Video Convolution Filter : Introduction and Performance Estimation</a>
<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
</b></p>
 