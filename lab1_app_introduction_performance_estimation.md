# Video Convolution Filter : Introduction and Performance Estimation
In this lab we will have a look at 2D video convolution filter measure its performance on host machine. These measurement will establish a performance baseline. Based on the required performance constraints and software performance we can calculate what acceleration a hardware implementation should provide. We will also estimate the performance of FPGA accelerator that will be implemented in latter labs. In nutshell during this lab you will:
- Learn about video convolution filter
- Measure the performance of host based software implementation
- Calculate required acceleration vs. software implementation
- Estimate the performance of hardware accelerator

## Video Filtering Applications and Filter Structure
Video applications use different type of filters extensively for multiple reasons such as to filter noise,  manipulate motion blurring, color and contrast enhancements, edge detection, creative effects etc. The essential purpose of video filter is to carry out some form of data average around a pixels that effects the amount and type of correlation any pixel has to its surrounding  area. Such filtering is carried out for all the pixels in video frames. Convolution operation is essentially a sum of products carried on a pixel set( a frame/image sub-matrix centered around a give pixel) and co-efficient matrix. 2D Convolution filters are actually defined by a matrix of co-efficients.
      ![](images/convolution.jpg)
Lab 1: ( Application intro , software performance and hardware performance estimation)

    Introduction to convolution video filter
    Video Specification
    measurements using timers.
    Setting throughput goals
    Estimating the accelerator performance