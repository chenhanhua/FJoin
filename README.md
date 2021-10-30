# FJoin: an FPGA-based parallel accelerator for stream join

FJoin is an FPGA-based parallel accelerator for stream which leverages a large number of basic join units connected in series to form a deep join pipeline to achieve large-scale parallelism.FJoin can do High-Parallel Flow Join, in which data of the join window can flow through once to complete all join calculations after loading multiple stream tuples. The host CPU and FPGA device coordinate control, divide the continuous stream join calculation into independent small-batch tasks and efficiently ensure completeness of parallel stream join.  FJoin is implemented on a platform equipped with an FPGA accelerator card.The test results based on large-scale real data sets show that FJoin can increase the join calculation speed by 16 times using a single FPGA accelerator card and reach 5 times system throughput compared with the current best stream join system deployed on a 40-node cluster, and latency meets the real-time stream processing requirements.

# Introduction
In the era of the Internet of Everything, real-time streaming data is in multiple sources and ubiquitous in everywhere. Since streams join between different sources can extract key information between multi-source streaming data, stream join has become an important operation in stream processing. However, among stream processing operators, the stream join operator is likely to become a system performance bottleneck due to high computational overhead. When the computing power of the stream processing system cannot meet the actual stream data rate, it will immediately cause serious congestion, resulting in a rapid increase in processing latency, and cannot achieve stream processing requires. Therefore, it is important to study high-performance stream join systems.  
  
Existing studies generally design multi-core parallel systems to obtain low-latency and high-throughput processing performance. The advantage of multi-core architecture is that it is easy to extend the parallelism to the number of CPU cores while maintaining the global order of stream tuples. Under the global order , it is easy to ensure that the join results are not repeated or omitted, that is, ensure the completeness. The parallel mode is also easy to collect the results generated by all the join cores, and sort them in the original order of the source tuples, that is, keep order-preserving output. However, the expansion of parallelism is limited by the resources of the machine and cannot cope with large-scale data in parallel mode.  
  
To further improve the parallelism of stream join processing, distributed stream join systems that support scale-out have received extensive attention in recent years. Distributed methods are generally built on distributed stream processing systems, such as the widely used Apache Storm system. This introduces the inherent overhead of the framework, and at the same time increases the communication overhead between nodes. Due to the loss of hardware performance, the distributed stream join system requires a big number of CPU cores, which makes deployment cost and maintenance cost high. In summary, the multi-core parallel stream join system is easy to scale efficiently, but the expansion of parallelism is limited, and the distribution method of scale-out can increase the expansion of parallelism further but causing a serious drop in hardware processing efficiency.
  
Because of the efficient and large-scale expansion problem of stream join systems, we focus on using FPGAs to accelerate stream join in parallel. On the one hand, for join predicates that are easy to implement in RTL, FPGAs have the advantage of realizing a large number of dedicated join core circuits. Comparing to the distributed system investing a big number of CPU cores to expansion, the cost-effective advantage of FPGA is obvious. On the other hand, a parallel stream join system has many advantages, but it is difficult to increase CPU and memory to achieve expansion. It is easy to add FPGA accelerator peripherals for nodes to scale up. Combining these advantages, FPGA is a very suitable accelerator platform for stream join computing. We propose and design an FPGA stream join parallel accelerator FJoin, which consists of a multi-core host and multiple FPGAs with independent memory access channels. In each FPGA, a large number of basic join units are connected in series to form a deep join pipeline, and the units include custom join predicates. The join pipeline loads multiple tuples of one stream and joins tuples in the window of another stream at a time. When the join window flows through the pipeline, all units are doing High-Parallel Flow Join. Considering that it is difficult to ensure the completeness of FPGA parallel join, a management thread is allocated to each pipeline in the host part of the system, so that the CPU and FPGA can work together. This parallel acceleration framework can not only break through the expansion limit of the multi-core parallel stream join system but also avoid the waste of hardware performance for distributed scale-out.  
  
  
# FJoin architecture
<img src="https://github.com/CGCL-codes/FJoin/blob/main/images/FJoin_img_arch.PNG"  width="600" height="400" alt="FJoin_img_arch"/><br/>
  
  
The overview of FJoin is shown in the picture. The system is divided into the upper part of the host-side software and the lower part of the device-side hardware. The hardware part includes the pipeline state machine(FSM) in each FPGA, and the memory access interface(Mem I/ F), and a deep join pipeline composed of a big number of basic join units in series. The software part includes a stream join task scheduler(Task Scheduler), a join completeness control module(Completeness Controller) in which generates join calculation batch tasks, and a result post-processing module(Post-Process Module), and management threads corresponding to each FPGA join pipeline.

