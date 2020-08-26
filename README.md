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

# Accelerating Video Convolution Filtering Application
The tutorial introduces the users with a compute intensive application which is accelerated using Xilinx Alveo Data Center FPGA Acceleration Card. It goes through the design of specific kernel ( customized hardware implementation of application part chosen for acceleration) that runs on FPGA and briefly discusses host side application optimized for performance. The kernel is designed to maximize throughput and host application is optimized to transfer data in an effective manner that moves in-between host and FPGA card. The host application essentially uses a mechanism to hide the data movement latency by overlapping different data moves for multiple kernel calls. Another essential purpose of this tutorial is to show **_how one can easily estimate the performance of hardware kernels that can be built using Vitis HLS how accurate and close these estimates are to actual hardware performance_**

---------------------------------------

<p align="center"><b>
Next Lab Module: <a href="lab0_intrigue.md">Experience the Acceleration</a>
<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
</b></p>


