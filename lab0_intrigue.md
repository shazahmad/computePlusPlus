# Experience the Acceleration
The first lab is designed to be very simple and straight forward. The objective of the labs is let the user experience what acceleration performance can be achieved by porting the video filter to an FPGA card. The card being used is from Xilinx's Alveo series which are designed for accelerating data center applications. But in general this tutorial can be adapted to other FPGA cards with some simple changes.
 The steps to be carried out for this lab include:
- Setting up Vitis acceleration environment
- Running hardware optimized accelerator and comparing its performance with baseline

The lab simply demonstrates the huge performance gain that can be achieved as compared to CPU performance. Whereas next labs in this tutorial will guide through how such performance can be achieved using different optimizations and design techniques for 2D convolutional kernel and the host side application.
## Cloning Repo and Vitis setup
Clone the repository using following command:
```bash
git clone "------------------ADD REPO PATH FINAL LOCATION------------"
```
Setup application build and runtime environment using the following commands

```bash
source XILINX_INSTALL_PATH/settings64.sh
source /opt/xilinx/xrt/setup.sh
source XRT_INSTALL_PATH/setup.sh
```


## Baseline Application Performance
The software application is design to process HD video resolution images(1920x1080). It performs convolution on a set of images and prints the summary of performance results. It is used for measuring baseline software performance. Run the application to measure performance as follows:

```bash
./run.sh 
```
Results similar to the following will be printed out, note down the CPU throughput.

```
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
Run the following script to build host application. The binary file for FPGA(xclbin) takes considerable time for building so it is provided prebuilt.
    
    ```bash
    
    ```
### Launching the Host Application
Now launch the application to run using FPGA accelerated video convolution filter.

```bash
 ./run
```
 The result summary similar to the one given below will be printed.
 
## Results



 