# Basic join unit
<img src="https://github.com/CGCL-codes/FJoin/blob/main/images/FJoin_img_basic_join_unit.PNG"  width="600" height="300" alt="FJoin_img_basic_join_unit"/><br/>
  
The structure of the basic join unit is shown in the picture, which contains the join logic that can be customized using RTL, three streams of stream tuples, window tuples, and result tuples passed from the front to back, the control signals passed each stage clear and stall, and a join status register.

# How to use?

## Prerequisites
### Hardware
This project works on [Xilinx U280 Data Center Accelerator card](https://www.xilinx.com/products/boards-and-kits/alveo/u280.html).  

You can also use other FPGA acceleration devices, such as U250, just set it accordingly.

### Operation System
Ubuntu 18.04 LTS

### Software
[Vitis 2019.2](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vitis/2019-2.html)

[U280 Package File on Vitis 2019.2](https://www.xilinx.com/products/boards-and-kits/alveo/u280.html#gettingStarted)

## Environment
To compile and run the project, you need to configure the environment according to the following documents.
https://www.xilinx.com/html_docs/xilinx2020_1/vitis_doc/vhc1571429852245.html

## Build Project
After running Vitis, please set up a workspace, and then import the project from the zip file in the /proj directory. All source code and project configuration are included in it.

After that, select the "Hardware" target in the left down corner, and press the hammer button to build it. Please wait patiently for hours. The sample project is set to each join pipeline contains 16 basic join units, including a total of 32 units. Generally, the building time will not be too long.  

After building, you also need to prepare datasets and configuration files before running.
Dataset: Download the data files(gps_0102, gps_0304) from the /data, and put them into /src/data of the project.
Configuration file: Copy the run configuration file(run.cfg) in the /src to the Vitis running directory /Hardware/join_hw-Default

After running, the result will be written to the files in the /result directory, and the console will have corresponding output.

# Project Configuration

# Evaluation Result
<img src="https://github.com/CGCL-codes/FJoin/blob/main/images/FJoin_img_evaluation.PNG" alt="FJoin_img_evaluation"/><br/>
The direct manifestation of the processing capacity of the stream join system is the number of join calculations completed per unit of time. The picture on the left is a real-time comparison of the number of join calculations completed per second between FJoin and the distributed stream join system BiStream. We use the data set of Didi Chuxing and the corresponding join predicate, the sliding time window size is set to 180 seconds. The results show that FJoin with 1024 basic join units can complete more than 100 billion join predicate calculations per second. However, BiStream which runs in a 40-node cluster with 512 CPUs completes join predicate calculations about 6 billion times per second. From the perspective of connection predicate calculations, FJoin achieves a speedup of about 17.  

The picture on the right compares the real-time throughput between FJoin and BiStream. This test uses the data set of the network traffic trajectory and the corresponding join predicate, and the sliding time window size is set to 15 seconds. We also divided it into multiple tests to increase the input stream rate to test the system Throughput value. Compared with the BiStream system with 512 CPU cores, FJoin with 1024 basic join units can reach 5x real-time throughput. In addition, compared with the same 512 unit system scale, the throughput of FJoin is increased to about 4x.  

# Publication
If you want to know more detailed information, please refer to this paper:  
Lin L T, Chen H H, Hai J. FJoin: an FPGA-based parallel accelerator for stream join (in Chinese). Sci Sin Inform  

doi:10.1360/SSI-2021-0214(https://www.sciengine.com/publisher/scp/journal/SSI/doi/10.1360/SSI-2021-0214)

# Authors and Copyright
FJoin is developed in National Engineering Research Center for Big Data Technology and System, Cluster and Grid Computing Lab, Services Computing Technology and System Lab, School of Computer Science and Technology, Huazhong University of Science and Technology, Wuhan, China by Litao Lin (litaolin@hust.edu.cn), Hanhua Chen (chen@hust.edu.cn), Hai Jin (hjin@hust.edu.cn).

Copyright (C) 2021, [STCS & CGCL](http://grid.hust.edu.cn/) and [Huazhong University of Science and Technology](https://www.hust.edu.cn/).
