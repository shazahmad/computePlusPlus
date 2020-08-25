# Accelerating Video Convolution Filtering Application
The tutorial introduces the users with a compute intensive application. The applications is accelerated using Xilinx Alveo Data Center Acceleration Card. The tutorial will go through the design of specific kernel ( customized hardware implementation of application part chosen for acceleration) that runs on FPGA and briefly discusses host side application optimized for performance. The kernel is designed to maximize throughput and host application is optimized to transfer data in an effective manner that moves in-between host and FPGA card. The host application essentially uses a mechanism to hide the data movement latency by overlapping different data moves for multiple compute calls.
<p align="center"><b>
Proceed to Next Lab: <a href="lab0_intrigue.md">Experience the Acceleration</a>
<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
</b></p>  

